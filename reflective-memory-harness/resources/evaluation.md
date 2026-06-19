# Proving it works — the three proofs

The build is the easy part. The proof is the point: a stored "lesson" means nothing until you show it changed the agent's next decision. **Do not declare the build done until these pass.** Three proofs, in order.

## Proof 1 — cross-session user memory (learn-the-user)

- Session 1: the user states durable facts (e.g. *"I'm vegetarian, planning Tokyo in April, budget-friendly."*).
- Session 2 (a **new** session): ask something that needs those facts without repeating them.
- **Pass** = the answer uses the facts. This also catches the fire-and-forget amnesia trap (Guardrail 1): if the memory write was async, session 2 starts blank.

## Proof 2 — the lesson changed the decision (learn-the-job): the honest A/B

This is the proof people fake. Get it right or the demo proves nothing. Four design rules keep it honest:

1. **The agent's base instruction must contain ZERO domain knowledge about the fix.** The lesson ("verify permissions before generic troubleshooting") must be able to reach the agent *only* as a dream-distilled memory — never baked into the prompt. If it's in the instruction, the A/B is meaningless.
2. **Change exactly one variable: whether the job lesson is retrievable.** Hold the model, instructions, user facts, and task shape identical between the OFF and ON runs.
3. **Use a symptom-only task** for the test (e.g. *"the billing page is blank"*) — no wording that hands over the diagnosis.
4. **Score the ACTION with an LLM judge, majority-of-3** — not a human eyeball, not keyword matching. The judge rates the recommended action, not name-dropped context.

Expected result:

| condition | retrieved | primary action | result |
|---|---|---|---|
| job memory **OFF** | 0 lessons | generic (clear cache / try another browser) | ❌ fail (0/3) |
| job memory **ON** | 1 lesson | check account roles/permissions first | ✅ pass (3/3) |

The lesson is the only variable, and the outcome flips. That is the evidence chain:

> experience → recorded outcome → distilled lesson → scoped retrieval → changed action → judged result

## Proof 3 — a second agent inherits (it's harness, not agent code)

- Add a **different** agent that has never seen the failed task and has no private history with it.
- Give it a fresh task in the same domain.
- **Pass** = it retrieves the `harness_shared` lesson and acts on it on first contact, **and** a `private` lesson owned by the first agent stays invisible to it.
- Sharing without isolation is a leak, not a success — check both halves.

## The data lesson that makes or breaks Proof 2

If the failed trajectory only records *"the answer was generic,"* the dream distills generic meta-advice. The **resolution / root cause** on the trajectory is the learning signal (Guardrail 4) — garbage outcome signal, garbage lesson. Real ticketing systems log resolutions on close; that resolution *is* the signal. Log it, or Proof 2 will never be convincing.
