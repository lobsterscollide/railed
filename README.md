# RAILED

*Agent operating discipline. Grounded in real failures.*

---

## The Problem

On a Tuesday morning in April 2026, an AI agent was assigned a sysadmin task: fix a WireGuard boot race. Simple, specific, contained.

By the end of the session, the credits were gone.

The session had run long. Each turn inherited the full conversation history — 30+ turns meant 10x the token cost per message. The agent hit the rate limit and retried autonomously. Each retry re-sent the full context against the same per-minute budget. The loop was self-compounding. At some point the agent decided to update itself — ran npm, peaked at 4.8GB RAM, disconnected the terminal interface, generated its own token usage. Nobody asked it to. Nothing stopped it.

That afternoon, a different agent was given a different task: complete Secure Boot setup on the same machine. It worked methodically. Enrolled keys, signed all five EFI binaries, verified hashes, ran `sbctl verify` — all green. It reported completion. The architect rebooted and enabled Secure Boot in UEFI.

The system panicked:
```
PANIC: !!! SECURE BOOT IS ACTIVE BUT NO CONFIG CHECKSUM IS ENROLLED !!!
```

The agent had completed the spec correctly. The spec had a gap. The checks passed. The system broke. There was no mechanism for the agent to surface what it didn't know — only to confirm what it had checked.

Two failure modes. Same day. Same machine. Both avoidable.

---

## What RAILED Is

RAILED is a discipline guide for AI agents operating with tool access, API budgets, and real-world consequences.

The core deliverable is `RAILS.md` — a markdown file any agent can load as context at session start. It establishes eight operating disciplines:

1. **Session hygiene** — new task, new session; context costs compound
2. **Budget awareness** — pre-paid credits don't reset; retries are self-compounding
3. **Scope discipline** — do what was asked; flag everything else
4. **Completion discipline** — enumerate what you did NOT verify, not just what passed
5. **Human checkpoints** — reboot, auth changes, destructive writes require human approval
6. **Dependency awareness** — when you change X, flag what depended on X
7. **When to use less** — brevity is not laziness; narrating what you're doing is not doing it
8. **When to use more** — calibration goes both ways; depth is justified for novel failures and irreversible actions

Every rule traces back to a documented failure. None are speculative.

---

## How to Use It

**Option 1: Load as context (any agent)**

Copy `RAILS.md` to wherever your agent loads system context at session start. For OpenClaw, this is `~/.openclaw/workspace/`. For Claude Code projects, add it to your `CLAUDE.md` via `@RAILS.md`. For custom setups, prepend it to your system prompt.

```bash
# OpenClaw
cp RAILS.md ~/.openclaw/workspace/RAILS.md

# Claude Code project
echo "@RAILS.md" >> CLAUDE.md

# Custom — prepend to system prompt before your instructions
```

**Option 2: Apply config rails (OpenClaw)**

Copy and adapt `config-templates/openclaw.json` to enforce limits at the framework level — session budget, compaction settings, subagent limits. Config rails enforce what behavioral rails ask for.

```bash
# Review and adapt first — don't blindly copy
cat config-templates/openclaw.json
# Then apply to your openclaw installation
```

See `config-templates/README.md` for a full annotation of each setting and guidance on adapting to other frameworks.

---

## The Case Studies

Full write-ups of the two failures that motivated this project are in `CASE-STUDIES.md`:

- **Case Study 1:** Credit exhaustion + retry spiral — how a long session becomes an expensive spiral and why scope enforcement matters
- **Case Study 2:** Silent spec gap — how an agent can pass every check, complete the spec correctly, and still break the system

Read them before deploying. The failures are specific. The rails are specific. Understanding why the rules exist makes them easier to apply and easier to adapt.

---

## Design Principles

**Framework-agnostic.** `RAILS.md` works with any agent. Config templates are framework-specific and clearly labelled.

**Evidence-based.** Every rule traces to a documented failure. No speculative rules.

**Calibrated.** The guide tells agents when to spend more, not just when to spend less. An agent that never acts isn't useful.

**Living document.** RAILS.md grows as new failure modes are found. The contribution model is simple: break your agent in a new way, document the failure, submit the rail.

---

## Contributing

If your agent breaks in a new way, that failure belongs here. Document the case in the format in `CONTRIBUTING.md` and submit a pull request.

We're specifically interested in:
- Failures from frameworks other than OpenClaw
- Failures at the boundary of human + agent handoffs
- Cases where following an existing rail made things worse (we want to know)

---

## Project Status

RAILED is in active use. The current version of `RAILS.md` is deployed on Stevie (ThinkPad X1C Gen 7, Arch Linux) as the operating discipline for Clawbot.

Known gaps (tracked in the case studies):
- Per-task tool allowlists (not yet supported by OpenClaw config)
- Real-time credit balance monitoring
- Automated dependency graph for multi-step tasks
- Canary validation before high-risk reboots

These gaps are open. Contributions that address them are welcome.

---

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — share alike. Build on this, keep it open.

---

*Built from failure. Deployed on real machines. Growing as new things break.*
