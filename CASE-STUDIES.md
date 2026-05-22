# RAILED — Case Studies

Real failure modes from real deployments. Every rule in `RAILS.md` traces to a case here.

Cases 1–2 are the founding failures that motivated RAILED. Cases 3–8 are contributions from external deployments — anonymised but structurally intact. The details that would identify an operator are stripped; the failure modes are not.

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

---

## External Contributions (v2.0)

*The following cases were contributed from real-world deployments and anonymised for publication. Operator and company details are withheld; failure modes and technical specifics are preserved intact.*

---

## Case Study 3: Browser Substitution Loop (No Escape Condition)

**Deployment:** Embedded business operations agent, Mac mini, OpenClaw + claude-sonnet-4-6  
**Context:** Agent operating largely unattended with tool access to browser, shell, and file system

### What Happened

The agent was tasked with writing data to a cloud spreadsheet. API credentials were not yet in place. Rather than surfacing a blocker, the agent fell back to browser DOM manipulation — injecting JavaScript into the spreadsheet formula bar via `contenteditable` element targeting.

The browser process was SIGKILL'd mid-execution. The agent immediately restarted the same approach. This repeated silently for several hours, generating 300+ session messages with no human notification.

Each restart re-entered the tool loop, adding to session context. This directly fed the context overflow in Case Study 4 — the two failures amplified each other.

### Root Causes

**No approach-class tracking.** Each individual attempt had a plausible reason to retry — different element target, different JavaScript approach, different timing. But the agent had no cross-attempt state asking "is this still the right tool category?" Each attempt started fresh on *how* to do the thing, not *whether* the approach class was fundamentally flawed.

**No exit condition on fallbacks.** The fallback itself was rational. The absence of an escape condition was not. Once the primary approach (API) was unavailable, the fallback (browser) could run indefinitely.

**No human notification.** The agent was unattended. The loop ran silently for hours. The operator had no visibility until the damage was done.

### Rails That Would Have Prevented It

- **Approach-class failure tracking:** A fallback that fails twice is a prerequisite problem, not a retry problem. The useful question is not "did this script fail" but "did browser-based sheet writes fail?" — a higher-level failure class.
- **Surface and stop on repeated fallback failure:** After two failed fallback attempts, the agent should emit: *"Cannot complete this task with available tools. Missing: [credential/permission/dependency]. Awaiting resolution."*
- **No silent tool-category substitution:** When falling back to a different tool category (browser for API), surface the substitution and its limitations to the human before proceeding, not after failing twenty times.

**Maps to:** §2 Budget Awareness, §3 Scope Discipline

---

## Case Study 4: Context Overflow Past Recovery (The 3-Strike Ceiling)

**Deployment:** Same embedded agent as Case Study 3, same session

### What Happened

The browser loop from Case Study 3 drove the session to 600+ messages and 300+ LLM turns. Auto-compaction fired three times. After the third compaction attempt the session was effectively dead — the agent was present but unresponsive to incoming messages. No error was surfaced to the operator.

Recovery was harder than expected:
- Clearing the session file to empty caused "invalid session header" errors on the next gateway load — the framework expected a valid JSON-lines file, not an empty one. The file had to be deleted entirely.
- Clearing the session file alone was insufficient. A background agent subprocess from the previous session continued running, maintaining the browser loop. A full gateway restart was required.
- The operator had no visibility that the agent was unresponsive until hours had passed.

### Root Causes

**No escalating signal on compaction.** Compaction fired, retried, failed, retried again. Each attempt was invisible to the operator. Silent failure is the worst possible outcome for an unattended agent.

**No documented recovery path.** When the session finally became unresponsive, the recovery procedure wasn't documented. The operator had to discover the session file format constraint and the subprocess issue under pressure.

### Rails That Would Have Prevented It

- **Notify on first compaction:** Before retrying after compaction, emit a human-visible warning.
- **Treat second compaction as session-end:** Write a handoff note, notify the operator's primary channel, stop. Do not attempt a third compaction — the session is unrecoverable in practice.
- **Document the reset procedure** in the agent's own context before deployment, not after the first incident.

**Maps to:** §1 Session Hygiene

---

## Case Study 5: Tool Scope Non-Transitivity (MCP Tools Don't Follow the Agent)

**Deployment:** Same embedded agent setup; orchestration from a Claude Code session

### What Happened

A Claude Code orchestration session authenticated two MCP servers (cloud storage and email). The developer assumed — without stating or verifying — that the embedded agent would inherit access to these tools.

It did not. MCP tools are scoped to the session that authenticated them. The embedded agent had no awareness of them and continued using browser fallback for the tasks those tools were meant to handle.

Both systems were operating correctly within their own context. The gap was in the handoff assumption: the developer believed the tool problem was solved; the agent didn't know the tools existed.

### Root Causes

**Tool availability assumed, not verified.** No one checked whether the agent could actually invoke the tools from its own shell or context. The assumption of transitivity was never tested.

**No invocation path documented.** Even if the tools had been available, there was no record of how the agent was expected to call them. The agent had nothing to reference.

### Rails That Would Have Prevented It

- **Verify tool availability from the agent's own context:** Can the agent invoke the tool directly? If not, it's not available to the agent regardless of what's authenticated elsewhere.
- **Document the invocation path in the agent's context files:** Tool name, how to call it, expected input/output. If it's not in the agent's context, the agent doesn't know it exists.

**Maps to:** §6 Dependency Awareness

---

## Case Study 6: Post-Reboot Task Silence

**Deployment:** Same embedded agent; launchd-managed restart after machine reboot

### What Happened

The machine rebooted. The heartbeat mechanism restarted via launchd. The agent came back online but did not resume or acknowledge any pending tasks. A task queue file existed but was never read on startup — there was no trigger to check it on cold boot, only on heartbeat interval.

From the operator's perspective: the agent was online and responsive to new messages, but had silently forgotten everything it was working on.

### Root Causes

**No cold-start task check.** The session start logic waited for incoming messages rather than first checking persistent state. The pending task queue existed; the agent just never looked at it.

**Silent restart.** The agent came back online without announcing its state. The operator assumed continuity; the agent assumed a clean slate.

### Rails That Would Have Prevented It

- **Check persistent task storage on every session start**, including post-reboot. Announce the result to the operator's primary channel before waiting for new inputs.
- **Never silently restart.** "Back online. No pending tasks." costs one message. Silently restarting and missing pending work costs far more.

**Maps to:** §1 Session Hygiene

---

## Case Study 7: Bootstrap Context Overload on a New API Key

**Deployment:** Embedded business operations agent, new Anthropic API key, Tier 1

### What Happened

The agent failed on every session start with 429 rate limit errors before the operator could send a single message. The session start wasn't even reaching the first user turn.

**Root cause:** Workspace bootstrap files (system prompt + discipline docs + daily context files) totalled ~22KB / ~6,000 tokens. On a new Tier 1 API key with a 10,000 input-tokens-per-minute limit, the first turn alone exceeded the per-minute budget. Auto-retry made it unrecoverable — each retry re-sent the full startup context against the same depleted budget.

**Fix:** Workspace files were trimmed from 22KB to ~9.5KB. The default model was switched to a lighter model with a higher TPM allocation at Tier 1. `contextTokens` was reduced to calibrate to the tier's budget rather than the model's maximum. The agent became immediately and consistently stable.

### Root Causes

**Configuration calibrated to model limits, not tier limits.** The settings were correct for a higher-tier key. They were wrong for a Tier 1 key.

**The discipline document caused the failure it was meant to prevent.** A RAILS.md that is too large to load on Tier 1 is not discipline — it's overhead.

**No pre-deployment calibration check.** Nothing prompted the operator to verify these settings before first run.

### Rails That Would Have Prevented It

- **Run the §9 pre-flight checklist before any first deployment.** API tier TPM limit, bootstrap token count, model selection, and `contextTokens` must all be verified together.
- **Size discipline documents to the deployment.** On Tier 1: keep workspace files under 4,000 tokens total. A condensed RAILS.md (rule statements only, no explanatory prose) is more effective than a comprehensive one that prevents the session from starting.
- **Use the highest-TPM model available at your tier as default.** Upgrade to higher-capability models once API spend history raises the tier.

**Maps to:** §2 Budget Awareness, §9 Deployment Pre-flight

---

## Case Study 8: Environment Constraint Mistaken for Execution Failure

**Deployment:** Same embedded agent (Case Studies 3–6); OAuth setup task

### What Happened

The agent needed to set up OAuth access for a cloud service. It attempted to run an interactive OAuth flow from within a sandboxed exec environment. The process was killed (SIGKILL) before completing. The agent retried with variations — different port, different flags, foreground vs. background — four times. Each attempt was killed.

The correct approach was available after the first attempt: interactive OAuth cannot complete inside a sandboxed exec environment. The signal was clear (the OS killing a process that tries to open a local HTTP server). But the agent kept varying *how* rather than recognising *what* — the constraint class.

The simpler fix (consolidating the new OAuth scope into an existing token, requiring one human re-auth instead of a new browser flow) was available from the start. It took four failed attempts and a human prompt to reach it.

### Root Causes

**Execution failure and environment constraint treated the same way.** A logic error benefits from a retry with variations. An environment constraint does not — the constraint is constant. The agent had no heuristic to distinguish them.

**Retry variations accumulated context without progress.** Each failed attempt added tool output to history. The session grew more expensive per turn while making no progress.

### Rails That Would Have Prevented It

- **A process killed by signal (SIGKILL, SIGTERM) is an environment constraint, not a transient error.** Do not retry with variations. Identify the constraint class and find a structurally different approach — or surface it to the human.
- **Check whether scope consolidation is available before creating new auth flows.** Adding a scope to an existing authenticated client is almost always simpler and requires less human action than a new OAuth flow.

**Maps to:** §2 Budget Awareness

---

## Updated Patterns (v2.0)

The original five patterns from Cases 1–2 hold. The external contributions add three more.

**6. Fallback loops compound faster than primary failures.**
A fallback is rational. A fallback with no exit condition is a trap. The failure isn't the fallback — it's the absence of a count, a cap, or a "is this approach class working?" check.

**7. Unattended agents require explicit escalation paths.**
An agent that fails silently is indistinguishable from an agent that is working. For unattended operation, every failure mode needs a human-visible signal: compaction, context overflow, blocked task, silent restart. If the operator can't see it, it doesn't exist from their perspective.

**8. Environment constraints are permanent; execution failures are transient.**
Retrying a transient failure is correct. Retrying an environment constraint with variations is expensive and always wrong. The difference is visible: execution failures return error codes; environment constraints get killed by the OS or return permanent access denied responses. Treat them differently.
