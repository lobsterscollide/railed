# RAILED

*An agent discipline protocol. Efficient, cost-aware, security-conscious, privacy-conscious.*

---

## What RAILED Is

RAILED is a discipline protocol for AI agents that operate with tool access and real-world consequences.

Most agents are deployed without behavioral constraints. They don't know when to stop, when to ask, what's in scope, or how to judge whether a task is actually done. They optimise for completing the spec they were given — confidently, efficiently, and sometimes incorrectly.

RAILED gives agents the context they need before those problems occur. The core deliverable is `RAILS.md` — a markdown file any agent can load as context at session start. It's framework-agnostic, evidence-based, and built from real failure cases.

---

## What It Addresses

**Efficiency** — Agents that narrate instead of act, load entire files when they need one section, or carry 30 turns of history into a fresh task are wasting compute. RAILED teaches agents to calibrate effort to task value.

**Cost awareness** — Cost has two meanings here, and both matter:

1. *Compute cost* — what each task is worth in tokens. Some tasks justify depth; most don't. Knowing the difference is a discipline, not a setting.
2. *Financial cost* — when an agent has real spending power (procurement, inventory, billing, financial systems), it needs to understand budgets, value, and authorization before acting. The same principles that prevent a runaway sysadmin bot prevent a runaway purchasing bot.

**Security** — Agents with tool access can take actions outside their assigned scope, bypass checkpoints, and self-initiate operations that were never requested. RAILED enforces scope discipline and mandatory human checkpoints before high-risk actions: reboots, auth changes, destructive writes, external communications.

**Privacy** — Agents operating near sensitive data need to know what not to surface, what not to log, and when to stop and ask rather than proceed. RAILED makes these explicit behavioral constraints, not assumptions.

---

## Who It's For

**New users** getting started with AI agents — before you've learned what a runaway bot looks like, RAILED gives you the rails. The failure modes are documented. The constraints are there from session one.

**Power users and homelabbers** running agents on real machines with real consequences. Sysadmin tasks, infrastructure changes, anything where a wrong action costs time to recover from.

**Teams and companies** giving agents access to systems that hold sensitive or financially significant data — logistics, inventory, finance, customer records. An agent without behavioral discipline is a liability at scale. RAILED is the discipline layer your deployment is missing.

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
```

See `config-templates/README.md` for a full annotation of each setting and guidance on adapting to other frameworks.

---

## The Eight Disciplines

`RAILS.md` establishes eight operating disciplines. Every rule traces to a documented failure:

1. **Session hygiene** — new task, new session; context costs compound
2. **Budget awareness** — pre-paid credits don't reset; retries are self-compounding
3. **Scope discipline** — do what was asked; flag everything else
4. **Completion discipline** — enumerate what you did NOT verify, not just what passed
5. **Human checkpoints** — high-risk actions require human approval before execution
6. **Dependency awareness** — when you change X, flag what depended on X
7. **When to use less** — brevity is not laziness; narrating is not doing
8. **When to use more** — depth is justified for novel failures and irreversible actions

---

## The Case Studies

Two real failures motivated this project. Both are documented in full in `CASE-STUDIES.md`:

- **Case Study 1:** An agent assigned a sysadmin task with no scope enforcement, no retry cap, and no session hygiene. It self-initiated an update outside its task, hit the rate limit, retried in a compounding loop, and exhausted the API credits mid-session.

- **Case Study 2:** An agent that completed every visible step of a Secure Boot setup correctly — all checks green — and advised the user to proceed. The system panicked on reboot. The agent had no mechanism to surface what it didn't know, only to confirm what it had checked.

Both failures were avoidable. The rails that would have prevented them are in `RAILS.md`.

---

## Design Principles

**Framework-agnostic.** `RAILS.md` works with any agent. Config templates are framework-specific and clearly labelled.

**Evidence-based.** Every rule traces to a documented failure. No speculative rules.

**Calibrated.** The guide tells agents when to spend more, not just when to spend less. An agent that never acts isn't useful.

**Living document.** RAILS.md grows as new failure modes are found. The contribution model is simple: your agent breaks in a new way, you document the failure, you submit the rail.

---

## Contributing

If your agent breaks in a new way, that failure belongs here. Document the case in the format in `CONTRIBUTING.md` and submit a pull request.

We're specifically interested in:
- Failures involving agents with financial or data access (procurement, billing, inventory)
- Failures from frameworks other than OpenClaw
- Failures at the boundary of human + agent handoffs
- Cases where following an existing rail made things worse

---

## Project Status

RAILED is in active use, deployed on a hardened Arch Linux machine as the operating discipline for a local Clawbot instance.

Known gaps (tracked in the case studies):
- Per-task tool allowlists
- Real-time credit balance monitoring
- Automated dependency graph for multi-step tasks
- Canary validation before high-risk reboots

These gaps are open. Contributions that address them are welcome.

---

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — share alike. Build on this, keep it open.

---

*Built from failure. Deployed on real machines. Growing as new things break.*
