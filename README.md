# autoresearch-creator

A Claude Code plugin that generates project-specific autonomous optimization skills, inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch).

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **[中文文档](./README_CN.md)**

---

## What This Is

[karpathy/autoresearch](https://github.com/karpathy/autoresearch) showed that an autonomous loop of experiment, measure, decide can steadily optimize ML models without human intervention. **autoresearch-creator** generalizes that philosophy into any domain -- CPU performance, test coverage, API latency, conversion rates, and beyond.

It is a **factory skill** (`/create-autoresearch`) that analyzes your codebase and generates a custom autoresearch skill tailored to YOUR optimization problem. You describe what you want to improve; it builds the entire experiment loop for you.

## Core Philosophy

Every optimization problem can be decomposed into the same five primitives:

```
Any optimization problem = State(code) x Metric(scalar) x Mutation(change) x Evaluation(run) x Selection(keep/discard)
```

ML loss, page load time, query latency, mutation test score -- they are all scalars attached to a codebase. The only things that change between domains are _what_ you measure and _how_ you mutate. autoresearch-creator captures those specifics through structured dialogue, then generates a skill that runs the loop autonomously.

## How It Works

The `/create-autoresearch` skill operates in four phases:

### Phase 1 -- Expert Analysis

Four specialized agents analyze your codebase in parallel:

| Agent | Focus |
|---|---|
| Architecture | Code structure, entry points, module boundaries |
| Metrics | Existing measurements, benchmarks, observable outputs |
| Runtime | Build system, test harness, execution environment |
| Domain | Business context, optimization opportunities, constraints |

### Phase 2 -- Pillar Dialogue

An interactive conversation defines the five pillars of your optimization problem:

1. **Metric** -- The scalar to optimize (and guard metrics to prevent gaming)
2. **Surface** -- Which files and code regions are mutable
3. **Budget** -- Time, cost, and iteration limits per experiment
4. **Harness** -- The command(s) that evaluate a mutation and produce the metric
5. **Protocol** -- Rules, constraints, and domain-specific heuristics

Guard metrics are validated through a lightweight simulation before the skill is finalized.

### Phase 3 -- Skill Generation

Generates a complete, project-specific skill containing:

- An experiment loop with automated branching, mutation, evaluation, and rollback
- 3-layer judgment system for deciding whether to keep or discard each change
- Meta-review protocol for evolving experiment strategy over time
- A knowledge base seeded with domain-relevant patterns and anti-patterns

### Phase 4 -- User Approval

The generated skill is presented for review. Nothing is written to your project until you approve.

## Key Features

**3-Layer Judgment**

- **L1 -- Metric Gate:** Did the scalar improve beyond the noise threshold?
- **L2 -- Compliance Check:** Does the change satisfy explicit rules (style, safety, API contracts)?
- **L3 -- Expert Review:** Does the change respect tacit knowledge -- the kind of understanding practitioners have but struggle to articulate? (cf. Michael Polanyi, _The Tacit Dimension_)

**3-Tier Support**

| Tier | Feedback | Autonomy | Example |
|---|---|---|---|
| T1 | Deterministic, fast | Fully autonomous | Compiler benchmarks, unit tests |
| T2 | Noisy or slow | Autonomous with statistical checks | Lighthouse scores, load tests |
| T3 | Requires human judgment | Interactive, human-in-the-loop | Conversion rates, UX metrics |

**Meta-Review** -- After a configurable number of experiments, the skill reviews its own protocol and proposes adjustments. Changes to mutable rules require user approval.

**Knowledge Base** -- Maintains three categories of learned knowledge that grow through experimentation:

- Anti-patterns (mutations that reliably fail)
- Proven patterns (mutations that reliably succeed)
- Domain heuristics (context-specific rules of thumb)

**Guard Metrics** -- Secondary metrics monitored alongside the primary objective to prevent metric gaming (e.g., optimizing latency at the cost of correctness).

## Applicable Domains

| Domain | Example Metric | Tier |
|---|---|---|
| ML Training | val_loss | T1 |
| Compiler Optimization | benchmark time | T1 |
| DB Query Performance | avg query time | T1 |
| Web Performance | Lighthouse score | T2 |
| API Latency | p95 latency | T2 |
| Test Coverage | mutation score | T2 |
| Game Rendering | FPS | T2 |
| Bundle Size | gzipped KB | T2 |
| Conversion Rate | CTR | T3 |
| UX Satisfaction | task completion rate | T3 |

Any problem with a measurable scalar and a mutable codebase is a candidate.

## Installation

```bash
# Via Claude Code marketplace
/plugin install autoresearch-creator

# Or from source
claude --plugin-dir /path/to/autoresearch
```

## Quick Start

```bash
# In your project directory, run the factory skill
/create-autoresearch

# Or provide a domain hint to skip some discovery
/create-autoresearch web-performance
```

The skill will walk you through expert analysis, pillar definition, and approval before generating anything.

## Generated Output

After approval, the following structure is created in your project:

```
.claude/skills/autoresearch-<domain>/
  SKILL.md              # Skill entry point with experiment loop
  program.md            # Optimization program and protocol rules
  meta-review.md        # Meta-review schedule and criteria
  knowledge-seed.md     # Initial domain knowledge base

autoresearch/
  results.tsv           # Experiment log (appended each run)
  knowledge.md          # Accumulated patterns and anti-patterns
```

Run the generated skill with:

```bash
/autoresearch-<domain>
```

## Acknowledgments

This project is inspired by and builds on the ideas from [karpathy/autoresearch](https://github.com/karpathy/autoresearch), which demonstrated that autonomous experiment loops can systematically optimize ML models. autoresearch-creator generalizes that engineering philosophy -- the tight loop of hypothesize, mutate, measure, decide -- so it can be applied to any domain with a measurable objective.

## License

MIT

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=vibe-agi/autoresearch-creator&type=Date)](https://star-history.com/#vibe-agi/autoresearch-creator&Date)
