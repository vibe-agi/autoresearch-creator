# autoresearch-creator

A Claude Code plugin that generates project-specific autonomous optimization skills.

## What This Is

A factory skill (`/create-autoresearch`) that:
1. Analyzes a user's codebase with 4 expert agents
2. Guides the user through defining an optimization problem (5 pillars)
3. Generates a complete autoresearch skill in the user's project

## Project Structure

```
.claude-plugin/plugin.json              — Plugin manifest
skills/create-autoresearch/
  SKILL.md                              — Factory skill entry point
  expert-analysis.md                    — 4 expert agent prompts
  pillar-dialogue.md                    — User dialogue protocol
  skill-generation.md                   — Templates for generated files
  knowledge-seed-generic.md             — Generic domain knowledge seed
docs/
  specs/                                — Design specifications
  plans/                                — Implementation plans
```

## Development

Test locally: `claude --plugin-dir .`
Reload after changes: `/reload-plugins`

## Key Concepts

- **5 Pillars:** Metric, Surface, Budget, Harness, Protocol
- **3-Layer Judgment:** L1 (metric), L2 (compliance), L3 (tacit knowledge)
- **3 Tiers:** T1 (autonomous/deterministic), T2 (autonomous/noisy), T3 (interactive/slow)
- **Meta-Review:** Self-optimization of mutable protocol rules
- **Knowledge Base:** Anti-patterns + proven patterns + domain heuristics
