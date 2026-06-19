# Project layout

Memory machinery lives in a `harness/`; agents are thin config. That boundary is what lets a second agent inherit memory for free (Proof 3). Adapt the names to your domain — this is the shape, not a mandate.

```
learn-the-user/                 # Memory Bank (managed) — durable per-user facts
  create_engine.py                provision the Memory Bank backend → AGENT_ENGINE_ID
  run_demo.py                     session 1 states facts → reflect → session 2 recalls (Proof 1)

learn-the-job/                  # Firestore + the dream — lessons from the agent's own work
  harness/
    memory/    trajectory_log · tools (recall / search_job_memory) · schema
    dream/     the offline reflection (distill → chunk → embed → index)
    config
  agents/                         # each agent = thin config (instructions, tools, scopes); NO memory code
  dream_job/  + Dockerfile        the dream packaged as a Cloud Run Job
  demo_level1.py                  the honest A/B (Proof 2)
  demo_level2.py                  cross-agent inheritance (Proof 3)
```

Two tests of a correct boundary:

- **Addition test** — add a new agent without touching `harness/`.
- **Inheritance test** — that new agent gets logging, `recall`, and dream lessons with *zero* memory code of its own.

If you can't add an agent without editing `harness/`, the boundary leaked and memory is still agent code wearing a nicer name.
