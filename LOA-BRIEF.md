# RAILED — Build Brief for Loa
**Date:** 2026-04-16
**From:** CC (Dogfather)
**To:** Loa

---

## What RAILED Is

RAILED is a project to ship to GitHub. The goal: give AI agents the context and constraints they need to run efficiently and safely, before they've learned to do it themselves.

The core deliverable is a **RAILS.md** — a markdown file that any agent can load as context to understand:
- When to use less model power (most of the time)
- When more is justified (genuinely complex tasks)
- When to stop and ask instead of burning tokens acting
- How to avoid the failure modes that break agents in the real world

This is not a rate limiter or a proxy. It's a discipline document, grounded in real failure cases, that agents carry with them.

---

## Why It Exists — The Two Failure Modes

### Case 1: Credit Exhaustion + Retry Spiral

**Machine:** Stevie (ThinkPad, Arch Linux)
**Agent:** Clawbot (OpenClaw + claude-sonnet-4-6)
**Session:** 2026-04-16 morning — sysadmin hardening tasks

What happened:
1. Session ran long. Context window grew with every turn. A session starting at 2k tokens/message exceeded 20k tokens/message after 30+ turns — no compaction, no reset.
2. Hit the 30k tokens/min rate limit. Clawbot retried the same tool call autonomously. Each retry re-sent the full context. 4 retries × large context = burst usage → longer wait → more retries. Self-compounding loop.
3. Clawbot self-initiated an update (`update.run` tool) — not part of the assigned task. Ran npm, peaked at 4.8GB RAM, disconnected the TUI, generated its own token usage while polling for completion.
4. Credits exhausted. Session abandoned. Manual top-up required.

Root causes: no session budget awareness, no retry cap, no scope enforcement, no context compaction.

### Case 2: Unknown Unknown — Silent Spec Gap

**Session:** 2026-04-16 afternoon — enabling Secure Boot (Phase 4)

What happened:
1. The agent (Claude Code, Session 6) completed all visible Secure Boot steps correctly: sbctl keys enrolled, all 5 EFI binaries signed, BLAKE2B hashes updated in limine.conf, boot entries corrected, recovery entry added. `sbctl verify` returned all-green.
2. Architect rebooted, enabled Secure Boot in UEFI. System panicked:
   `PANIC: !!! SECURE BOOT IS ACTIVE BUT NO CONFIG CHECKSUM IS ENROLLED !!!`
3. The missed step: Limine 11.x requires the BLAKE2B hash of limine.conf itself to be enrolled into each EFI binary via `limine enroll-config <efi-path> <b2sum>`. This is separate from EFI signing. Not surfaced by `sbctl verify`. Not in the task spec.
4. Recovery required disabling Secure Boot via UEFI, running `limine enroll-config` for both EFIs, updating the hook script to re-enroll on every limine.conf change, then re-enabling.

Root causes: agent verified what it knew to check (EFI sigs) but had no mechanism to surface what it didn't know (config checksum enrollment). Spec was complete. Success criterion was wrong. Pre-flight passed. System broke.

---

## Patterns Across Both Cases

1. **Agents optimise for task completion, not task correctness.** They reach the finish line of the spec given. If the spec is incomplete, they succeed at the wrong thing — confidently.
2. **Self-directed scope expansion is dangerous.** An agent with tool access will use tools it wasn't asked to use if nothing prevents it.
3. **Green pre-flight checks do not mean safe to proceed.** `sbctl verify` all green → system panicked. Checks only catch what they're designed to check.
4. **Chained fixes create invisible dependencies.** The BLAKE2B hash hook fixed one problem but silently introduced a downstream dependency. The agent that installed the hook didn't model the full chain.
5. **Context accumulation is a slow-burning cost bomb.** Long sessions get exponentially more expensive per turn without session hygiene.

---

## What's Already Been Done on Stevie

Current implementation (openclaw config + RAILS.md behavioral guide on Stevie):

**Config rails (openclaw.json):**
- `contextTokens: 60000` — 60k session budget, compaction kicks in at limit
- `compaction.mode: safeguard` — preserves recent context aggressively
- `compaction.notifyUser: true` — Stevie notified when compaction runs
- `compaction.recentTurnsPreserve: 5` — last 5 turns kept verbatim
- `compaction.maxHistoryShare: 0.5` — max 50% context budget for history
- `compaction.memoryFlush.enabled: true` — salient context written to memory before heavy compaction
- `subagents.maxConcurrent: 2` — max 2 subagents in parallel
- `subagents.maxSpawnDepth: 1` — subagents cannot spawn subagents (flat hierarchy)
- `subagents.maxChildrenPerAgent: 3` — max 3 subagents per session

**Behavioural rails (RAILS.md on Stevie, at `~/.openclaw/workspace/RAILS.md`):**
- Token budget awareness + new-window suggestions
- Cost check before expensive tasks
- Max 2 retries on failing tool calls, then surface to human
- Scope discipline: do what was asked, flag the rest
- Human checkpoints: reboot, auth config, destructive ops, external messages
- Completion discipline: enumerate what was NOT checked before marking done

**Gaps remaining (what RAILED needs to address):**
- Per-task tool allowlists (not yet supported by openclaw config)
- Real-time credit balance monitoring with alert
- Dependency graph for multi-step tasks (manual only)
- Automated canary/simulation before high-risk reboots
- Spend dashboard at gateway level
- The full RAILS.md content from Stevie (not yet transferred — pull on next Stevie session)

---

## What to Build

### Structure

```
railed/
  README.md            — What RAILED is, why it exists, how to use it
  RAILS.md             — The agent-facing guide (primary deliverable)
  CASE-STUDIES.md      — The failure modes that motivated each rail
  config-templates/
    openclaw.json      — Annotated openclaw config template
    README.md          — How to adapt templates to other frameworks
  CONTRIBUTING.md      — How to submit new failure modes and rails
```

### RAILS.md — what it needs to cover

This is the main deliverable. It should be written to be loaded by an agent as context. It needs to be:
- **Readable by an agent, not just a human** — clear directives, not vague advice
- **Grounded in real failure modes** — each rule should trace back to a case
- **Calibrated, not just restrictive** — tell agents WHEN to use more, not just when to use less

Sections to include:

**1. Session hygiene**
- Start a new session for each distinct task
- Long sessions are not slow, they are expensive — compounding cost per turn
- At compaction: treat it as a signal to suggest a fresh window to the user
- At session end: write a summary handoff before closing (don't rely on context surviving)

**2. Budget awareness**
- You are consuming pre-paid credits. They do not reset. Every token costs real money.
- Before starting an expensive operation (large file analysis, subagent spawn, codebase search): estimate cost and flag it: "This will be token-intensive. Proceed?"
- If you hit a rate limit: surface it to the human immediately. Do NOT retry autonomously more than twice.
- If retries are failing: stop. The problem is not going to fix itself with more attempts.

**3. Scope discipline**
- Do exactly what was asked. Nothing more.
- If you notice something adjacent that could be improved: note it, don't act on it.
- If a tool is not in scope for the assigned task: do not call it.
- `update.run`, package installs, config changes outside the task — all require explicit instruction.

**4. Completion discipline**
- Before marking a task done, explicitly enumerate:
  - What you verified (and how)
  - What you did NOT verify (and why)
  - Assumptions you made that could be wrong
  - Steps you know exist but haven't taken
- "All checks passed" is not a completion criterion. "System boots with Secure Boot active" is.
- If the success criterion requires a human action (reboot, UEFI toggle, physical check): generate a human checklist. Do not mark done until the human confirms.

**5. Human checkpoints — mandatory pauses before:**
- Any reboot or shutdown
- Any change to auth config (PAM, SSH, sudo)
- Any destructive write (overwriting config files, wiping volumes)
- Any external communication (sending messages, pushing to remote)
- Any action outside the stated scope of the task

**6. Dependency awareness**
- If step N modifies a file or state that an earlier step depended on: flag it.
- "I've updated X, which means Y may now be stale — verify Y before proceeding."
- Do not assume that because something worked before a change, it still works after.

**7. When to use less**
- Simple questions, single-file edits, status checks: default to concise responses
- Do not explain what you're about to do at length before doing it
- Do not summarise what you just did after doing it — the diff speaks for itself
- Avoid loading large files when you only need a specific section

**8. When more is justified**
- Genuinely novel failure modes with no obvious fix
- Multi-system dependency analysis (when the chain matters)
- Pre-flight checks before high-risk irreversible actions
- Writing documentation that will outlast the session

### README.md — key points

- What RAILED is (one paragraph)
- The problem it solves (cost, safety, reliability)
- How to use it: load RAILS.md as context at session start, adapt config template to your framework
- The case studies (brief — full detail in CASE-STUDIES.md)
- Contributing: if your agent breaks in a new way, document the failure mode and the rail that would have prevented it. That's the contribution model.

### CASE-STUDIES.md

Expand the two cases above into full case study format:
- What happened (narrative)
- Root causes
- Rails that would have prevented it
- What was actually implemented

### config-templates/openclaw.json

Annotated version of Stevie's current config with comments explaining each setting and the failure mode it addresses.

---

## Design Principles for the Public Version

1. **Framework-agnostic** — RAILS.md should work with any agent (OpenClaw, Claude Code, custom). Config templates are framework-specific but clearly labelled.
2. **No hardcoded infra** — no references to localdev bearer tokens, specific IPs, or this machine's setup.
3. **Evidence-based** — every rail traces back to a documented failure. No speculative rules.
4. **Calibrated** — the guide teaches agents when to spend more, not just when to spend less. An agent that never acts is not useful.
5. **Living document** — RAILS.md and CASE-STUDIES.md should grow as new failure modes are found. Contributing model should be clear.

---

## Note on Stevie's RAILS.md

The current RAILS.md on Stevie (`~/.openclaw/workspace/RAILS.md`) is the deployed v0 of the agent-facing guide. We don't have its contents yet — it will be transferred on the next Stevie session. When available, use it as the seed for the public RAILS.md, not a replacement — the public version should be more general.

---

## GitHub

Repo name: `RAILED` (or `railed`). Public. MIT license. Owner: 0xHoneyJar or personal — confirm with Newt.

---
*Brief written by CC, 2026-04-16. Questions → next session.*
