# autoresearch-creator

A Claude Code plugin that generates project-specific autonomous optimization skills.

## What This Is

A factory skill (`/create-autoresearch`) that:
1. Analyzes a user's codebase with 4 expert agents (Architecture, Metric Scout, Runtime, Domain)
2. Guides the user through 9-step pillar dialogue
3. Writes `autoresearch/pillars.json` as the single source of truth
4. Renders generated skill files from pillars.json + templates
5. Supports upgrade: `/create-autoresearch upgrade` re-renders existing skills with new templates

## Project Structure

```
.claude-plugin/plugin.json              — Plugin manifest
skills/create-autoresearch/
  SKILL.md                              — Factory entry point (4-phase flow + upgrade flow)
  expert-analysis.md                    — 4 expert agent prompts + conflict resolution rules
  pillar-dialogue.md                    — 9-step user dialogue protocol
  skill-generation.md                   — Templates (pillars.json schema + all generated files)
  knowledge-seed-generic.md             — Flat domain-agnostic knowledge seed
docs/
  specs/                                — Design specifications
  plans/                                — Implementation plans
```

## Development

Test locally: `claude --plugin-dir .`
Reload after changes: `/reload-plugins`

## Core Architecture (v0.5.0+)

### Single Source of Truth: pillars.json

Every generated skill is defined by `autoresearch/pillars.json`. All `.md` files
(SKILL.md, program.md, meta-review.md, knowledge-seed.md) are rendered from
this JSON. Users and agents MUST NOT edit generated .md files directly — all
changes flow through pillars.json.

Meta-review modifies the `mutable` section of pillars.json and triggers a
re-render. `frozen` fields can only be changed by re-running `/create-autoresearch`
from scratch.

### Upgrade Mechanism

When `autoresearch-creator` itself upgrades (via `/plugin update`), generated
skills can be refreshed with the new templates by running:

```
/create-autoresearch upgrade
```

This reads `autoresearch/pillars.json`, loads the current templates, re-renders
all .md files, and shows a diff for user approval. `results.tsv` and
`knowledge.md` are preserved (user-accumulated assets).

Legacy skills (pre-v0.5.0) without pillars.json require manual migration. See
the factory SKILL.md upgrade flow for instructions.

## Key Concepts

### 5 Pillars
- **Metric** — single scalar with direction (+ optional guards)
- **Surface** — files the agent can modify
- **Budget** — resource cap per experiment (flexible unit: seconds, iterations, tokens, $, etc.)
- **Harness** — frozen evaluation command + extraction
- **Protocol** — keep/discard rules, mutation strategy, tier

### 3-Layer Judgment
- **L1 (metric check)** — did the scalar improve? guards satisfied?
- **L2 (compliance)** — tests pass, lint clean, Protected Patterns grep-verified
- **L3 (secondary review)** — LLM reviews diff for defensive code removal + reasonableness

### 2 Tiers (v0.5.0)
- **T1** — autonomous, deterministic (ML training, compiler, DB)
- **T2** — autonomous with noise handling (Web perf, API latency, game FPS)
- **T3 (removed)** — interactive loops are out of scope for v0.5.0

### Pre-Experiment Sanity Check
Every experiment starts with 4 cheap checks:
1. File existence (Surface + Frozen files)
2. Protected Pattern integrity (grep verification)
3. Evaluation command smoke test
4. Metric extraction health

Any failure pauses the loop and asks the user to resolve.

### Git Safety
- Factory never auto-commits to a dirty worktree
- Generated skill refuses to start with uncommitted changes
- Only explicit file paths are staged — never `git add .`
- User's uncommitted code is never auto-stashed or auto-committed

### Meta-Review
Signal-driven self-optimization. Analyzes results.tsv + knowledge.md, proposes
changes to `pillars.mutable.*` fields (never `frozen.*`), user approves each,
then re-renders .md files.

### Knowledge Base
Flat list of anti-patterns and proven patterns. No candidate/formal/deprecated
lifecycle. Entries accumulate during experiments; meta-review consolidates them.

### External Knowledge Fallback
When meta-review has no direction and knowledge.md has no relevant pattern,
the skill uses WebSearch (max 3 per fallback session) to find new techniques.
Findings become candidate experiments subject to normal 3-layer judgment.

## Versioning

- `autoresearch-creator` version is in `.claude-plugin/plugin.json`
- Also tracked in `vibe-marketplace` at `.claude-plugin/marketplace.json`
- Generated skills embed their generator version in `pillars.generator_version`
- Generated SKILL.md does a version self-check at each invocation and prompts
  the user to run `/create-autoresearch upgrade` when out of date
