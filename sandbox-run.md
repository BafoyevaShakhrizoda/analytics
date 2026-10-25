# Sandbox: running one analysis

Sequence diagrams for the sandbox part of the `analysis` path: from `extract` to `persist`.
Questions to the user are already over at this point (`freeze`), see [build-graph-hil.md](build-graph-hil.md).
Architecture and limits: [architecture.md](architecture.md), sections 5 and 9.

**Status:** prototype in `sandbox/` (zehn-analytics_backend). The prototype talks to the container with
`docker run -i` and JSON lines over stdin/stdout. The target is a runner service over HTTP on k3s with gVisor.
The diagrams show the target; the harness commands (`load`, `run`, `close`) are the same in both.

## Participants

| Name | What it is |
|---|---|
| Worker | Celery worker running the build graph |
| Runner | sandbox runner service: creates containers, moves files, sends commands; only the worker can call it (token) |
| WS | workspace on the host: `ws/build_{id}/` with `in/`, `code/`, `work/`, `out/`, `meta.json` |
| Harness | `harness.py` inside the container: loads data once, runs steps, writes results |
| LLM | LLM API (only `self_correct` and `summarize` here) |
| MinIO | storage for charts, models, notebooks |
| Postgres | build status and results |

## Harness commands

| Command | What it does |
|---|---|
| `{"op": "load"}` | reads `in/dataset.parquet` into memory, once per session |
| `{"op": "run", "step", "file", "params", "timeout"}` | runs `code/<file>`, calls `run(ctx)`, writes `out/steps/<step>.json` |
| `{"op": "close"}` | stops the harness |

---

## 1. Normal run

```mermaid
sequenceDiagram
    participant W as Worker
    participant R as Runner
    participant WS as Workspace
    participant H as Harness (gVisor)
    participant M as MinIO
    participant P as Postgres

    Note over W: spec is frozen, no more questions
    W->>W: extract (SQL result → parquet, row cap)
    W->>R: create workspace (build_id)
    R->>WS: mkdir in/ code/ work/ out/, meta.json
    W->>R: PUT in/dataset.parquet, in/spec.json
    R->>WS: write files

    W->>R: open_session
    R->>H: start container (no network, read-only, non-root, limits)
    R->>H: load
    H->>WS: read in/dataset.parquet
    H-->>R: ok (rows, columns)
    R-->>W: session_id

    loop each step in spec (clean → describe → classify ...)
        W->>W: render_code (copy template, no LLM)
        W->>W: precheck (template only, import allow-list)
        W->>R: PUT code/step_k.py
        R->>WS: write code (read-only for the container)
        W->>R: POST run step_k (params, timeout by kind)
        R->>H: run
        H->>H: run(ctx) with ctx.df, ctx.params
        H->>WS: write work/*, out/charts/*, out/steps/step_k.json, merge into metrics.json
        H-->>R: ok, duration, outputs
        R-->>W: step result
        W->>P: event: step k done
    end

    W->>R: close_session (always, also after errors)
    R->>H: close, remove container
    W->>R: GET out/ (manifest.json, metrics.json, charts, model)
    R-->>W: files
    W->>W: check sizes and manifest
    W->>M: upload charts, model.skops, analysis.ipynb
    W->>W: summarize (LLM reads metrics.json only)
    W->>W: verify_numbers
    W->>P: persist (status = done)
    W->>R: delete workspace
    R->>WS: rm -r ws/build_{id}
```

---

## 2. A step fails: self_correct

The LLM sees only this step's code and the trimmed traceback. The fixed code is saved as a new
version (`step_k.v2.py`) and must pass `precheck` again.

```mermaid
sequenceDiagram
    participant W as Worker
    participant R as Runner
    participant H as Harness (gVisor)
    participant LLM
    participant P as Postgres

    W->>R: POST run step_k
    R->>H: run
    H-->>R: error + traceback
    R-->>W: step failed

    loop at most MAX_FIX_ATTEMPTS (3)
        W->>LLM: self_correct (step code + traceback)
        LLM-->>W: fixed code
        W->>W: precheck again
        alt precheck fails
            W->>W: count as a failed attempt
        else precheck ok
            W->>R: PUT code/step_k.v2.py
            W->>R: POST run step_k (v2)
            R->>H: run
            H-->>R: ok or error
        end
    end

    alt step fixed
        W->>W: continue with the next step
    else still failing and step is essential
        W->>R: close_session
        W->>P: persist (status = failed)
    else still failing and step is not essential
        W->>W: skip, continue with the next step
        Note over W,P: build ends with status = partial
    end
```

`MAX_LLM_CALLS = 8` covers the whole build: when the budget runs out, the build ends with `failed: budget`.

---

## 3. Timeout and out of memory

```mermaid
sequenceDiagram
    participant W as Worker
    participant R as Runner
    participant H as Harness (gVisor)
    participant P as Postgres

    W->>R: POST run step_k (timeout 60 s or 10 min)
    R->>H: run

    alt step takes too long
        H->>H: SIGALRM stops the step
        H-->>R: error = timeout
        R-->>W: step failed (timeout)
        Note over W: treated like a failed step (section 2)
    else container runs out of memory
        H--xR: container killed (OOM)
        R-->>W: session lost
        W->>P: persist (status = failed, reason = memory)
    end

    W->>R: close_session (always)
```

---

## 4. Cleanup

The reaper deletes workspaces by build status, not by a timer, so a build that is still running is never deleted.

```mermaid
sequenceDiagram
    participant RP as Reaper (in Runner)
    participant WS as Workspace
    participant P as Postgres

    loop every few minutes
        RP->>WS: list ws/build_*/meta.json
        RP->>P: status of these builds
        P-->>RP: statuses
        alt done, partial, failed, rejected or cancelled
            RP->>WS: remove container if still there, rm -r ws/build_{id}
        else running and last_activity is fresh
            RP->>RP: keep
        else running but no activity (worker died)
            RP->>WS: remove container and workspace
            RP->>P: status = failed (reason = lost)
        end
    end
```

---

## Rules

- No questions inside the sandbox. Everything the user decides is in `spec.json` before `open_session`.
- One container per build. The data is loaded once; each step runs on the in-memory data.
- The container gets only parquet: no DB credentials, no network, no secrets.
- Only `metrics.json` carries numbers out. `summarize` reads it and `verify_numbers` checks the summary against it.
- The container writes `out/`, so the worker does not trust it blindly: it checks the manifest and file sizes before upload.
- `meta.json` is outside the container's reach, so code in the sandbox cannot stop its own cleanup.

## Open questions

1. Timeout or out of memory: fail the step, or retry once on a smaller sample of the data?
2. How often the reaper runs, and after how long without activity a running build counts as lost.
