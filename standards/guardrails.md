# Guardrails

The canon of what the agent may do on its own, what it must ask about, and what it
must never do. This document is the **source of truth**; the hooks in each repo's
`.claude/settings.json` *enforce a subset of it*. A line written here but not enforced
is a rule of discipline, not a wall — so each item below is marked:

- 🔒 **enforced** — a PreToolUse hook hard-blocks this. The agent cannot cross it.
- 📝 **written** — relies on the agent reading and obeying. Not mechanically blocked.

The split matters. Don't mistake a 📝 line for protection. If something genuinely
must not happen, it belongs in 🔒, not here in prose alone.

## Always do — the autopilot defaults

No permission needed; these are the baseline the agent runs on. No hook required —
the cost of *not* doing them is low, and they speed everything up.

- Read code, docs, and config freely.
- Run read-only inspection: `git status`, `git diff`, `git log`, tests, type checks,
  `task` targets that don't mutate state.
- Search the codebase before writing new code — never duplicate what exists.
- Format and lint files it edits.
- Write to `tmp/` and scratch space.

## Ask first — anything with consequences

Stop and get explicit human approval before acting. The common thread: a *cost*, a
*one-way door*, or an action whose blast radius the agent can't fully see.

- 📝 Installing or adding dependencies.
- 📝 Schema changes / migrations on any database (including the hive substrate).
- 📝 Anything that spends money or calls a paid API at non-trivial volume.
- 📝 Bulk file moves, renames, or deletes outside `tmp/`.
- 📝 Adding a new ADR to `docs/decisions/` or editing an existing one — locked
  decisions are changed deliberately, with a human in the loop, never silently.
- 📝 Pushing branches, opening PRs, anything that leaves the local machine.

These are 📝 today because "ask first" needs an interactive prompt, which inline hooks
can't do cleanly. They're enforced either by Claude Code's normal permission prompts
or by your own guard script on the PreToolUse wiring. Until that script exists, this
tier rests on discipline — promote the high-stakes ones to 🔒 if discipline isn't enough.

## Never do — the lines that don't get crossed

Hard stops. These are where rules get *imposed*, not requested.

- 🔒 Writing to live `secrets.env` (the `.example` is fine). Secrets are never edited
  by the agent and never echoed into logs, output, or commits.
- 🔒 Touching `hive/data/` directly. The substrate's data is off-limits to edits;
  it changes only through the running system, never by hand-editing files.
- 🔒 Destructive shell: `rm -rf`, dropping tables/databases.
- 🔒 Force-pushing (`git push --force` / `-f`) — history is not rewritten by the agent.
- 🔒 Running `scripts/sync.sh` in any mode that reaches the remote machine. The remote
  is production-adjacent; syncing to it is a deliberate human action, never an agent's.
- 📝 Sending real communication (email, messages) to real people.
- 📝 Disabling or editing these guardrails or the hooks that enforce them.

The 🔒 items are blocked by PreToolUse hooks (`exit 2`) in `hive/.claude/settings.json`
and `swarm/.claude/settings.json`. The harness runs those hooks outside the agent's
control, so they can't be reasoned around. The 📝 items in this tier are the next
candidates to harden into a guard script — they're "never" in intent but not yet in
machine.

## Why this lives in two places

Splitting *policy* (here) from *mechanism* (the hooks) is deliberate, not redundant.
The policy explains the reasoning and stays readable; the hook is a terse pattern that
enforces it. When they drift, this document wins on intent and the hook wins on what
actually happens — and any gap between them (a 📝 that should be 🔒) is a known piece
of work, not a surprise.
