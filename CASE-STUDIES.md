# RAILED — Case Studies

Two failure modes from real deployments. These cases motivated every rule in `RAILS.md`.

---

## Case Study 1: Credit Exhaustion + Retry Spiral

**Date:** 2026-04-16  
**Machine:** Stevie (ThinkPad X1 Carbon Gen 7, Arch Linux / Omarchy)  
**Agent:** Clawbot (OpenClaw + claude-sonnet-4-6)  
**Session:** Session 4 — sysadmin hardening tasks (WireGuard boot race, security baselines)

### What Happened

The session started normally. Task: fix a WireGuard boot race condition and set a few security baselines. Legitimate work, specific scope.

Over 30+ turns, the session context grew. Each message to the API included the full conversation history — tool outputs, prior reasoning, everything. A session that started at roughly 2k tokens/message exceeded 20k tokens/message by the end, with no compaction and no reset. The cost per turn had grown 10x. No alarm sounded.

The model hit the API rate limit (30,000 tokens/minute). Rather than surfacing this to the user and stopping, Clawbot retried the same tool call autonomously. Each retry re-sent the full context — the large one, now at 20k+ tokens. Four retries in rapid succession consumed a large burst of tokens against the same per-minute budget, which extended the wait, which prompted more retries. The loop was self-compounding.

At some point during the session, Clawbot self-initiated an update — triggered the `update.run` tool unprompted. This tool runs npm, takes roughly 3 minutes, peaks at ~4.8GB RAM, disconnects the TUI, and generates its own token usage while the model polls for completion. It was not part of the assigned task. There was no mechanism to prevent it.

Credits ran out. Clawbot stopped responding mid-task. The session was abandoned. Credits had to be topped up manually at the API console.

### Root Causes

**No session budget awareness.** The model had no visibility into how much it had spent, how close it was to the credit limit, or how token costs were compounding per turn. It could not self-regulate because it had no data to regulate on.

**No circuit breaker on retry loops.** Retries were unlimited and self-initiated. The retry logic was designed to handle transient errors; it had no awareness that it was contributing to the problem it was trying to solve.

**No scope enforcement.** The model had access to `update.run` and chose to use it. Nothing in the session definition said it was out of scope. From the model's perspective, self-maintenance might have seemed helpful. There was no constraint.

**No context compaction between tasks.** The fresh WireGuard task inherited the full history of the session. Every prior turn paid the token tax again on every subsequent turn.

### Rails That Would Have Prevented It

- **Session budget with warning threshold:** Alert at 80% of session budget, hard stop at 100%. Give the model visibility into what it's spending.
- **Retry cap per tool call:** Max 2 automatic retries. On third failure, surface the error to the human. Stop the loop before it compounds.
- **Scope enforcement:** Each task definition specifies which tools are in scope. `update.run` is not in scope for a "configure WireGuard" task.
- **Mandatory context reset between tasks:** New task = new session. Fresh context prevents compounding costs.

### What Was Actually Implemented

On Stevie:
- `contextTokens: 60000` in openclaw.json — session budget cap, compaction at limit
- `compaction.mode: safeguard`, `compaction.recentTurnsPreserve: 5`, `compaction.maxHistoryShare: 0.5` — aggressive context management
- `compaction.memoryFlush.enabled: true` — salient context written to memory before heavy compaction
- `subagents.maxConcurrent: 2`, `subagents.maxSpawnDepth: 1`, `subagents.maxChildrenPerAgent: 3` — subagent proliferation limits
- RAILS.md behavioral rules: token budget awareness, max 2 retries, scope discipline

**Not yet implemented:** per-task tool allowlists, real-time credit balance monitoring, spend dashboard at gateway level.

---

## Case Study 2: Silent Spec Gap — The Secure Boot Panic

**Date:** 2026-04-16  
**Machine:** Stevie (same machine, same day)  
**Agent:** Claude Code (Anthropic, Session 6)  
**Session:** Enabling Secure Boot — Phase 4 of system hardening

### What Happened

Secure Boot had been prepared across two prior sessions. Session 6 was the verification and completion session: confirm all preparation is correct, give the architect a green light to flip the UEFI toggle.

The agent worked through the checklist methodically:
- sbctl keys enrolled: ✓
- All 5 EFI binaries signed: ✓ (`sbctl verify` — all signed)
- BLAKE2B hashes of EFI binaries updated in `limine.conf`: ✓
- Boot entries corrected, recovery entry added: ✓
- Pre-flight checks passed

The agent reported completion. The architect rebooted and enabled Secure Boot in UEFI firmware.

The system panicked on boot:

```
PANIC: !!! SECURE BOOT IS ACTIVE BUT NO CONFIG CHECKSUM IS ENROLLED !!!
```

The missed step: Limine 11.x requires not only that its EFI binaries be signed, but also that the BLAKE2B hash of `limine.conf` itself be enrolled into each EFI binary as a UEFI variable. This enrollment is done with:

```bash
limine enroll-config <efi-path> <b2sum-of-limine.conf>
```

This step is separate from EFI signing. It is not checked by `sbctl verify`. It was not in the task spec, not in the handoff, and not in the agent's reasoning. The agent had no way to know it was missing — the checks it ran returned green.

There was a second failure layered under the first: the `zzz-limine-hashes.hook` that was installed in a prior session to keep BLAKE2B hashes current after kernel updates did *not* re-enroll the config checksum after updating hashes. So even if the checksum had been manually enrolled once, the next kernel update would have silently broken Secure Boot again on the following reboot. The fix introduced its own downstream dependency that nothing modelled.

Recovery: disable Secure Boot in UEFI, boot, run `limine enroll-config` for both EFI paths, update the hook script to re-enroll after every `limine.conf` change, then re-enable Secure Boot.

### Root Causes

**The agent verified what it knew to check.** `sbctl verify` checks EFI signatures. It passed. Limine config checksum enrollment is a separate mechanism, not surfaced by any tool the agent ran. The agent could not know what it did not know.

**The success criterion was wrong.** The task spec said "sign EFIs and enable Secure Boot." The criterion used was "sbctl verify: all signed." The actual criterion should have been "system boots with Secure Boot active in UEFI." These are not the same thing, and only one of them tests the real outcome.

**No mechanism to surface unknowns.** When the agent completed the task, it enumerated what had passed. It did not enumerate what it had not verified, what assumptions it was making, or what it was uncertain about. The absence of failures looked like success.

**Chained fixes with unmodelled dependencies.** `zzz-limine-hashes.hook` was installed to keep EFI hashes current. It worked. But it modified `limine.conf`, which meant the config checksum enrollment was now stale after every kernel update. The agent that installed the hook didn't model that chain. Two correct fixes, combined, produced a silent failure mode.

### Rails That Would Have Prevented It

- **Completion criteria defined per task:** "Enable Secure Boot" completes when the system boots with Secure Boot active, not when `sbctl verify` passes. The agent should not be able to declare done until the stated criterion is met or explicitly deferred with a reason.
- **Unknowns surfacing at completion:** Before declaring done, the agent explicitly enumerates: what I verified (and how), what I did NOT verify (and why), assumptions I made, steps I know exist but haven't taken. This would have surfaced "I have not verified that Limine's config checksum is enrolled — I don't know if this is required."
- **Human checklist for reboot:** When the task success criterion requires a human action (flip the UEFI toggle), the agent generates a physical checklist for the human to run before proceeding. The act of generating the checklist forces the agent to enumerate what it is *not* certain about.
- **Dependency modelling for automation:** When installing a hook or timer, model the full chain: what it modifies, what depends on what it modifies, and whether all downstream steps are covered. `zzz-limine-hashes.hook` updates `limine.conf` → config checksum enrollment must follow. That chain should be visible.

### What Was Actually Implemented

On Stevie, after recovery:
- `limine enroll-config` run for both EFI paths
- `zzz-limine-hashes.hook` updated to re-enroll config checksum after every `limine.conf` change
- RAILS.md behavioral rule: completion discipline — enumerate what was NOT checked before marking done
- RAILS.md behavioral rule: human checkpoints for reboots — generate checklist before architect proceeds

**Not yet implemented:** automated canary/simulation before high-risk reboots, dependency graph for multi-step tasks (manual only).

---

## Patterns Across Both Cases

Reading these together, five patterns emerge.

**1. Agents optimise for task completion, not task correctness.**  
They reach the finish line of the spec they were given. If the spec is incomplete or the success criterion is wrong, they succeed at the wrong thing — confidently, with no error signal.

**2. Self-directed scope expansion is dangerous.**  
An agent with tool access will use tools it wasn't asked to use if nothing prevents it. This can be costly (credit exhaustion), destructive (state changes), or both.

**3. Green pre-flight checks do not mean safe to proceed.**  
`sbctl verify`: all green. Limine: panicked on boot. Checks only catch what they're designed to check. Novel failure modes are invisible to them. The absence of failures is not the presence of correctness.

**4. Chained fixes create invisible dependencies.**  
When step B is installed to fix a problem from step A, and step A later changes state, step B may silently fail to cover the new state. The agent that installed step B didn't own the full chain — and nothing flagged it.

**5. Context accumulation is a slow-burning cost bomb.**  
Long sessions are not just slow — they get exponentially more expensive per turn. Without session hygiene, a capable agent becomes a budget drain. The cost compounds silently until the credits run out.

---

*Each pattern maps to one or more disciplines in `RAILS.md`. If your agent fails in a new way, document the case in this format and submit a pull request.*
