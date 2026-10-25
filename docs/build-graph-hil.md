# Build graph: reject, ask_user, resume

Sequence diagrams for the zehn-analytics build graph (LangGraph).
They show what happens when a prompt is rejected, when the graph stops to ask the user,
and how it continues after the answer, a cancel or a timeout.

This is the **target** design. Today only the `sql_chart` nodes exist in code
(`title`, `introspect`, `generate_sql`, `validate`, `execute`, `normalize`, `map_chart`, `persist`).
Everything else is planned (status *Future* in IcePanel).

## Participants

| Name | What it is |
|---|---|
| User | Business user in the Frontend |
| API | FastAPI backend: starts the graph, streams events over SSE, receives answers |
| Graph | The single LangGraph build graph (runs in the Celery worker, see section 4) |
| Redis | Celery broker + event stream per build (`build:{prompt_id}:events`) |
| Checkpointer | Postgres, keeps graph state between pause and resume (`thread_id = build:{prompt_id}`) |
| LLM | LLM API (route, plan, summarize) |
| Sandbox | Sandbox runner + container (gVisor) |

## Limits

| Constant | Value |
|---|---|
| `MAX_CLARIFY_ROUNDS` | 2 |
| `MAX_FIX_ATTEMPTS` | 3 |
| Answer timeout | 24 h, then status `cancelled` |

End statuses: `done`, `partial`, `failed`, `rejected`, `cancelled`.

## Two places where the graph asks the user

| | Where | Who asks | Example |
|---|---|---|---|
| **1-HIL** | before SQL | `route` (what to do?) | `gross_revenue` or `net_revenue`? |
| **2-HIL** | after SQL, before the sandbox | `spec_check` (how to do it?) | impute 40% nulls or drop rows? |

Both use the same `ask_user` node. `ask_origin` in the state says where to go back after the answer:
`route` for 1-HIL, `spec_check` for 2-HIL. There are no questions inside the sandbox.

---

## 0. Quick chart (no questions)

The most common case: a clear question that needs one SQL query and a chart.
Today this runs inside the API request (`app/build_graph/runner.py`); the target moves it to the worker (section 4).

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant G as Graph
    participant LLM
    participant DB as Tenant DB
    participant CP as Checkpointer

    U->>API: send prompt
    API->>G: start (thread_id = build:{prompt_id})
    API-->>U: SSE connected
    par
        G->>LLM: title
        G-->>U: SSE: title
    and
        G->>G: introspect (DB metadata)
    end
    G->>LLM: route
    LLM-->>G: intent = sql_chart

    loop at most MAX_SQL_ATTEMPTS (3)
        G->>LLM: generate_sql
        G->>G: validate (sqlglot / firewall)
        G->>DB: execute (read-only, timeout)
        DB-->>G: rows or error
    end
    G-->>U: SSE: sql

    G->>G: normalize → map_chart
    G->>LLM: suggest (follow-up ideas)
    G->>CP: persist (status = done)
    G-->>U: SSE: chart + suggestions, done
```

---

## 1. Rejected prompt

`route` decides the prompt is unsafe or out of scope. No SQL is generated.

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant G as Graph
    participant LLM
    participant CP as Checkpointer

    U->>API: send prompt
    API->>G: start (thread_id = build:{prompt_id})
    par
        G->>LLM: title
    and
        G->>G: introspect (DB metadata)
    end
    G->>LLM: route (intent + safety)
    LLM-->>G: intent = reject, reason
    G->>G: reject (explain reason)
    G->>CP: persist (status = rejected)
    G-->>API: event: rejected + reason
    API-->>U: SSE: "can't do this, because ..."
```

---

## 2. 1-HIL: clarify before SQL (route asks)

`route` finds the prompt unclear (e.g. `gross_revenue` or `net_revenue`?).
The graph stops, the user answers, `route` runs again. At most 2 rounds.

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant G as Graph
    participant LLM
    participant CP as Checkpointer

    U->>API: send prompt
    API->>G: start
    G->>LLM: route
    LLM-->>G: intent = clarify

    loop at most MAX_CLARIFY_ROUNDS (2)
        G->>G: ask_user (ask_origin = route)
        G->>CP: save checkpoint
        G-->>API: event: question
        API-->>U: SSE: question with buttons
        Note over G,CP: graph is paused (interrupt)

        U->>API: answer
        API->>G: resume (thread_id, answer)
        CP-->>G: load state
        G->>G: apply_answer
        G->>LLM: route again
    end

    LLM-->>G: intent = sql_chart / analysis
    G->>G: generate_sql → validate → ...
```

---

## 3. 2-HIL: after SQL, before the sandbox (confirm, cancel, timeout)

After `plan`, code in `spec_check` finds gaps (target column, 40% nulls, ...).
The user confirms an option, cancels, or does not answer in 24 h.

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant G as Graph
    participant LLM
    participant CP as Checkpointer
    participant SB as Sandbox

    Note over G: route said analysis
    G->>LLM: generate_sql
    G->>G: validate (sqlglot / firewall)
    G->>G: profile (SQL aggregates, no LLM)
    G->>LLM: plan
    LLM-->>G: AnalysisSpec
    G->>G: spec_check → SpecGaps found
    G->>G: ask_user (ask_origin = spec_check, questions from templates)
    G->>CP: save checkpoint
    G-->>API: event: question
    API-->>U: SSE: question with buttons
    Note over G,CP: graph is paused (interrupt)

    alt confirm
        U->>API: answer (chosen option)
        API->>G: resume (thread_id, answer)
        CP-->>G: load state
        G->>G: apply_answer (code writes choice into spec)
        G->>G: back to spec_check (ask_origin)
        G->>G: freeze (spec_frozen)
        G->>G: extract (parquet, row cap)
        G->>SB: open_session
        loop each step
            G->>G: render_code (template, no LLM)
            G->>SB: run_step
            alt error
                G->>LLM: self_correct (at most 3)
            end
        end
        G->>SB: close_session (always)
        G->>LLM: summarize (reads metrics.json)
        G->>G: verify_numbers
        G->>CP: persist (status = done / partial / failed)
        G-->>API: event: done
        API-->>U: SSE: insights, metrics, charts
    else cancel
        U->>API: cancel
        API->>G: resume (thread_id, cancel)
        G->>CP: persist (status = cancelled)
        API-->>U: SSE: cancelled
    else no answer in 24 h
        Note over G: Celery beat finds builds paused > 24 h
        G->>G: resume (thread_id, timeout)
        G->>CP: persist (status = cancelled)
    end
```

---

## 4. User leaves and comes back (proposed)

A question can wait up to 24 h, so the graph must not depend on the user's browser.
Proposal: the graph runs in the **Celery worker**, not inside the API request.
The worker writes events to a Redis stream; the API only reads that stream for SSE.
Closing the page closes the SSE connection, but the build keeps its state.

```mermaid
sequenceDiagram
    actor U as User
    participant API
    participant R as Redis
    participant W as Worker (Graph)
    participant CP as Checkpointer

    U->>API: send prompt
    API->>R: enqueue run_build(prompt_id)
    API-->>U: SSE connected
    R->>W: run_build
    W->>W: ... route / spec_check → ask_user
    W->>CP: save checkpoint (status = waiting_answer)
    W->>R: event: question
    R-->>API: event: question
    API-->>U: SSE: question
    Note over W: worker is free, graph is paused

    U--xAPI: closes the page (SSE closed)
    Note over U,CP: nothing is cancelled, build waits up to 24 h

    U->>API: opens the page again
    API->>CP: load build state
    CP-->>API: status = waiting_answer, question
    API-->>U: shows the same question
    U->>API: SSE reconnect (Last-Event-ID)
    R-->>API: events after Last-Event-ID
    API-->>U: missed events (if any)

    U->>API: answer
    API->>R: enqueue resume_build(prompt_id, answer)
    R->>W: resume_build
    CP-->>W: load state
    W->>W: apply_answer → back to ask_origin → ...
    W->>R: events: stages ..., done
    R-->>API: events
    API-->>U: SSE: result
```

What changes against today's code: `app/build_graph/runner.py` runs the graph inside FastAPI
(`asyncio.create_task`) and cancels it when the client disconnects. With this proposal the
disconnect only closes the stream; the build itself is stopped only by cancel, timeout or the end.

---

## Rules to keep in mind

- On resume only the interrupted node runs again, so its code before the interrupt must be safe to repeat (idempotent).
- Re-plan only if the answer changes the goal. Re-profile only if the dataset changes.
- No questions inside the sandbox: all questions happen before `freeze`.
- `close_session` always runs, even after an error.
