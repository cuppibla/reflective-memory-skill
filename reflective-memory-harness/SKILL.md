---
name: reflective-memory-harness
description: >
  Use when building or scaffolding an AI agent that should LEARN, not just persist —
  i.e. it needs reflective memory on Google Cloud: durable per-user facts (Vertex AI
  Memory Bank) AND lessons distilled from its own past tasks (Firestore + a scheduled
  offline "dream"), wired with Google ADK. Encodes the two-store architecture, the
  non-negotiable guardrails, and the verified gotchas (fire-and-forget amnesia, the
  3072-dim vector-index wall, --oauth vs --oidc). Triggers on: "reflective memory",
  "agent that learns from its mistakes", "Memory Bank + Firestore", "build my own
  Dreams", "self-improving support agent".
---

# Reflective-Memory Harness (Google Cloud + ADK)

This skill builds an ADK agent with **two kinds of reflective memory**, and keeps the build on the rails that took real GCP runs to find. Use it whenever the user wants an agent that **improves from experience** rather than one that merely stores transcripts.

Persistence is not memory. Storing a transcript is easy; the value is the loop — *extract → consolidate → retrieve* — running online for facts and offline for lessons.

## What you're building

Two kinds of memory, two stores, merged at read time:

1. **Learn-the-user (semantic facts).** At the end of each conversation, reflect it into **Vertex AI Memory Bank**, which extracts + consolidates durable per-user facts (*e.g. "prefers vegetarian food," "on the Enterprise plan"*). Retrieve with `PreloadMemoryTool` / `search_memory`.
2. **Learn-the-job (episodic lessons).** Log each finished task as a **trajectory** in **Firestore**. An offline **"dream"** — a Cloud Run Job on a schedule — reflects over the *failed* (or inefficient) trajectories, distills each into a `{lesson, procedure}`, embeds it, and indexes it in a `memory_chunks` vector collection. Retrieve with `find_nearest`. For a **multi-tool / multi-step** agent the `procedure` can encode a *better path* (which tool to call first) — so with a capable base model the win is often a **shorter trajectory to the same answer**, not a wrong→right fix.

A single **`recall()`** tool fans out to both stores and merges them into one context the agent acts on.

> This is "build your own **Dreams**" — the open, Google-Cloud equivalent of Anthropic's managed Dreams feature: Memory Bank does the user-memory consolidation, and the offline dream surfaces the job lessons.

## Does this fit your agent?

This skill is domain-neutral. The reference repo *happens* to build a customer-support agent, but the pattern fits **any agent that talks to users and/or completes tasks** — keep your prose generic and rename `recall`'s return keys and the trajectory fields to your domain. Decide which half you need:

- Pure chatbot / Q&A assistant → **learn-the-user only** (Memory Bank). No dream.
- Agent that completes *tasks* (tickets, workflows, multi-step tool use) → **both halves** — its tasks become trajectories the dream learns from.
- Stateless one-shot tool ("summarize this PDF") → neither; skip the harness entirely.

## The build plan

Work in this order — **plan → user review → implement module by module → verify.** Do not declare the build done until the three proofs in `resources/evaluation.md` pass. Steps:

1. **Layout** — `learn-the-user/` (Memory Bank) and `learn-the-job/` (harness + dream); keep memory machinery inside a `harness/`, agents as thin config, so a second agent inherits memory for free. Full tree in `resources/project-layout.md`.
2. **Sessions** — `DatabaseSessionService` on Cloud SQL (async `asyncpg`). Never `InMemorySessionService` outside a throwaway test.
3. **Learn-the-user** — wire `VertexAiMemoryBankService`; reflect at session end **synchronously** (Guardrail 1).
4. **Learn-the-job** — a `trajectory_log` (records `agent_id`, `user_id`, events, an `outcome`, and the **resolution / root cause**); the `dream` (distill → chunk → embed → index); the `recall` tool (merge both stores).
5. **Ship the dream** — package it as a Cloud Run Job with a Dockerfile, fired nightly by Cloud Scheduler (Guardrail 3).
6. **Verify before declaring done** — run the three proofs in `resources/evaluation.md`: cross-session user memory, the **honest A/B** (the lesson is the *only* variable, scored by an LLM judge), and cross-agent inheritance. A stored lesson proves nothing until it changes the next decision.

Resources: **`reference-snippets.md`** (verified code for each step), **`gotchas.md`** (hard constraints), **`evaluation.md`** (the three proofs), **`project-layout.md`** (the tree).

## Guardrails (non-negotiable — each one cost a real debugging session to learn)

1. **Reflection into Memory Bank MUST be synchronous.** Use `add_events_to_memory(..., custom_metadata={"wait_for_completion": True})`. The default `add_session_to_memory()` is fire-and-forget; in a short-lived process the client closes before the write lands → total amnesia.
2. **Embeddings: `text-embedding-005` (768-dim), NOT `gemini-embedding-001` (3072-dim).** Firestore's vector index caps at 2048 dims; 3072 fails the index build.
3. **Cloud Scheduler → Cloud Run Job uses `--oauth-service-account-email`, NOT `--oidc-*`.** The target is a Google API (the Run Admin API); `--oidc` looks right and silently never fires.
4. **Every trajectory carries an `outcome` AND its resolution / root cause.** The dream is only as good as the outcome signal; without a real root cause it distills vague meta-advice.
5. **Every memory chunk carries a `scope`** (`private` | `agent_shared` | `harness_shared`) **and a `source_trajectory`; scope is enforced at retrieval** (no cross-agent or cross-tenant leakage). `source_trajectory` lets you trace and revoke a poisoned or wrong lesson.
6. **Least-privilege service account; ADC auth — no embedded JSON keys.**

## Reference implementation

A complete, runnable example lives at **https://github.com/cuppibla/reflective-memory-demo** — one worked instance (a support agent); `learn-the-user/` = Memory Bank, `learn-the-job/` = harness + the dream. Point the user there to run the demos, or to diff against what you generate.
