# Config Templates

These templates show how RAILED's behavioral principles translate into framework-level configuration. Config rails work alongside RAILS.md — the document governs agent behavior; the config enforces limits the agent cannot override on its own.

---

## openclaw.json

Annotated guide to `openclaw.json` — the configuration for [OpenClaw](https://github.com/0xHoneyJar/openclaw) deployments.

### contextTokens

```json
"contextTokens": 60000
```

**What it does:** Sets the session token budget. Compaction kicks in when the context approaches this limit.

**Why 60,000:** Anthropic's Sonnet supports 200k context, but a 200k session is not a 200k cost — it's that cost per turn, every turn. 60k is the sustainable daily operating budget that prevents runaway accumulation. Adjust based on your actual workload.

**What failure it prevents:** Case Study 1 — the session that grew to 20k tokens/message after 30 turns with no compaction.

---

### compaction.mode

```json
"compaction": {
  "mode": "safeguard"
}
```

**What it does:** Sets the compaction strategy. `safeguard` mode preserves recent context more aggressively than default, prioritising recency over compression ratio.

**Why safeguard:** The agent should not lose track of what it just did. Aggressive compression that drops the last few turns causes the agent to repeat work or lose thread on multi-step tasks.

---

### compaction.notifyUser

```json
"notifyUser": true
```

**What it does:** Notifies the user when compaction runs.

**Why enable it:** Compaction is a natural session boundary signal. When the agent tells the user "compaction just ran," the user can decide whether to continue in the same session or start fresh for the next task. Without notification, compaction is invisible and the user has no signal to act on.

---

### compaction.recentTurnsPreserve

```json
"recentTurnsPreserve": 5
```

**What it does:** Keeps the last N turns verbatim during compaction — they are not summarised.

**Why 5:** The most recent decisions and tool outputs are the most load-bearing. Summarising them risks losing exact outputs (file paths, command results, error messages) that the agent needs for the next action. 5 turns is enough to preserve working context without consuming too much of the post-compaction budget.

---

### compaction.maxHistoryShare

```json
"maxHistoryShare": 0.5
```

**What it does:** Caps the fraction of the context budget that retained history can consume after compaction.

**Why 0.5:** Ensures at least 50% of the context budget is available for new work — tool outputs, generation, reasoning. Without this cap, a long session can compact and still leave the agent with almost no headroom for the next task.

---

### compaction.memoryFlush.enabled

```json
"memoryFlush": {
  "enabled": true
}
```

**What it does:** Before heavy compaction runs, writes salient context (key decisions, open actions, blocking unknowns) to the agent's memory files.

**Why enable it:** Compaction summarises; memory persists. Decisions that matter across sessions should survive compaction. Enabling the flush ensures the agent's working memory is updated before context is compressed.

---

### subagents.maxConcurrent

```json
"subagents": {
  "maxConcurrent": 2
}
```

**What it does:** Limits the number of subagents running in parallel.

**Why 2:** Parallel subagents multiply token spend. Two subagents running simultaneously can double the per-minute token rate. Keep this low unless parallel work is genuinely required.

---

### subagents.maxSpawnDepth

```json
"maxSpawnDepth": 1
```

**What it does:** Prevents subagents from spawning their own subagents. Hierarchy is flat.

**Why flat:** Subagent trees are hard to reason about and can produce explosive token usage. A subagent that spawns three subagents, each of which spawns two more, creates 7 concurrent agents before any human has noticed. Flat hierarchy keeps the operator in control.

---

### subagents.maxChildrenPerAgent

```json
"maxChildrenPerAgent": 3
```

**What it does:** Caps the total number of subagents a session can spawn.

**Why 3:** Combined with `maxSpawnDepth: 1`, this means a maximum of 3 subagents ever active in a session (sequentially or in batches of 2). Enough for genuinely parallel work; not enough for runaway proliferation.

---

## Adapting to Other Frameworks

The principles these settings implement are framework-agnostic. If you're using a different agent framework, look for equivalent controls:

| Principle | What to configure |
|-----------|------------------|
| Session budget | Context window limit or token budget per session |
| Compaction/summarisation | Automatic context compression with notification |
| Retry limits | Max automatic retries per tool call (target: 2) |
| Subagent limits | Max concurrent workers, max spawn depth |
| Scope enforcement | Tool allowlists per task (where supported) |

Where your framework doesn't support a control at the config level, rely on RAILS.md to instruct the agent to enforce it behaviorally. Both layers matter.

---

## Contributing a Template

If you've adapted these settings for a different framework (Claude Code, LangChain, AutoGen, custom), contribute it:

1. Add a `<framework-name>.json` (or equivalent config format) to this directory
2. Add a matching section to this README with annotations
3. Note which failure modes from `CASE-STUDIES.md` each setting addresses
4. Submit a pull request

Templates should be minimal — only settings that directly implement a RAILED principle. Don't add settings for unrelated framework tuning.
