# Proving it works — the three proofs

The build is the easy part. The proof is the point: a stored "lesson" means nothing until you show it changed the agent's next decision. **Do not declare the build done until these pass.** Three proofs, in order.

## Proof 1 — cross-session user memory (learn-the-user)

- Session 1: the user states durable facts (e.g. *"I'm vegetarian, planning Tokyo in April, budget-friendly."*).
- Session 2 (a **new** session): ask something that needs those facts without repeating them.
- **Pass** = the answer uses the facts. This also catches the fire-and-forget amnesia trap (Guardrail 1): if the memory write was async, session 2 starts blank.

## Proof 2 — the lesson changed the decision (learn-the-job): the honest A/B

This is the proof people fake. Get it right or the demo proves nothing. Four design rules keep it honest:

1. **The agent's base instruction must contain ZERO domain knowledge about the fix.** Whatever the lesson teaches must be able to reach the agent *only* as a dream-distilled memory — never baked into the base prompt. If it's in the instruction, the A/B is meaningless.
2. **Change exactly one variable: whether the agent CONSULTS job memory.** Hold model, instructions, user facts, and task identical. **Make OFF a real toggle — the recall tool simply doesn't query the job store.** Do NOT rely on "run before the store is populated": on any real or shared store the lesson (or near-duplicates) may already exist, so OFF silently retrieves them and the A/B won't flip. The lesson can be present for BOTH runs; only *consulting* it differs. *(Verified live: the empty-store assumption gave a false OFF=pass; the toggle gave the true fail→pass flip.)*
3. **Use a symptom-only task** for the test — phrased so it does not hand the agent the diagnosis.
4. **Score the ACTION with an LLM judge, majority-of-3** — not a human eyeball, not keyword matching. The judge rates the recommended action, not name-dropped context.

Expected result:

| condition | retrieved | primary action | result |
|---|---|---|---|
| job memory **OFF** | 0 lessons | the uninformed default action | ❌ fail (0/3) |
| job memory **ON** | 1 lesson | the action the distilled lesson teaches | ✅ pass (3/3) |

*Worked example — the reference repo's support agent: a cold access ticket gets generic cache-clearing (fail); after the dream, it leads with "check the account's roles/permissions first" (pass).*

The lesson is the only variable, and the outcome flips. That is the evidence chain:

> experience → recorded outcome → distilled lesson → scoped retrieval → changed action → judged result

## When the win is the PATH, not the answer

A capable base model often already reaches the *right* answer without the lesson — it just takes a worse route (checks the wrong things first, wanders, more tool calls). A pure correctness A/B then won't flip (OFF and ON both pass), but the agent is still improving: it learned a **shorter trajectory**. So for a **multi-tool / multi-step** agent, **measure trajectory efficiency too** — tool-call count / path length, not only final-answer correctness. Hold everything constant, toggle job memory, and report **avg steps OFF vs ON** next to the correctness numbers. *(Verified live: a support agent's correctness held at 0 wrong either way, while avg tool-call steps dropped 3.0 → 1.8 once it retrieved the procedure — same answers, fewer dead-ends.)*

## Proof 3 — a second agent inherits (it's harness, not agent code)

- Add a **different** agent that has never seen the failed task and has no private history with it.
- Give it a fresh task in the same domain.
- **Pass** = it retrieves the `harness_shared` lesson and acts on it on first contact, **and** a `private` lesson owned by the first agent stays invisible to it.
- Sharing without isolation is a leak, not a success — check both halves.

## The data lesson that makes or breaks Proof 2

If the failed trajectory only records *"the answer was generic,"* the dream distills generic meta-advice. The **resolution / root cause** on the trajectory is the learning signal (Guardrail 4) — garbage outcome signal, garbage lesson. Real systems log an outcome/resolution on close; that resolution *is* the signal. Log it, or Proof 2 will never be convincing.
