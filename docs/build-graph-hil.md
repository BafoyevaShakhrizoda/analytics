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
| Graph | The single LangGraph build graph |
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

## 2. Clarify before SQL (route asks)

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
        G->>G: ask_user (question + options)
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

## 3. Analysis: confirm, cancel, timeout

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

    G->>G: profile (SQL aggregates, no LLM)
    G->>LLM: plan
    LLM-->>G: AnalysisSpec
    G->>G: spec_check → SpecGaps found
    G->>G: ask_user (questions from templates)
    G->>CP: save checkpoint
    G-->>API: event: question
    API-->>U: SSE: question with buttons
    Note over G,CP: graph is paused (interrupt)

    alt confirm
        U->>API: answer (chosen option)
        API->>G: resume (thread_id, answer)
        CP-->>G: load state
        G->>G: apply_answer (code writes choice into spec)
        G->>G: spec_check again
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
        API->>G: resume (thread_id, timeout)
        G->>CP: persist (status = cancelled)
    end
```

---

## Rules to keep in mind

- On resume only the interrupted node runs again, so its code before the interrupt must be safe to repeat (idempotent).
- Re-plan only if the answer changes the goal. Re-profile only if the dataset changes.
- No questions inside the sandbox: all questions happen before `freeze`.
- `close_session` always runs, even after an error.
