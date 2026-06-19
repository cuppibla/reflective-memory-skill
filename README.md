# reflective-memory-harness — an agent skill

A portable **agent skill** that teaches your agentic IDE (Google Antigravity, Claude Code) to build an AI agent with **reflective memory** on Google Cloud — and to dodge the traps that bite everyone the first time.

Hand it to your IDE, ask for *"a reflective-memory support agent on Google Cloud,"* and it scaffolds:

- **Learn-the-user** — durable per-user facts via **Vertex AI Memory Bank**.
- **Learn-the-job** — lessons distilled from the agent's own failed tasks via **Firestore + a scheduled "dream"** (a Cloud Run Job that reflects offline).
- a single **`recall()`** that merges both stores into one context.

It's the open, build-your-own equivalent of a managed background-memory service — your own *Dreams*, on your stack.

## What's inside

```
reflective-memory-harness/
  SKILL.md                      the skill — architecture, build plan, and non-negotiable guardrails
  resources/
    gotchas.md                  the verified traps + fixes (fire-and-forget amnesia, the 3072-dim wall, --oauth)
    reference-snippets.md       the load-bearing, verified API calls
```

## Why it's worth installing

The skill encodes three traps that each cost a real debugging session:

1. **Fire-and-forget amnesia** — the default memory write returns before it lands → use the synchronous form.
2. **The 3072-dim wall** — Firestore's vector index caps at 2048 → use `text-embedding-005` (768-dim).
3. **`--oauth`, not `--oidc`** — the scheduled dream silently never fires otherwise.

An IDE that loads this skill avoids all three from the first prompt.

## Install

**Claude Code** — copy the skill directory into your skills folder:
```bash
cp -r reflective-memory-harness ~/.claude/skills/        # personal
# or: cp -r reflective-memory-harness .claude/skills/    # this project only
```

**Google Antigravity** — copy `reflective-memory-harness/` into your Antigravity skills directory (Workspace or Global Skills). See Antigravity's Skills docs for the exact location on your version.

Then open a project and ask, in plain English: *"Build me a reflective-memory support agent on Google Cloud."* The skill loads when relevant and keeps the build on the rails.

## See also

- **Reference implementation (runnable):** https://github.com/cuppibla/reflective-memory-demo
- **The walkthrough:** "Vibe-Code Your Own Agent 'Dreams' in Google Antigravity" (Part 5 of the Reflective Memory series).

## License

[MIT](LICENSE) © 2026 Annie Wang
