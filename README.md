# RAILED

> Operating discipline for AI agents with real tool access — every rule traced to a documented failure.

[![License: CC BY-SA 4.0](https://img.shields.io/badge/license-CC%20BY--SA%204.0-lightgrey.svg)](LICENSE)

## The idea

Most agents are deployed without behavioural constraints. They don't know when to stop, when to
ask, what's in scope, or how to tell whether a task is actually done. They optimise for completing
the spec they were given — confidently, efficiently, and sometimes incorrectly.

RAILED is a single markdown file, [`RAILS.md`](RAILS.md), that any agent loads as context at
session start. It covers four things:

- **Efficiency** — calibrate effort to task value; stop narrating, stop carrying stale context.
- **Cost** — both compute (tokens per task) and money, for agents with real spending power.
- **Security** — stay in scope; stop for a human before reboots, auth changes, destructive writes
  and external communication.
- **Privacy** — know what not to surface or log, and when to stop and ask.

It's for anyone running agents where mistakes cost something: newcomers who haven't yet seen a
runaway bot, homelabbers giving agents root, and teams giving agents access to finance, inventory
or customer data.

## What's inside

| Path | What it is |
|------|------------|
| `RAILS.md` | The protocol — nine disciplines, ready to load as agent context |
| `CASE-STUDIES.md` | Eight documented failures the rules come from |
| `config-templates/` | Framework-level limits (OpenClaw) that enforce what the rules ask for, annotated |
| `CONTRIBUTING.md` | How to add a failure case or a rail |

## Quick start

```sh
# Claude Code project
cp RAILS.md /path/to/project/ && echo "@RAILS.md" >> /path/to/project/CLAUDE.md

# OpenClaw
cp RAILS.md ~/.openclaw/workspace/RAILS.md

# Anything else: prepend RAILS.md to the system prompt
```

For OpenClaw, also review and adapt [`config-templates/openclaw.json`](config-templates/openclaw.json)
(session budget, compaction, subagent limits) — see [`config-templates/README.md`](config-templates/README.md).

## How it works

`RAILS.md` sets nine disciplines:

1. **Session hygiene** — new task, new session; announce state on cold start; context costs compound
2. **Budget awareness** — credits don't reset; retries compound; calibrate to your API tier
3. **Scope discipline** — do what was asked; flag everything else
4. **Completion discipline** — list what you did *not* verify; verify artefacts, not exit codes
5. **Human checkpoints** — high-risk actions need approval before they run
6. **Dependency awareness** — when you change X, flag what depended on X
7. **When to use less** — brevity is not laziness; narrating is not doing
8. **When to use more** — depth is justified for novel failures and irreversible actions
9. **Deployment pre-flight** — size bootstrap context, model and token budget to the API tier before first run

Each rule traces to a case in [`CASE-STUDIES.md`](CASE-STUDIES.md):

| Case | Failure |
|------|---------|
| 1 | Retry spiral + out-of-scope self-update exhausted API credits mid-session |
| 2 | Every check green, system panicked on boot — the agent never listed what it hadn't verified |
| 3–4 | A browser-substitution loop with no exit ran silently for hours into a context overflow |
| 5 | Tools authenticated in the orchestrator didn't extend to the embedded agent |
| 6 | After a reboot the agent silently forgot its pending tasks |
| 7 | A discipline file too large for a Tier 1 key caused the rate-limit failure it was meant to prevent |
| 8 | A permanent sandbox constraint was retried four times instead of recognised |

## Requirements

- Any agent that can load a markdown file as context. Config templates target OpenClaw.

## Security notes

- Case studies are anonymised: operator, machine and company details are removed; the failure
  modes are kept intact.
- Config templates change framework limits — review them before applying; don't copy blindly.

## Status

v2.0 — complete and shared as a reference; not actively maintained. Known open gaps (per-task tool
allowlists, live credit monitoring, automated dependency graphs, canary checks before risky
reboots) are listed in the case studies. Forks that extend it are welcome — see
[`CONTRIBUTING.md`](CONTRIBUTING.md) for the case format.

## Part of a set

| Project | What it is |
|---------|------------|
| **railed** | Operating discipline for AI agents with real tool access, built from documented failures |
| [hardened-api](https://github.com/lobsterscollide/hardened-api) | Linux hardening checks and fixes as an API an agent can drive |
| [usb-triage](https://github.com/lobsterscollide/usb-triage) | Run-from-USB security audit for Linux, macOS and Windows |
| [workstation-scripts](https://github.com/lobsterscollide/workstation-scripts) | Daily → quarterly security routine for an Arch workstation |
| [a2a-schemas](https://github.com/lobsterscollide/a2a-schemas) | Permission boundaries and handoffs between a "build" agent and an "apply" agent |

## License

[CC BY-SA 4.0](LICENSE) — build on it, keep it open.
