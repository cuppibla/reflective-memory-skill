# Gotchas — the traps that cost real debugging sessions

Each of these was hit on live GCP while building the reference implementation. They are the main reason this skill exists: a builder agent that knows them up front skips an afternoon of confusion.

## 1. Fire-and-forget amnesia (the #1 trap)
- **How it shows:** the agent stores nothing; in a short-lived script you may see `Background ingest_events task failed: Cannot send a request, as the client has been closed.`
- **Why:** the obvious `add_session_to_memory()` returns *before* the write lands. The client closes first.
- **Fix:** `await memory_service.add_events_to_memory(app_name=APP, user_id=uid, events=session.events, custom_metadata={"wait_for_completion": True})` — blocks until extract + consolidate finish.

## 2. The 3072-dim vector-index wall
- **How it shows:** the Firestore composite vector index fails to build / similarity search errors out.
- **Why:** `gemini-embedding-001` returns **3072** dims; Firestore's vector index caps at **2048**.
- **Fix:** embed with **`text-embedding-005` (768-dim)**. (And the index is *mandatory* — `find_nearest` returns nothing until the composite vector index on `memory_chunks.embedding` exists.)

## 3. `--oauth`, not `--oidc`, for the scheduled dream
- **How it shows:** the Cloud Scheduler job exists but the dream never runs; no error that points at auth.
- **Why:** the scheduler's target is a Google API (the Cloud Run Admin API), which expects an OAuth token. `--oidc-*` is for your own services.
- **Fix:** `--oauth-service-account-email <sa>` on the scheduler job. The SA needs `roles/run.invoker` (which includes `run.jobs.run`), plus `aiplatform.user` + `datastore.user` to do the work.

## 4. The outcome-signal trap (decides whether learning works at all)
- **How it shows:** the dream distills only vague meta-advice — *"characterize the problem, then apply a structured diagnostic flow."*
- **Why:** the failed trajectory recorded only *"the reply looked generic"*, with no real root cause.
- **Fix:** log the **resolution / root cause** on every finished task. A reflective system is only as good as the outcome signal in its trajectories. Real systems record an outcome/resolution on close — that resolution *is* the learning signal.

## 5. `InMemorySessionService` in production
- **How it shows:** sessions vanish on restart; "it worked in the demo."
- **Fix:** `DatabaseSessionService` on Cloud SQL, with the **async** driver (`postgresql+asyncpg://`). Reserve `InMemorySessionService` for throwaway tests.

## 6. Scope leakage and poisoned lessons (governance)
- **How it shows:** one agent (or tenant) reads another's private memories; or a wrong lesson keeps resurfacing with no way to remove it.
- **Fix:** every chunk carries a `scope` (`private` | `agent_shared` | `harness_shared`) enforced **at retrieval**, and a `source_trajectory` so a poisoned/wrong lesson can be traced and revoked. A reflective system preserves a bad conclusion as faithfully as a good one — make it revocable.
