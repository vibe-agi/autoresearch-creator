---
name: create-autoresearch
description: Use when user wants to create an autonomous optimization loop for their project. Analyzes codebase with expert agents, defines optimization pillars, generates a self-improving experiment skill.
argument-hint: domain hint, e.g. web-performance
---

# create-autoresearch

Generate a project-specific autonomous optimization skill for this codebase.

Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch): any optimization problem can be expressed as iterative hill-climbing over a mutable code surface, evaluated against a frozen metric, with a ratchet mechanism that only advances on improvement.

## Core Formula

```
Optimization = State(code) x Metric(scalar) x Mutation(change) x Evaluation(run) x Selection(keep/discard)
```

## How This Works

You will execute 4 phases to create a custom autoresearch skill for this project:

1. **Phase 1: Project Analysis** — Dispatch 4 expert agents in parallel to understand the codebase
2. **Phase 2: Pillar Dialogue** — Walk the user through confirming 5 optimization pillars
3. **Phase 3: Skill Generation** — Generate the project-specific skill files
4. **Phase 4: User Approval** — Present the generated skill for review

The output is a fully functional skill at `.claude/skills/autoresearch-<domain>/` that the user can invoke to run autonomous optimization loops.

## Phase 1: Project Analysis

Read `expert-analysis.md` for the full expert prompts.

Dispatch 4 agents in parallel using the Agent tool (subagent_type: Explore for the first three, general-purpose for Domain Analyst):

| Expert | Focus |
|--------|-------|
| **Architecture Analyst** | Project structure, languages, frameworks, modules, dependencies |
| **Metric Scout** | Existing tests, benchmarks, metrics, CI evaluation, monitoring |
| **Runtime Analyst** | Build/test/run commands, environment, how to launch and evaluate |
| **Domain Analyst** | README, docs, comments — business goals and user value |

Wait for all 4 to complete. Synthesize their outputs into a unified project understanding.

If `$ARGUMENTS` contains a domain hint (e.g., "web-performance"), pass it to all experts to focus their analysis.

## Phase 2: Pillar Dialogue

Read `pillar-dialogue.md` for the full dialogue protocol.

Walk the user through confirming each pillar **one at a time**. Do NOT batch questions.

**Order matters — each pillar builds on the previous:**

1. **Domain Understanding** — Confirm what the project does and its core goal
2. **Metric Selection** — Present discovered metrics, user picks primary metric + direction
3. **Guard Metrics** — Experts simulate "what could go wrong without this guard", user selects
4. **Surface Definition** — Propose mutable file scope, user adjusts
5. **Harness Confirmation** — Confirm evaluation command and extraction logic
6. **Budget Setting** — Propose resource cap per experiment
7. **Tier Classification** — Classify feedback cycle speed, confirm loop mode

After all pillars are confirmed, summarize the complete configuration and get final approval before proceeding to generation.

## Phase 3: Skill Generation

Read `skill-generation.md` for the full templates.

Generate files using the confirmed pillars. All template variables (marked with `<angle-brackets>`) must be filled with concrete values from the pillar dialogue.

**Generate in `.claude/skills/autoresearch-<domain>/`:**

1. `SKILL.md` — Loop entry point that loads program.md and executes the experiment loop
2. `program.md` — The complete experiment protocol with Frozen Rules and Mutable Rules
3. `meta-review.md` — Self-optimization protocol referenced by SKILL.md
4. `knowledge-seed.md` — Domain-appropriate initial knowledge (copy from `knowledge-seed-generic.md` and customize for the specific domain, adding domain-specific anti-patterns, proven patterns, and heuristics)

**Generate in project root `autoresearch/`:**

5. `results.tsv` — Empty TSV with header row
6. `knowledge.md` — Initialized from the knowledge seed

After generating, create a git branch `autoresearch/<domain>` for the optimization work.

## Phase 4: User Approval

Present each generated file to the user with a summary of what it does:

```
Generated autoresearch skill for: <domain>

Files created:
  .claude/skills/autoresearch-<domain>/SKILL.md      — Experiment loop (invoke with /autoresearch-<domain>)
  .claude/skills/autoresearch-<domain>/program.md     — Experiment protocol
  .claude/skills/autoresearch-<domain>/meta-review.md — Self-optimization rules
  .claude/skills/autoresearch-<domain>/knowledge-seed.md — Initial domain knowledge
  autoresearch/results.tsv                            — Experiment log
  autoresearch/knowledge.md                           — Knowledge base

Configuration:
  Primary Metric: <name> (<direction>)
  Guard Metrics: <list>
  Surface: <file patterns>
  Budget: <resource cap>
  Tier: <1|2|3> (<loop mode>)

Review the generated files. If everything looks good, I'll commit them.
Then invoke /autoresearch-<domain> to start the optimization loop.
```

If the user requests changes, modify the relevant files and re-present.

Once approved, commit all files:
```bash
git add .claude/skills/autoresearch-<domain>/ autoresearch/
git commit -m "feat: generate autoresearch skill for <domain>"
```

## Important Rules

- ALWAYS communicate in the user's language. Detect the language from the user's messages and use it throughout all phases. Technical terms (metric names, commands, file paths) stay in English, but all explanations, questions, and dialogue must match the user's language.
- NEVER skip Phase 2 (pillar dialogue). Even if the domain seems obvious, the user must confirm each pillar.
- NEVER generate a skill with placeholder values. Every `<variable>` must be replaced with a concrete value.
- The Metric and Evaluation Command in Frozen Rules are the user's decision. Suggest, but do not override.
- If the user's project has no obvious quantifiable metric, help them define one. Every domain has something measurable.
