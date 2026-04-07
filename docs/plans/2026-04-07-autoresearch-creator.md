# autoresearch-creator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin containing the `create-autoresearch` factory skill that generates project-specific autonomous optimization skills.

**Architecture:** A marketplace plugin with one skill (`create-autoresearch`) that dispatches 4 expert agents to analyze a codebase, guides the user through defining 5 optimization pillars, then generates a complete project-level skill with experiment loop, 3-layer judgment, meta-review, and knowledge base.

**Tech Stack:** Claude Code plugin system, SKILL.md format (YAML frontmatter + markdown), Agent tool for expert dispatch, Git for state management.

**Spec:** `docs/specs/2026-04-07-autoresearch-creator-design.md`

---

## File Structure

```
.claude-plugin/
  plugin.json                          # Plugin manifest for marketplace distribution

skills/
  create-autoresearch/
    SKILL.md                           # Factory skill entry point — orchestrates all 4 phases
    expert-analysis.md                 # Reference: 4 expert agent prompts + dispatch instructions
    pillar-dialogue.md                 # Reference: user dialogue protocol for each pillar
    skill-generation.md                # Reference: templates for all generated files
    knowledge-seed-generic.md          # Reference: generic domain knowledge seed

CLAUDE.md                              # Project instructions for contributors
```

---

### Task 1: Plugin Scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Create plugin manifest**

```json
{
  "name": "autoresearch-creator",
  "description": "Generate project-specific autonomous optimization skills inspired by karpathy/autoresearch. Analyzes your codebase, defines optimization pillars, and creates a self-improving experiment loop.",
  "version": "0.1.0",
  "author": {
    "name": "vibe-agi"
  },
  "homepage": "https://github.com/vibe-agi/autoresearch",
  "repository": "https://github.com/vibe-agi/autoresearch",
  "license": "MIT",
  "keywords": [
    "autoresearch",
    "optimization",
    "experiment-loop",
    "self-improving",
    "skill-generator"
  ]
}
```

- [ ] **Step 2: Verify plugin loads**

Run: `cd /Users/null/Code/github/vibe-agi/autoresearch && claude --plugin-dir . --print-plugins 2>&1 | head -20`

Expected: Plugin `autoresearch-creator` appears in the list (may show 0 skills until SKILL.md is created).

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add plugin manifest for autoresearch-creator"
```

---

### Task 2: Factory Skill Entry Point (SKILL.md)

**Files:**
- Create: `skills/create-autoresearch/SKILL.md`

This is the main orchestration file. It defines the 4-phase flow and references detail files for each phase. Must stay under 500 lines.

- [ ] **Step 1: Write SKILL.md**

```markdown
---
name: create-autoresearch
description: Use when user wants to set up an autonomous optimization loop for their project. Analyzes codebase with expert agents, guides through defining optimization pillars (metric, surface, budget, harness, protocol), then generates a project-specific autoresearch skill with experiment loop, 3-layer judgment, meta-review, and knowledge base.
argument-hint: [optional: domain hint, e.g. "web-performance"]
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

- NEVER skip Phase 2 (pillar dialogue). Even if the domain seems obvious, the user must confirm each pillar.
- NEVER generate a skill with placeholder values. Every `<variable>` must be replaced with a concrete value.
- The Metric and Evaluation Command in Frozen Rules are the user's decision. Suggest, but do not override.
- If the user's project has no obvious quantifiable metric, help them define one. Every domain has something measurable.
```

- [ ] **Step 2: Verify skill appears in plugin**

Run: `cd /Users/null/Code/github/vibe-agi/autoresearch && claude --plugin-dir . --print-plugins 2>&1 | head -20`

Expected: `create-autoresearch` skill appears in the plugin's skill list.

- [ ] **Step 3: Commit**

```bash
git add skills/create-autoresearch/SKILL.md
git commit -m "feat: add create-autoresearch factory skill entry point"
```

---

### Task 3: Expert Analysis Prompts

**Files:**
- Create: `skills/create-autoresearch/expert-analysis.md`

This reference file contains the exact prompts for each of the 4 expert agents. Loaded on-demand by SKILL.md during Phase 1.

- [ ] **Step 1: Write expert-analysis.md**

```markdown
# Expert Analysis: 4-Agent Codebase Analysis

## Dispatch Instructions

Use the Agent tool to dispatch all 4 experts **in parallel** (a single message with 4 Agent tool calls).

Each expert receives:
- The project root path
- The domain hint from `$ARGUMENTS` (if provided)
- Their specific analysis prompt (below)

All experts should return structured output that can be synthesized.

## Expert 1: Architecture Analyst

**Agent config:** `subagent_type: Explore`, thoroughness: `very thorough`

**Prompt:**

```
Analyze the codebase at <project-root> to produce a technical landscape map.

Your goal: understand the project's architecture so we can determine which files
are safe to modify in an autonomous optimization loop, and which must stay frozen.

Investigate:
1. Languages and frameworks used (check package.json, requirements.txt, Cargo.toml, go.mod, etc.)
2. Directory structure and module organization
3. Core vs peripheral code (what's the hot path vs utility/config?)
4. Dependencies (external libraries, internal module relationships)
5. Build system and compilation pipeline
6. Entry points (main files, API routes, CLI commands)

<if domain hint provided>
Focus especially on code related to: <domain-hint>
</if>

Output format:
## Technical Landscape
- **Languages:** [list]
- **Frameworks:** [list]
- **Build system:** [tool and key commands]
- **Directory map:**
  - `src/` — [purpose]
  - `tests/` — [purpose]
  - ...
- **Hot path files:** [the files most likely to affect performance/quality]
- **Frozen candidates:** [files that should NOT be modified — config, data, eval]
- **Mutable candidates:** [files safe to modify for optimization]
```

## Expert 2: Metric Scout

**Agent config:** `subagent_type: Explore`, thoroughness: `very thorough`

**Prompt:**

```
Analyze the codebase at <project-root> to discover all quantifiable metrics.

Your goal: find everything that is currently measured, benchmarked, or testable,
so the user can choose a primary optimization metric.

Investigate:
1. Test suites — what testing frameworks, what coverage tools, what assertions
2. Benchmarks — any performance benchmarks, load tests, profiling scripts
3. CI/CD pipelines — what gets measured in CI (check .github/workflows/, .gitlab-ci.yml, etc.)
4. Monitoring — any metrics collection, logging of performance data
5. Scripts — any scripts that measure, evaluate, or score something
6. Configuration — any thresholds, SLAs, or targets defined in config

<if domain hint provided>
Focus especially on metrics related to: <domain-hint>
</if>

Output format:
## Metrics Inventory
For each metric found:
- **Name:** [metric name]
- **Current value:** [if discoverable]
- **How measured:** [command or tool]
- **Where defined:** [file path]
- **Direction:** [higher-is-better / lower-is-better]
- **Noise level:** [deterministic / low noise / high noise]

## Metric Recommendations
- **Best primary metric candidate:** [name] — [why]
- **Guard metric candidates:** [list with rationale]
- **Tier assessment:** [1/2/3 based on evaluation speed and noise]
```

## Expert 3: Runtime Analyst

**Agent config:** `subagent_type: Explore`, thoroughness: `very thorough`

**Prompt:**

```
Analyze the codebase at <project-root> to determine how to build, run, test,
and evaluate this project.

Your goal: produce an executable command inventory so the autoresearch loop
knows exactly what commands to run for each experiment.

Investigate:
1. Build commands — how to compile/build (npm run build, make, cargo build, etc.)
2. Test commands — how to run tests (pytest, jest, go test, etc.)
3. Run commands — how to start the application
4. Environment requirements — runtime dependencies, env vars, Docker, databases
5. Evaluation commands — how to measure the metric (benchmark scripts, test with coverage, etc.)
6. Time estimates — how long does each command take to run

<if domain hint provided>
Focus especially on commands related to: <domain-hint>
</if>

Output format:
## Command Inventory
- **Build:** `[command]` (~[N]s)
- **Test:** `[command]` (~[N]s)
- **Run:** `[command]`
- **Evaluate:** `[command]` (~[N]s)
- **Metric extraction:** `[command or grep pattern to extract the scalar]`

## Environment Requirements
- **Runtime:** [Node 20, Python 3.11, etc.]
- **Dependencies:** [how to install — npm install, pip install, etc.]
- **External services:** [databases, APIs, Docker containers needed]
- **Env vars:** [required environment variables, without values]

## Budget Recommendation
- **Estimated evaluation time:** [N seconds]
- **Recommended budget per experiment:** [N seconds/minutes]
- **Repetitions needed:** [1 if deterministic, 3-5 if noisy]
```

## Expert 4: Domain Analyst

**Agent config:** `subagent_type: general-purpose`

**Prompt:**

```
Analyze the codebase at <project-root> to understand what this project does,
who it serves, and what its business/technical goals are.

Your goal: provide the domain context needed to create an intelligent optimization
strategy. The autoresearch loop needs to understand WHY certain changes are good
or bad, not just WHETHER the metric improved.

Investigate:
1. README and documentation — project purpose, features, target users
2. Code comments and docstrings — developer intent, design rationale
3. Issue tracker references — recent priorities, known problems
4. Domain-specific patterns — what engineering methodology fits this domain
5. Quality dimensions — what does "better" mean beyond the metric (UX, maintainability, correctness, etc.)

<if domain hint provided>
Focus especially on the domain: <domain-hint>
</if>

Output format:
## Domain Context
- **Project purpose:** [1-2 sentences]
- **Target users:** [who]
- **Core value proposition:** [what problem it solves]

## Optimization Landscape
- **What "better" means in this domain:** [beyond just the metric]
- **Common optimization directions:** [what experienced engineers would try]
- **Known pitfalls:** [what NOT to do — domain-specific anti-patterns]
- **Quality dimensions to protect:** [things the metric might not capture]

## Engineering Methodology
- **Recommended approach:** [how to iterate in this domain]
- **Single-variable principle:** [how to isolate changes in this domain]
- **Domain heuristics:** [rules of thumb from this field]

## Tier Assessment
- **Can the metric be evaluated locally?** [yes/no — if no, Tier 3]
- **Is the metric deterministic?** [yes/no — if no, Tier 2]
- **Feedback cycle:** [seconds / minutes / hours / days]
```

## Synthesis

After all 4 experts return, synthesize their outputs:

1. Merge the technical landscape + command inventory + metrics inventory + domain context
2. Identify consensus on tier classification (majority wins, present to user)
3. Identify conflicts between experts (e.g., architect says file X is frozen, metric scout says it contains the metric) — resolve or flag for user
4. Prepare the pillar dialogue with expert-backed defaults for each question
```

- [ ] **Step 2: Commit**

```bash
git add skills/create-autoresearch/expert-analysis.md
git commit -m "feat: add 4 expert agent analysis prompts"
```

---

### Task 4: Pillar Dialogue Protocol

**Files:**
- Create: `skills/create-autoresearch/pillar-dialogue.md`

This reference file defines the exact dialogue flow for Phase 2. Loaded on-demand.

- [ ] **Step 1: Write pillar-dialogue.md**

```markdown
# Pillar Dialogue Protocol

## Rules

- **One question at a time.** Never batch pillar questions.
- **Expert-backed defaults.** Every question comes with a recommended answer derived from expert analysis.
- **User always decides.** Suggest, but never override.
- **Build sequentially.** Each pillar builds on the previous confirmed pillar.

## Step 1: Domain Understanding

**Purpose:** Establish shared context. Everything else flows from this.

Present the domain analyst's summary and confirm:

> Based on my analysis, your project is **[project purpose]**, serving **[target users]**.
> The core value proposition is **[value proposition]**.
>
> Is this understanding correct? Anything to add or correct?

Wait for confirmation. If the user corrects something, update the domain context before proceeding.

## Step 2: Metric Selection

**Purpose:** Choose the single primary scalar to optimize.

Present the metric scout's inventory:

> I found these quantifiable metrics in your project:
>
> | # | Metric | Current Value | How Measured | Direction |
> |---|--------|--------------|-------------|-----------|
> | 1 | [name] | [value]      | `[command]` | [higher/lower is better] |
> | 2 | [name] | [value]      | `[command]` | [higher/lower is better] |
> | ... | | | | |
>
> Which metric do you want to optimize? (Pick one as the primary metric)
>
> If none of these are right, describe what you want to optimize and I'll help define a metric.

Record: `primary_metric_name`, `primary_metric_direction`, `primary_metric_command`

## Step 3: Guard Metrics (Simulation-Driven)

**Purpose:** Set hard constraints that prevent gaming the primary metric.

For each candidate guard metric, run a domain expert simulation:

> To protect against gaming, I recommend these guard metrics.
> For each one, here's what could go wrong WITHOUT it:
>
> **Guard 1: [name] >= [floor]**
> Simulation: Without this guard, the agent might [concrete bad scenario].
> Example: "Without a visual regression guard (SSIM >= 0.95), the agent might
> reduce rendering resolution to 10% — FPS goes to 300 but the game is unplayable."
>
> **Guard 2: [name] >= [floor]**
> Simulation: Without this guard, the agent might [concrete bad scenario].
>
> **Guard 3: [name] >= [floor]** (optional)
> Simulation: Without this guard, the agent might [concrete bad scenario].
>
> Which guards do you want to enable? You can adjust the floor values.

Record: `guard_metrics[]` with names, floors, and measurement commands.

If the user selects no guards, warn once: "Running without guards means the agent can make any trade-off to improve [primary metric]. Are you sure?"

## Step 4: Surface Definition

**Purpose:** Define what the agent can and cannot modify.

Present the architecture analyst's recommendations:

> I recommend this as the mutable surface (files the agent can modify):
>
> **Mutable (can change):**
> - `[file/pattern]` — [why it's safe to modify]
> - `[file/pattern]` — [why it's safe to modify]
>
> **Frozen (must NOT change):**
> - `[file/pattern]` — [why: evaluation harness / data / config]
> - `[file/pattern]` — [why: core infrastructure / safety]
>
> Want to add or remove files from either list?

Record: `surface_mutable[]`, `surface_frozen[]`

## Step 5: Harness Confirmation

**Purpose:** Lock down the evaluation command. This becomes frozen and ungameable.

Present the runtime analyst's findings:

> The evaluation harness will run this command for each experiment:
>
> ```bash
> [full evaluation command from runtime analyst]
> ```
>
> Metric extraction: `[grep/jq/parse command to extract the scalar]`
>
> Estimated time per evaluation: ~[N] seconds
> Repetitions: [1 if Tier 1, 3 if Tier 2] (take median)
>
> Is this correct? The evaluation command will be FROZEN — the agent cannot modify it.

Record: `eval_command`, `metric_extraction`, `eval_time_estimate`, `repetitions`

## Step 6: Budget Setting

**Purpose:** Set the resource cap that makes experiments comparable.

> Based on evaluation time (~[N]s x [repetitions] runs), I recommend:
>
> **Budget per experiment: [calculated recommendation]**
>
> This means each experiment gets [budget] of resources. Changes that "cost more"
> automatically get fewer iterations, and the metric reveals whether the trade-off
> was worth it.
>
> Does this budget feel right? Too tight cuts off slow-but-valuable experiments.
> Too loose makes each cycle take too long.

Record: `budget_value`, `budget_unit`

## Step 7: Tier Classification

**Purpose:** Determine the loop mode (autonomous vs interactive).

Present the consensus from experts:

> Based on the analysis:
> - Metric can be evaluated locally: [yes/no]
> - Metric is deterministic: [yes/no]
> - Feedback cycle: ~[time]
>
> I classify this as **Tier [1/2/3]**:
>
> | Tier | Mode | Meaning |
> |------|------|---------|
> | 1 | Fully autonomous | Fast, deterministic — agent runs non-stop |
> | 2 | Autonomous + noise handling | Fast but noisy — agent runs with statistical checks |
> | 3 | Interactive | Slow feedback — agent proposes, you test, you report results |
>
> Does Tier [N] sound right?

Record: `tier`, `loop_mode`

## Final Confirmation

Summarize all confirmed pillars:

> Here's the complete optimization configuration:
>
> **Domain:** [description]
> **Primary Metric:** [name] ([direction])
> **Guard Metrics:** [list with floors]
> **Surface:** [N] mutable files, [M] frozen files
> **Harness:** `[command]` (~[time] per eval, [repetitions]x)
> **Budget:** [value] [unit] per experiment
> **Tier:** [N] — [mode description]
>
> Shall I generate the autoresearch skill with this configuration?

Only proceed to Phase 3 after explicit user approval.
```

- [ ] **Step 2: Commit**

```bash
git add skills/create-autoresearch/pillar-dialogue.md
git commit -m "feat: add pillar dialogue protocol with guard metric simulation"
```

---

### Task 5: Skill Generation Templates

**Files:**
- Create: `skills/create-autoresearch/skill-generation.md`

This is the largest reference file. Contains templates for all 4 generated skill files + 2 project files. Each template uses `<angle-bracket>` variables that get replaced with concrete values from the pillar dialogue.

- [ ] **Step 1: Write skill-generation.md**

```markdown
# Skill Generation Templates

## Overview

This file contains templates for all files generated by `create-autoresearch`.
Replace all `<angle-bracket>` variables with concrete values from the pillar dialogue.

**CRITICAL:** No `<variable>` may remain in the generated output. Every placeholder
must be replaced with a real value. If a value is unknown, go back and ask the user.

---

## Template 1: Generated SKILL.md

Write to: `.claude/skills/autoresearch-<domain>/SKILL.md`

````markdown
---
name: autoresearch-<domain>
description: Run autonomous optimization loop for <domain>. Iteratively improves <primary_metric_name> through experiment cycles with 3-layer judgment, knowledge base, and self-optimization. Invoke to start or resume the loop.
---

# AutoResearch: <domain>

Run the autonomous optimization loop for this project.

## Before Starting

Read `program.md` for the complete experiment protocol.

## Quick Reference

- **Metric:** <primary_metric_name> (<primary_metric_direction>)
- **Eval:** `<eval_command>`
- **Surface:** <surface_mutable_summary>
- **Budget:** <budget_value> <budget_unit> per experiment

## Execution

### Setup (first run only)

1. Check if branch `autoresearch/<domain>` exists. If not, create it from current HEAD.
2. Switch to the branch: `git checkout autoresearch/<domain>`
3. Verify evaluation works: run `<eval_command>` and extract baseline metric.
4. Record baseline in `autoresearch/results.tsv`:
   ```
   <commit>	<baseline_metric>	<guard_values>	baseline	Initial baseline
   ```
5. Confirm with user: "Baseline <primary_metric_name>: <value>. Starting optimization loop."

### The Loop

Read the full protocol from `program.md` and follow it exactly. The loop structure:

<if tier 1 or 2>
```
NEVER STOP. Once the loop begins, do NOT pause to ask the user.
Run experiments continuously until interrupted.

loop:
  1. Read autoresearch/knowledge.md — check anti-patterns, reference proven patterns
  2. Analyze codebase within Surface, propose ONE improvement hypothesis
  3. Modify files (Surface only: <surface_mutable_list>)
  4. Git commit with descriptive message
  5. Run evaluation: <eval_command>
     <if repetitions > 1>Run <repetitions> times, take median<endif>
  6. Extract metrics: <metric_extraction>
  7. Three-Layer Judgment (see program.md):
     L1: <primary_metric_name> improved >= <keep_threshold>?
         All guards met? (<guard_checks>)
     L2: Compliance checks pass? (see program.md L2 section)
     L3: Expert review needed? (see program.md L3 triggers)
  8. KEEP (all pass) or DISCARD (any fail):
     KEEP:  log "keep" to results.tsv
     DISCARD: git reset --hard HEAD~1, log "discard" to results.tsv
  9. Update autoresearch/knowledge.md with candidate entry
  10. Check meta-review triggers (see below)
  11. Back to 1
```
<endif>

<if tier 3>
```
loop:
  1. Read autoresearch/knowledge.md
  2. Analyze codebase, propose improvements
  3. Generate <N> variant proposals
  4. For each variant, evaluate with proxy metric
  5. Present variants to user:
     "Variant A: [description] — proxy score X
      Variant B: [description] — proxy score Y
      ...
      Please deploy/test and report real results."
  6. WAIT for user input
  7. Record proxy + real scores to results.tsv
  8. Keep best real-performing variant, discard others
  9. Update knowledge.md
  10. Check meta-review triggers
  11. Back to 1
```
<endif>

### Meta-Review

Read `meta-review.md` when ANY of these triggers fire:
- 5 consecutive discards
- 3 consecutive crashes
- User interrupts the loop (Ctrl+C)
- Every 20 experiments completed
- Rolling avg improvement < <convergence_threshold> for 10 rounds

Meta-review analyzes results.tsv + knowledge.md, then proposes changes
to program.md Mutable Rules. User must approve each change.

### Post-Experiment Knowledge Update

After EVERY experiment (keep, discard, or crash):

**If discard:**
```
Add candidate anti-pattern to autoresearch/knowledge.md:
### [CANDIDATE] AP-xxx: <brief title>
- Evidence: experiment #<N>
- What was tried: <description>
- Why it failed: <analysis of why metric didn't improve>
```

**If keep:**
```
Add candidate proven pattern to autoresearch/knowledge.md:
### [CANDIDATE] PP-xxx: <brief title>
- Evidence: experiment #<N>
- What was done: <description>
- Why it worked: <analysis of why metric improved>
```

**If crash:**
```
Add error pattern to autoresearch/knowledge.md:
### [CANDIDATE] AP-xxx: <brief title> [CRASH]
- Evidence: experiment #<N>
- What was tried: <description>
- Error: <stack trace summary>
- Fix attempted: <what was tried to fix it>
```
````

---

## Template 2: Generated program.md

Write to: `.claude/skills/autoresearch-<domain>/program.md`

````markdown
# AutoResearch Protocol: <domain>

## Project Context

<domain_analyst_output>

## Frozen Rules (NEVER modify these — changing them invalidates all prior results)

- **Primary Metric:** <primary_metric_name> (<primary_metric_direction>)
- **Evaluation Command:** `<eval_command>`
- **Metric Extraction:** `<metric_extraction>`
- **Guard Metrics:**
  <for each guard>
  - <guard_name> <guard_direction> <guard_floor> — measured by: `<guard_command>`
  </for>
- **Frozen Files (DO NOT TOUCH):**
  <for each frozen file>
  - `<file_path>` — <reason>
  </for>
- **Repetitions:** <repetitions> per experiment (take median)
- **Prohibited Actions:**
  - Do not modify any file outside the Surface
  - Do not modify the evaluation command or metric extraction
  - Do not add external dependencies without L3 expert approval
  - Do not modify test fixtures or evaluation data
  <domain_specific_prohibitions>

## Mutable Rules (can be updated via meta-review with user approval)

- **Budget:** <budget_value> <budget_unit> per experiment
- **Surface (mutable files):**
  <for each mutable pattern>
  - `<file_pattern>` — <purpose>
  </for>
- **Keep Threshold:** <primary_metric_name> must improve by >= <keep_threshold> <keep_threshold_unit>
- **Mutation Strategy:**
  - Prefer single-variable changes (change one thing per experiment)
  - All else being equal, simpler is better
  - A tiny improvement from deleting code? Definitely keep
  - A tiny improvement from adding 50 lines of complexity? Probably not worth it
  <domain_specific_strategy>

## The Loop Algorithm

<tier_specific_loop_from_template_1>

## Three-Layer Judgment

### L1: Metric Check (automatic, every experiment)

```
Extract <primary_metric_name> from evaluation output.
Compare: new_value <better_operator> best_value <threshold_operator> <keep_threshold>?

<if repetitions > 1>
Run <repetitions> evaluations. Take the median.
</if>

Check all guard metrics:
<for each guard>
  <guard_name>: extract from `<guard_command>`, verify <guard_direction> <guard_floor>
</for>

PASS: metric improved AND all guards met
FAIL: metric not improved OR any guard violated
```

### L2: Compliance Check (automatic, every experiment)

```
<for each l2_check>
Check: <check_description>
Command: `<check_command>`
Expected: <expected_result>
FAIL if: <failure_condition>
</for>
```

<domain_specific_l2_checks>

### L3: Expert Review (conditional, tacit knowledge)

**Trigger conditions (any one triggers L3):**
- Metric improvement > 2x the rolling average improvement
- Git diff shows > 50 lines deleted
- Changes touch files at the edge of the Surface boundary
- Every <l3_periodic_interval> keeps (periodic audit)

**Expert prompt:**

You are a senior expert in <domain>. Review this experiment:

```
[git diff of the change]
[metric change: old_value -> new_value]
[guard metric values]
```

Use your professional intuition (Michael Polanyi's tacit knowledge).

Ask yourself:
- Is this improvement "genuine progress" or "clever evasion"?
- As a senior engineer on this project, would you merge this?
- Is there anything the data doesn't tell you, but your experience
  makes you uneasy about?

Your judgment can go beyond written rules. If your intuition says
something is wrong, say so, even if you can't fully articulate why.

Verdict: APPROVE / REJECT (with reason) / FLAG (mark for meta-review)

## Domain-Specific Guidance

<domain_specific_guidance>

### Common Optimization Directions
<domain_optimization_directions>

### Known Pitfalls
<domain_pitfalls>

### Engineering Methodology
<domain_methodology>

## Meta-Review Triggers

Trigger meta-review when ANY condition is met:
- <consecutive_discards> consecutive discards (default: 5)
- <consecutive_crashes> consecutive crashes (default: 3)
- User manually interrupts the loop
- <periodic_review_interval> experiments completed (default: 20)
- Convergence: rolling average improvement < <convergence_threshold> for <convergence_window> rounds
````

---

## Template 3: Generated meta-review.md

Write to: `.claude/skills/autoresearch-<domain>/meta-review.md`

````markdown
# Meta-Review Protocol

## When This Runs

This protocol is triggered by SKILL.md when meta-review conditions are met.
Read `autoresearch/results.tsv` and `autoresearch/knowledge.md` before proceeding.

## Analysis Dimensions

### 1. Hit Rate Analysis

Count keep vs total experiments:
```
keep_rate = count(status == "keep") / count(all experiments)
```

| Rate | Interpretation | Action |
|------|---------------|--------|
| > 40% | Strategy too conservative, most changes work | Suggest more aggressive, structural mutations |
| 10-40% | Healthy range | No change needed |
| < 10% | Strategy too aggressive or direction is wrong | Suggest convergence or strategy pivot |

### 2. Diminishing Returns

Plot improvement magnitude of the last 10 keeps:
- If trend is flat or declining → approaching local optimum
- Suggest: expand Surface, try architectural changes, or declare converged

### 3. Crash Pattern Analysis

Group crashes by error type:
- If same error appears 3+ times → add specific prohibition to Domain-Specific Guidance
- If crashes cluster in a specific file → consider removing that file from Surface

### 4. Mutation Direction Retrospective

Categorize each experiment by change type (e.g., "hyperparameter", "architecture", "algorithm", "config"):
- Which categories have highest keep rate?
- Reinforce effective categories in Mutation Strategy
- De-prioritize categories with 0% keep rate over 5+ attempts

### 5. Knowledge Distillation

Review all `[CANDIDATE]` entries in `autoresearch/knowledge.md`:
- If a candidate anti-pattern appeared 2+ times → promote to formal `AP-xxx`
- If a candidate proven pattern appeared 2+ times → promote to formal `PP-xxx`
- If multiple formal entries share a theme → extract `DH-xxx` domain heuristic
- If a formal entry is contradicted by new evidence → mark as `[DEPRECATED]` with reason

## Output: Optimization Proposal

Present each proposed change to the user **one at a time**:

> **Meta-Review Proposal [N/M]:**
>
> **Change:** [specific change to program.md Mutable Rules]
> **Rationale:** [why, based on which analysis dimension]
> **Evidence:** [experiments that support this change]
>
> Approve / Reject / Modify?

## Safety Boundary

You **CANNOT** propose changes to Frozen Rules:
- Primary Metric definition
- Evaluation Command
- Guard Metrics and their floors
- Frozen file list
- Prohibited actions

If you believe Frozen Rules need changing (e.g., the metric is fundamentally wrong
for this project), you may ONLY report this to the user:

> **Frozen Rule Concern:**
> I believe [specific frozen rule] may need reconsideration because [reason].
> This requires re-running `/create-autoresearch` to redefine the optimization.
> I cannot change this within the current autoresearch loop.

## After Approval

1. Apply approved changes to `program.md` Mutable Rules section
2. Update `autoresearch/knowledge.md` with distilled knowledge
3. Commit: `git commit -am "autoresearch: meta-review evolution #<N>"`
4. Resume the experiment loop with the updated protocol
````

---

## Template 4: Generated knowledge-seed.md

Write to: `.claude/skills/autoresearch-<domain>/knowledge-seed.md`

Use `knowledge-seed-generic.md` as the base, then customize for the domain:
- Add domain-specific anti-patterns from the Domain Analyst's "Known Pitfalls"
- Add domain-specific heuristics from the Domain Analyst's "Engineering Methodology"
- Add domain-specific proven patterns from the Domain Analyst's "Common Optimization Directions"

---

## Template 5: results.tsv

Write to: `autoresearch/results.tsv`

```tsv
experiment	commit	primary_metric	guard_metrics	status	description
```

This is an empty TSV with a header row. Each experiment appends one line.

Fields:
- `experiment`: sequential integer (1, 2, 3, ...)
- `commit`: git short hash
- `primary_metric`: the scalar value
- `guard_metrics`: semicolon-separated `name:value` pairs
- `status`: `baseline` | `keep` | `discard` | `crash`
- `description`: brief description of what was changed

---

## Template 6: knowledge.md

Write to: `autoresearch/knowledge.md`

Initialize by copying the content of the generated `knowledge-seed.md`,
then add a header section:

```markdown
# Knowledge Base: <domain>

> This knowledge base grows through experimentation. Entries progress through:
> Candidate → Formal → Domain Heuristic → (optionally) Deprecated
>
> DO NOT manually edit formal entries without evidence. The meta-review protocol
> manages promotions and deprecations based on experimental results.

<content from knowledge-seed.md>
```
```

- [ ] **Step 2: Commit**

```bash
git add skills/create-autoresearch/skill-generation.md
git commit -m "feat: add skill generation templates for all output files"
```

---

### Task 6: Generic Knowledge Seed

**Files:**
- Create: `skills/create-autoresearch/knowledge-seed-generic.md`

Domain-agnostic knowledge that applies to ANY autoresearch loop. During skill generation, this gets copied and augmented with domain-specific knowledge from the Domain Analyst.

- [ ] **Step 1: Write knowledge-seed-generic.md**

```markdown
# Knowledge Seed: Generic

> This seed provides domain-agnostic knowledge applicable to any autoresearch loop.
> During skill generation, domain-specific entries are added by the Domain Analyst.

## Anti-Patterns

### AP-001: Changing multiple variables simultaneously
- Evidence: Universal engineering principle
- Failure reason: Cannot attribute metric change to any single modification. If the metric improves, you don't know which change helped. If it worsens, you don't know which change hurt.
- Lesson: Change one variable per experiment. If you have two ideas, run them as two separate experiments.
- Generalized rule: Single-variable principle — isolate the independent variable.

### AP-002: Optimizing the metric by degrading unmeasured quality
- Evidence: Goodhart's Law ("When a measure becomes a target, it ceases to be a good measure")
- Failure reason: The metric improves but the actual quality of the project decreases. Examples: deleting features to improve performance scores, writing empty tests to improve coverage, reducing resolution to improve FPS.
- Lesson: Guard metrics exist to prevent this. If you find yourself tempted to make a trade-off that "technically" improves the metric, it's probably gaming.
- Generalized rule: If it feels like cheating, it probably is. Genuine optimization improves the metric AND the project.

### AP-003: Making complex changes for marginal gains
- Evidence: Complexity is a cost that compounds over time
- Failure reason: Adding 50 lines of complex code for a 0.1% improvement creates maintenance burden that outweighs the gain. Future experiments build on this complexity, making the codebase harder to reason about.
- Lesson: All else being equal, simpler is better. A tiny improvement from deleting code is always worth keeping. A tiny improvement from adding complexity is usually not.
- Generalized rule: Simplicity bias — treat complexity as a cost in the keep/discard decision.

### AP-004: Retrying a previously failed approach without changes
- Evidence: Deterministic systems produce deterministic results
- Failure reason: If approach X was tried in experiment #N and discarded, trying the exact same approach in experiment #N+5 will produce the same result (assuming the evaluation is deterministic).
- Lesson: Always check the knowledge base before proposing an experiment. If a similar approach failed before, either try a meaningfully different variant or choose a different direction entirely.
- Generalized rule: Read before you write — consult the knowledge base pre-mutation.

### AP-005: Ignoring crash patterns
- Evidence: Repeated crashes waste experiment budget
- Failure reason: If a type of change causes crashes repeatedly (e.g., changing array sizes causes OOM), continuing to try similar changes wastes time.
- Lesson: After 2 crashes from similar changes, add the pattern to anti-patterns and avoid it.
- Generalized rule: Crash = signal, not noise. Learn from it.

## Proven Patterns

### PP-001: Profile before optimizing
- Evidence: Universal performance engineering principle
- Success reason: Optimization without profiling is guessing. Profiling identifies the actual bottleneck, which may not be where you expect.
- Applicable when: Any performance-oriented metric (latency, throughput, FPS, training speed)
- Generalized rule: Measure first, then optimize the measured bottleneck.

### PP-002: Start with the simplest possible change
- Evidence: Occam's Razor applied to engineering
- Success reason: Simple changes are easier to reason about, less likely to crash, and produce cleaner diffs. They also establish whether a direction is promising before investing in complexity.
- Applicable when: Always — the first experiment in a new direction should be simple.
- Generalized rule: Escalate complexity only after simple approaches are exhausted.

### PP-003: Delete before adding
- Evidence: Less code = less complexity = fewer bugs = often better performance
- Success reason: Removing unnecessary code, unused imports, dead branches, or redundant computations often improves the metric for free.
- Applicable when: Before adding new code for optimization, check if there's unnecessary code to remove first.
- Generalized rule: The best code is no code.

## Domain Heuristics

### DH-001: Diminishing returns are normal
- Derived from: General optimization theory
- Confidence: High
- Rule: Early experiments produce large improvements. Later experiments produce smaller improvements. This is expected, not a failure. When improvements become marginal, it's time for a meta-review, not panic.

### DH-002: The bottleneck shifts
- Derived from: Amdahl's Law, Theory of Constraints
- Confidence: High
- Rule: After optimizing component A, the bottleneck moves to component B. Always re-profile after a successful optimization to find the new bottleneck. Don't keep optimizing A when it's no longer the limiting factor.

### DH-003: Reversibility is a feature
- Derived from: Git-based experiment management
- Confidence: High
- Rule: Every experiment should be fully reversible via `git reset`. Never make changes that are hard to undo (e.g., data migrations, external state changes, published artifacts).
```

- [ ] **Step 2: Commit**

```bash
git add skills/create-autoresearch/knowledge-seed-generic.md
git commit -m "feat: add generic knowledge seed with universal patterns"
```

---

### Task 7: Project CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Write CLAUDE.md**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "feat: add project CLAUDE.md"
```

---

### Task 8: Integration Verification

**Files:**
- None (verification only)

- [ ] **Step 1: Verify plugin loads with all files**

Run: `cd /Users/null/Code/github/vibe-agi/autoresearch && claude --plugin-dir . --print-plugins 2>&1`

Expected: Plugin `autoresearch-creator` appears with skill `create-autoresearch`.

- [ ] **Step 2: Verify skill description is correct**

Check that `create-autoresearch` appears in the skill list with the correct description starting with "Use when user wants to set up an autonomous optimization loop".

- [ ] **Step 3: Verify all reference files are accessible**

Run: `ls -la skills/create-autoresearch/`

Expected: All 5 files present:
```
SKILL.md
expert-analysis.md
pillar-dialogue.md
skill-generation.md
knowledge-seed-generic.md
```

- [ ] **Step 4: Verify no placeholder variables in committed files**

Run: `grep -r '<[a-z_]*>' skills/create-autoresearch/SKILL.md | grep -v 'autoresearch-<domain>' | grep -v '<domain' | grep -v template | head -20`

Expected: No matches outside of template references (template variables should only appear in `skill-generation.md` where they are intentional).

- [ ] **Step 5: Final commit — tag v0.1.0**

```bash
git tag -a v0.1.0 -m "Initial release: create-autoresearch factory skill"
```
