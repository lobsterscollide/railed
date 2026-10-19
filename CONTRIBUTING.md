# Contributing to RAILED

RAILED grows from real failures. Every rule in `RAILS.md` traces to a documented case. The contribution model is simple: if your agent breaks in a new way, document the failure and the rail that would have prevented it.

---

## What Makes a Valid Contribution

A valid contribution is grounded in an observed failure — something that actually happened, with a real agent, on a real task. Speculative rules and preventive measures for things-that-haven't-happened-yet don't belong here yet.

Valid:
- A new failure mode not covered by the existing case studies
- A refinement to an existing rule based on operational experience
- A config template for a framework not yet covered
- A correction where a rule turned out to be wrong or counterproductive

Not valid (yet):
- Rules based on hypothetical risks
- Rules that duplicate existing disciplines without adding specificity
- Config templates that tune framework settings unrelated to RAILED principles

---

## Submitting a New Case Study

Open a pull request adding your case to `CASE-STUDIES.md`. Use this format:

```markdown
## Case Study N: [Short Title]

**Date:** YYYY-MM-DD
**Agent:** [Framework + model]
**Task:** [What the agent was doing]

### What Happened
[Narrative — what the agent did, what went wrong, what the impact was]

### Root Causes
[What actually caused the failure — not the symptom, the mechanism]

### Rails That Would Have Prevented It
[Specific, concrete rules that would have changed the outcome]

### What Was Actually Implemented
[If you've deployed a fix, describe it. If not, leave this section as "Not yet implemented."]
```

Keep it factual. Avoid generalising beyond what the case actually shows. One clear case is worth more than ten inferred principles.

---

## Submitting a New Rail

If your case study motivates a new rule for `RAILS.md`, include a proposed addition in your pull request. Rail proposals should:

- State the rule clearly — as a directive an agent can act on
- Cite the case study it comes from
- Fit into one of the existing disciplines, or propose a new discipline with justification
- Be calibrated — state when the rule applies, not just what the rule is

If the proposed rail would change or constrain an existing rule, explain why the existing rule was insufficient.

---

## Submitting a Config Template

Add your template to `config-templates/` and document it in `config-templates/README.md`. For each setting, note which RAILED principle it implements and which failure mode it addresses.

---

## What Happens After Submission

Pull requests are reviewed for:

1. **Grounding** — does this trace to a real failure?
2. **Specificity** — is the rule actionable, or is it vague advice?
3. **Calibration** — does it restrict appropriately, or does it over-restrict?
4. **Non-duplication** — does it add something the existing rules don't cover?

We'll engage with the substance. We may suggest edits, ask for more detail on the failure, or fold a proposed rail into an existing discipline.

---

## Versioning

`RAILS.md` uses a simple version number (`v1.0`, `v1.1`, etc.) incremented when the content changes. Config templates are unversioned — they reflect current best practice and evolve with the frameworks.

---

## A Note on Tone

This project is written for agents, not against them. The goal is not to restrict what agents can do — it's to give them the context and constraints they need to operate well. Rules should be explainable, not just enforceable. If a rule can't be justified to an agent, it probably can't be justified at all.
