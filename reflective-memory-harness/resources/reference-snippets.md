# Reference snippets (verified API calls)

These mirror the runnable reference implementation at **github.com/cuppibla/reflective-memory-demo** — copy exact imports/signatures from there, since they can shift across ADK versions. The *calls* below are the load-bearing, verified ones.

## 1. Sessions + Memory Bank, one Runner (learn-the-user wiring)
```python
session_service = DatabaseSessionService(
    db_url="postgresql+asyncpg://USER:PASS@127.0.0.1:5432/agentdb")   # async driver; durable sessions, never InMemory in prod
memory_service  = VertexAiMemoryBankService(
    project=PROJECT, location="us-central1", agent_engine_id=AGENT_ENGINE_ID)

# Import PreloadMemoryTool from the SUBMODULE — `from google.adk.tools import
# PreloadMemoryTool` raises ImportError on ADK 2.x (lazy __init__). Verified fix:
#   from google.adk.tools.preload_memory_tool import PreloadMemoryTool
assistant = Agent(
    model="gemini-2.5-flash", name="assistant",
    instruction="Use what you remember; never re-ask what you already know.",
    tools=[PreloadMemoryTool(), recall])
runner = Runner(agent=assistant, app_name=APP,
                session_service=session_service, memory_service=memory_service)
```

## 2. Reflect the finished conversation into Memory Bank — SYNCHRONOUSLY (Guardrail 1)
```python
await memory_service.add_events_to_memory(
    app_name=APP, user_id=user_id, events=session.events,
    custom_metadata={"wait_for_completion": True})   # blocks until extract + consolidate finish
# NEVER add_session_to_memory() in a short-lived path — fire-and-forget → amnesia
```

## 3. Log a finished task as a trajectory (learn-the-job; zero added latency)
```python
log_trajectory(
    agent_id=AGENT_ID, user_id=user_id,
    task=task_description,                # whatever the agent attempted
    events=[...], outcome="failure",
    failure_reason="<the REAL root cause — not just 'the output looked wrong'>")  # Guardrail 4
# → one doc in `trajectories`, processed=False (the dream's work queue)
```

## 4. The dream — reflect FAILED trajectories into lessons (runs as a Cloud Run Job)
```python
for traj in trajectories.where(filter=FieldFilter("processed", "==", False)).stream():
    t = traj.to_dict()
    if t["outcome"] == "failure":                      # reflect on failures
        d    = distill(t)                              # Gemini → {"lesson": ..., "procedure": [ ...steps... ]}
        text = chunk(d)                                # one compact "lesson + numbered steps"
        vec  = embed(text)                             # text-embedding-005, 768-dim — Guardrail 2
        memory_chunks.add({
            "text": text, "embedding": Vector(vec),
            "scope": "harness_shared", "source_trajectory": traj.id})   # Guardrail 5
    traj.reference.update({"processed": True})         # idempotent — never re-processed
```

## 5. recall — read BOTH stores, merge into one payload (the agent's tool)
```python
async def recall(query: str, tool_context) -> dict:
    facts   = await tool_context.search_memory(query)              # Memory Bank — durable facts about the user
    lessons = search_job_memory(query, agent_id=AGENT_ID, k=3)     # Firestore KNN — lessons from past tasks
    return {"user_facts": facts, "job_lessons": lessons}          # the merge (rename keys to your domain)
# search_job_memory does KNN — vector_field is REQUIRED and you must iterate .stream():
#   col.find_nearest("embedding", query_vector=Vector([...]), limit=k*4,
#                    distance_measure=DistanceMeasure.DOT_PRODUCT).stream()
# Scope is NOT a single server-side filter (it's an OR across scope AND agent_id):
# over-fetch (k*N) then post-filter in Python — harness_shared for anyone;
# private/agent_shared only for the owning agent_id.
```

## 6. Ship the dream — Cloud Run Job + nightly Scheduler (Guardrail 3)
```bash
cd learn-the-job
gcloud run jobs deploy dream-job --source . --region us-central1 \
  --service-account dream-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com

gcloud scheduler jobs create http dream-nightly --location us-central1 --schedule "0 3 * * *" \
  --uri "https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/$GOOGLE_CLOUD_PROJECT/jobs/dream-job:run" \
  --http-method POST \
  --oauth-service-account-email dream-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
