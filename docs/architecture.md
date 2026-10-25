# zehn-analytics architecture

zehn-analytics is the BI module of the zehnmind suite. A business user asks a question in plain
language about their own database and gets back a chart, or, for harder questions, an analysis
(cleaning, statistics, a trained model) with insights, metrics and charts.

C4 model (levels 1–3 and 4 flows): IcePanel, landscape `zehnmind`.
Sequence diagrams: [build-graph-hil.md](build-graph-hil.md).

**Status legend:** **Live** = in code and running today. **Future** = agreed design, not built yet.
**Prototype** = code exists but is not wired into the product.

---

## 1. Scope

**Does**
- Turns one prompt into one result: a chart widget (`sql_chart`) or an analysis (`analysis`).
- Reads the tenant's own database (Postgres, MySQL, Mongo), **read-only**.
- Asks the user at most a few button questions when the prompt or the data is ambiguous.
- Runs analysis and ML code (pandas, scikit-learn) in an isolated sandbox.

**Does not**
- Write to the tenant's database.
- Hold a chat: one prompt → one build (follow-ups are new builds).
- Let the LLM write free code: sandbox code comes from reviewed templates.
- Send raw data rows to the LLM: only schema, profile and metrics.

---

## 2. Context (C4 level 1)

| Element | Kind | Role |
|---|---|---|
| Business User | person | asks questions, answers clarifying questions, reads results |
| zehn-analytics | our system | builds charts and analyses |
| Tenant Database | external | the customer's data source (Postgres / MySQL / Mongo) |
| LLM API | external | route, SQL, plan, fixes, summary (DeepSeek through an OpenAI-compatible client today) |
| zehn-auth | external (suite) | sign-in through the tenant's own IdP, scopes; users and projects synced from the orchestrator |

---

## 3. Containers (C4 level 2)

| Container | Technology | Responsibility | Status |
|---|---|---|---|
| Frontend | web app | prompt box, charts, question buttons, SSE client | Live |
| API | Python 3.12, FastAPI, SQLAlchemy, Alembic | REST, auth, starts builds, streams events (SSE), receives answers | Live |
| Worker | Celery, LangGraph | runs the build graph, the only part that calls the LLM for analysis | Live (legacy task) / Future (graph) |
| Redis | Redis | Celery result backend; later broker and event stream per build `build:{id}` | Live (result backend) / Future (broker, streams) |
| RabbitMQ | RabbitMQ | Celery broker today | Live, to be removed (D6) |
| Postgres | PostgreSQL | source of truth: projects, data sources, widgets, prompts, results; LangGraph checkpoints | Live / Future (checkpoints) |
| MinIO | S3-compatible storage | analysis charts (PNG), models (skops), notebooks, parquet extracts | Future (config exists, unused) |
| Sandbox runner | Python service, gVisor, k3s | creates and deletes sandbox containers, copies files in and out | Prototype (`sandbox/host`) |
| Sandbox container | `zehn-sandbox` image | runs one step of analysis code with no network | Prototype (`sandbox/image`) |
| NATS listener | `zehn-common` events | syncs users, projects, API keys from the orchestrator | Live |

Rules
- API and worker talk **only through Redis** (queue + events), never directly.
- Only the worker can reach the sandbox runner (token auth). The runner is not reachable from the internet.
- The sandbox container has **no network**. It only gets a parquet file, never DB credentials.

---

## 4. Build graph (worker, C4 level 3)

One LangGraph graph for both paths. Full order:

```
START → title ∥ introspect → route
route: reject → persist
       clarify → ask_user → apply_answer → route          (1-HIL, max 2 rounds)
→ generate_sql → validate (sqlglot / firewall)
→ sql_chart: execute → normalize → map_chart → suggest → persist
→ analysis:  profile → plan → spec_check
             ⇄ ask_user → apply_answer                     (2-HIL)
             → freeze → extract → open_session
             → [render_code → run_step → self_correct ≤3] per step
             → close_session (always) → summarize → verify_numbers → persist → END
```

| Node | What it does | LLM? | Status |
|---|---|---|---|
| title | short title for the build | yes | Live |
| introspect | reads table metadata | no | Live |
| route | intent: `sql_chart`, `analysis`, `clarify`, `reject` + safety | yes | Future |
| reject | explains why the prompt is refused | no | Future |
| ask_user | `interrupt()`: stops the graph and sends a question with options | no | Future |
| apply_answer | code writes the chosen option into the state | no | Future |
| generate_sql | SQL from prompt + metadata | yes | Live |
| validate | parse with sqlglot, read-only firewall, retries ≤3 | no | Live |
| execute | runs SQL on the tenant DB (read-only, timeouts) | no | Live |
| normalize, map_chart | result rows → chart spec | no | Live |
| suggest | follow-up suggestions | yes | Future |
| profile | column stats with SQL aggregates in the tenant DB | no | Future |
| plan | `AnalysisSpec`: list of template steps with params | yes | Future (prototype) |
| spec_check | code finds gaps (target, nulls, class balance) and builds questions | no | Future (prototype) |
| freeze | locks the spec: no questions after this point | no | Future |
| extract | query result → `in/dataset.parquet` (row cap) | no | Future |
| open_session / close_session | one sandbox container per build | no | Future (prototype) |
| precheck | code must come from `code_templates` and pass the import allow-list (AST) | no | Future (prototype) |
| render_code | copies the template, params go in `ctx.params` JSON | no | Future (prototype) |
| run_step | runs one step in the sandbox | no | Future (prototype) |
| self_correct | fixes only the failed step from its traceback, ≤3 tries | yes | Future (prototype) |
| summarize | insights from `metrics.json` | yes | Future (prototype) |
| verify_numbers | every number in the summary must exist in `metrics.json` | no | Future (prototype) |
| persist | saves the result, sets the end status | no | Live |

LLM calls per build: `sql_chart` 2–3, `analysis` typically 4 (route, generate_sql, plan, summarize),
6–7 with fixes, hard cap `MAX_LLM_CALLS = 8` → `failed: budget`.

End statuses: `done`, `partial` (a non-essential step failed), `failed`, `rejected`, `cancelled`.

---

## 5. Sandbox (C4 level 3)

**Why templates, not free code:** cheaper (4–6 LLM calls instead of 15–30 for an agent loop),
safer (template code is reviewed and tested), and numbers are easier to check.

**Templates** (`sandbox/code_templates/`, each defines `run(ctx)`):
`clean`, `describe`, `classify`, `regress`, `cluster`, `reduce`.
The returned dict goes to `metrics.json`, which is the only way numbers leave the sandbox.

**Workspace** per build:

```
ws/build_{id}/
  in/    dataset.parquet, spec.json        runner writes, container read-only
  code/  harness.py, step_01_clean.py, step_02_train.v2.py ...   read-only
  work/  intermediate parquet, model.skops  container writes
  out/   metrics.json, charts/*.png, steps/step_k.json, logs/, manifest.json
  meta.json  status, session_id, last_activity   host only, not mounted
```

**Isolation**

| Control | Value |
|---|---|
| Runtime | gVisor (`runsc`) via RuntimeClass on k3s; Docker + runsc in dev |
| Network | none |
| File system | read-only root, `in/` and `code/` read-only, small tmpfs `/tmp` |
| User | non-root (uid 10001), all capabilities dropped, `no-new-privileges` |
| Limits per step | analysis 60 s / 2 GB, train 10 min / 4 GB (starting values) |
| Packages | pinned in the image, pip removed: numpy, pandas, pyarrow, scipy, scikit-learn, matplotlib, skops |
| Models | saved as skops (not joblib/pickle), opened only inside the sandbox |
| Cleanup | reaper deletes the workspace by build status, not by a timer |

---

## 6. Data: where what lives

| Store | Holds | Notes |
|---|---|---|
| Tenant DB | the customer's data | read-only connection, query firewall, timeouts |
| Postgres (ours) | projects, data sources (encrypted credentials), widgets, prompts, results, checkpoints | source of truth |
| Redis | Celery queue, event stream per build | broker in its own DB, AOF on, `noeviction` |
| MinIO | charts, models, notebooks, extracts | Future |
| Sandbox workspace | one build's files | deleted after persist |

Graph state is small and JSON only: paths, not DataFrames. Credentials (`uri`) are never in the state.

---

## 7. Communication

| From → To | How |
|---|---|
| Frontend → API | HTTPS (REST), SSE for build events |
| API → Worker | Celery task through Redis |
| Worker → API | events in a Redis stream, API forwards them over SSE |
| Worker / API → Tenant DB | DB driver, read-only |
| Worker → LLM API | HTTPS, OpenAI-compatible client |
| Worker → Sandbox runner | HTTP with token (prototype: `docker run -i` with JSON lines) |
| Runner ↔ Sandbox container | files in the workspace + commands to `harness.py` |
| Orchestrator → API | NATS events (users, projects, API keys) |

---

## 8. Security

- Query firewall: SQL is parsed with sqlglot, only reads allowed, permission enforcer and throttling per source.
- Data from the tenant is treated as untrusted text in prompts.
- The LLM sees schema, profile and metrics, never raw rows.
- Sandbox code comes only from templates; the LLM edits code only in `self_correct`, and the result passes `precheck` again.
- Numbers in the summary are checked against `metrics.json` (`verify_numbers`).
- Sandbox: gVisor, no network, no secrets, read-only inputs, limits (section 5).

---

## 9. Limits

| Name | Value |
|---|---|
| `MAX_SQL_ATTEMPTS` | 3 |
| `MAX_CLARIFY_ROUNDS` | 2 |
| `MAX_FIX_ATTEMPTS` | 3 per step |
| `MAX_LLM_CALLS` | 8 per build |
| Answer wait | 24 h, then `cancelled` |
| SQL driver / LLM timeouts | 30 s / 45 s |

---

## 10. Decisions

| # | Decision | Why |
|---|---|---|
| D1 | One graph for `sql_chart` and `analysis` | shared start (title, introspect, route, SQL), one place for events and statuses |
| D2 | Sandbox steps come from templates; the LLM only plans, fixes and summarizes | cost, safety, checkable numbers |
| D3 | Questions only before the sandbox, as buttons | the sandbox run is never paused; answers are easy to validate |
| D4 | HIL = `interrupt()` + Postgres checkpointer; the Celery task ends while waiting, the answer starts a resume task | no worker sits idle for up to 24 h |
| D5 | Graph runs in the Celery worker, events go through a Redis stream | the build survives a closed browser and can be resumed |
| D6 | Celery broker moves from RabbitMQ to Redis | the rest of the suite already uses Redis |
| D7 | gVisor by default, Kata only if prod has KVM and needs stronger isolation | works without KVM, near-native speed for CPU sklearn |
| D8 | Models as skops, not joblib | loading joblib means running code |
| D9 | Template code is plain `.py` with `run(ctx)`, not notebooks; a notebook is only an export | no hidden cell state, easy to review and test |

---

## 11. Today vs target

| Area | Today | Target |
|---|---|---|
| Graph location | inside the API request (`app/build_graph/runner.py`), cancelled when the client disconnects | Celery worker (D5) |
| Graph nodes | `sql_chart` nodes only | full graph (section 4) |
| Broker | RabbitMQ (`CELERY_BROKER=amqp://...`), Redis only as result backend | Redis (D6) |
| Checkpoints | graph compiled without a checkpointer | Postgres checkpointer |
| Sandbox | prototype in `sandbox/`, runs with Docker CLI | runner service on k3s with gVisor |
| MinIO | config only | charts, models, notebooks |

---

## 12. Open questions

1. **Timeout after 24 h:** cancel the build, or continue with the recommended option?
   An earlier draft used an `expire_question` task with the recommended option; the sequence diagrams use cancel.
2. **2-HIL rounds:** is there a limit for `spec_check ⇄ ask_user`, like the 2 rounds of 1-HIL?
3. **`sql_chart` location:** move it to the worker together with `analysis` (simpler, one event path),
   or keep it in the API for speed? 1-HIL happens before the path is known, so a split is awkward.
4. **Production sandbox host:** dedicated tainted node for sandbox pods? Does the host have `/dev/kvm`?
5. **Data lake size:** small per-tenant parquet in MinIO read with DuckDB, or a full lake with nightly ETL/CDC?
