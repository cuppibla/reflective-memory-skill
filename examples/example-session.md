# Example session — one prompt → a reflective-memory agent

This is what the `reflective-memory-harness` skill produces once it’s loaded into an agentic IDE (Antigravity or Claude Code) and you give it a single prompt.

## The prompt

> Build me a reflective-memory support agent on Google Cloud (Python, ADK, model `gemini-2.5-flash`): it remembers users across sessions with Vertex AI Memory Bank, and learns from its own failed tickets with a Firestore “dream” that reflects offline. Add demos that prove both.

## The plan the skill steers it to

The skill’s guardrails shape the plan before any code is written:

1. Layout: `learn-the-user/` (Memory Bank) + `learn-the-job/` (harness + dream).
2. Sessions on Cloud SQL (`DatabaseSessionService`, async) — not InMemory.  *(guardrail 6)*
3. Memory Bank wiring + **synchronous** reflect (`wait_for_completion`).  *(guardrail 1)*
4. `trajectory_log` (+ outcome & root cause) → the **dream** (distill → chunk → embed → index) → `recall` (merge both stores).  *(guardrails 4, 5)*
5. Embeddings: `text-embedding-005` (768-dim).  *(guardrail 2)*
6. Dream as a Cloud Run Job + nightly Scheduler with `--oauth`.  *(guardrail 3)*

## What it scaffolds

```
learn-the-user/
  create_engine.py        provision the Memory Bank backend → AGENT_ENGINE_ID
  run_demo.py             session 1 states facts → reflect → session 2 recalls them
learn-the-job/
  harness/                memory tools · trajectory_log · the dream · config
  demo_level1.py          the controlled A/B (0/3 → 3/3)
  demo_level2.py          cross-agent inheritance
  dream_job/ + Dockerfile the dream, packaged as a Cloud Run Job
```

## What you see when it runs

Real output, reproduced from the [reference implementation](https://github.com/cuppibla/reflective-memory-demo):

```
SESSION 1:  "I'm vegetarian, planning a trip to Tokyo in April. Keep it budget-friendly."
SESSION 2 (new chat):  "Suggest a few places to eat for my trip."
            → "For budget-friendly vegetarian options in Tokyo: T's TanTan (vegan ramen), Ain Soph…"

A/B (same agent, same ticket, same customer facts):
  job memory OFF → "clear your cache / try another browser"            → ❌ failure (0/3)
  job memory ON  → "confirm your account's roles/permissions for billing" → ✅ success (3/3)

cross-agent:
  billing_agent (first contact, never failed) inherits the lesson      → ✅ success
```

## The traps it skipped

Because the skill named the fixes up front, none of these ever fired:

- **fire-and-forget amnesia** — used the synchronous write *(guardrail 1)*
- **the 3072-dim index wall** — used `text-embedding-005` *(guardrail 2)*
- **a scheduler that never triggers** — used `--oauth` *(guardrail 3)*
