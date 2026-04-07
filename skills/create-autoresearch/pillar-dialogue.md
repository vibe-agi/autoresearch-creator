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
