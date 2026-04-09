# Pillar Dialogue Protocol

## Rules

- **Speak the user's language.** Detect from the user's messages. Technical terms (metric names, commands, file paths) stay in English; explanations, questions, and summaries use the user's language.
- **One question at a time.** Never batch.
- **Expert-backed defaults.** Every question has a recommended answer from expert analysis.
- **User always decides.** Suggest, never override.
- **Build sequentially.** Each pillar builds on the previous confirmed pillar.
- **Record answers directly into the in-memory pillars object.** At the end, this becomes `autoresearch/pillars.json`.

---

## Step 1: Domain Understanding

**Purpose:** Establish shared context. Everything else flows from this.

Present the domain analyst's summary and confirm:

> Based on my analysis, your project is **[project purpose]**, serving **[target users]**.
> The core value proposition is **[value proposition]**.
>
> Is this understanding correct? Anything to add or correct?

Wait for confirmation. Record the final agreed text in `pillars.project_context`.

---

## Step 2: Metric Selection

**Purpose:** Choose the single primary scalar to optimize.

Present the metric scout's inventory:

> I found these quantifiable metrics in your project:
>
> | # | Metric | Current Value | How Measured | Direction |
> |---|--------|--------------|-------------|-----------|
> | 1 | [name] | [value]      | `[command]` | [higher/lower is better] |
> | 2 | [name] | [value]      | `[command]` | [higher/lower is better] |
>
> Which should I optimize? If none fit, describe what you want and I'll help define one.

Record in `pillars.frozen.primary_metric`:
- `name`
- `direction` (`lower-is-better` | `higher-is-better`)
- `extraction` (the shell command to extract the scalar)

---

## Step 3: Guard Metrics (Simulation-Driven)

**Purpose:** Set hard constraints that prevent gaming the primary metric.

For each candidate guard, run a domain expert simulation:

> To protect against gaming, I recommend these guard metrics.
> For each one, here's what could go wrong WITHOUT it:
>
> **Guard 1: [name] [operator] [floor]**
> Simulation: Without this guard, the agent might [concrete bad scenario].
> Example: "Without visual regression (SSIM >= 0.95), the agent might reduce
> rendering resolution to 10% — FPS goes to 300 but the game is unplayable."
>
> **Guard 2: ...**
>
> Which guards do you want? You can adjust operators and floor values.

Record each selected guard in `pillars.frozen.guard_metrics[]`:
- `name`
- `operator` (`>=` | `<=` | `==` | `all_pass`)
- `floor`
- `command` (how to measure/verify)

If the user selects none, warn once: "Running without guards means the agent can make any trade-off to improve [primary metric]. Are you sure?"

---

## Step 4: Surface Definition

**Purpose:** Define what the agent can and cannot modify.

Present the architecture analyst's recommendations:

> I recommend this mutable surface (files the agent can modify):
>
> **Mutable (can change):**
> - `[pattern]` — [why it's safe to modify]
>
> **Frozen (must NOT change):**
> - `[pattern]` — [why: eval harness / data / infra]
>
> Want to add or remove files from either list?

Record:
- `pillars.mutable.surface_patterns[]` as `{pattern, purpose}` objects
- `pillars.frozen.frozen_files[]` as `{path, reason}` objects

---

## Step 5: Protected Patterns Confirmation

**Purpose:** Confirm which pieces of defensive code must NOT be removed.

Present the domain analyst's Defensive Code Inventory:

> I scanned your Surface files and found these defensive code patterns.
> These exist for non-functional reasons (concurrency, safety, correctness)
> and will be marked as "do not remove even for metric gains":
>
> | # | File | Pattern | Protects | Risk If Removed |
> |---|------|---------|----------|-----------------|
> | 1 | handler.go | `header.Clone()` | concurrency safety | data race under concurrent requests |
> | 2 | db.go | `sync.Mutex` | thread safety | race on shared connection |
> | 3 | ... |
>
> Please review each entry:
> - (K)eep — yes, this is protective, don't let the agent touch it
> - (R)emove — false positive, this code is not load-bearing
> - (M)odify — keep but change the protection description or risk
>
> Are there any defensive patterns I missed? Tell me the file and pattern.

For each confirmed pattern, record in `pillars.frozen.protected_patterns[]`:
- `file` (path)
- `grep_pattern` (regex the pattern must match — use regex that works with `grep -E`)
- `min_count` (auto-detect by running `grep -cE '<pattern>' <file>` at confirmation time; use that count as min_count)
- `protects` (what non-functional property)
- `risk_if_removed` (what breaks)

**CRITICAL: Use `grep_pattern`, NOT line numbers.** Line numbers drift; grep patterns survive refactoring.

**CRITICAL: Set `min_count` to the actual count at skill creation time**, not just 1. This catches the case where a defensive pattern appears multiple times in the file and the agent quietly removes one occurrence. Without min_count tracking, grep would still find the remaining instances and miss the removal.

When running grep to set min_count, use the exact command that L2 will use later:
```
count=$(grep -cE '<grep_pattern>' <file>)
```

If the count is 0, the pattern is wrong — adjust the regex.
If the count is unexpectedly high (e.g., 50+), the pattern is too generic — make it more specific.

---

## Step 6: Harness Confirmation

**Purpose:** Lock down the evaluation command. This becomes frozen and ungameable.

Present the runtime analyst's findings:

> The evaluation harness will run:
>
> ```bash
> [full evaluation command]
> ```
>
> Metric extraction: `[grep/jq/parse command]`
>
> Estimated time per evaluation: ~[N] seconds

**CRITICAL: Verify the harness actually runs before freezing it.**

Execute the proposed command ONCE and:
1. Check the exit code
2. Run the metric extraction on the output
3. Verify the extracted value is finite and non-zero
4. Also run each proposed guard metric command and verify it produces the expected kind of output

If ANY verification fails:
- Report the specific failure to the user (missing dependency, import error, wrong path, etc.)
- Do NOT commit the broken command as the harness
- Ask the user to either:
  - Fix the underlying issue (install dependencies, create missing files)
  - Propose a different harness command
  - Abort skill generation
- Re-verify after any fix

Common failures to expect:
- Missing optional dependencies (e.g., ruamel, pytest plugins)
- Test files that require a specific environment
- Commands that need a running service (database, HTTP server)
- File paths that are environment-dependent
- Metric extraction that relies on output format the tool doesn't produce yet

Do NOT skip verification just because the command "looks right". Real
projects have hidden gotchas that only a dry run catches.

After verification passes:

> The evaluation command will be FROZEN. Verified output:
> - Primary metric: [extracted value]
> - Guard metrics: [verified values]
>
> Is this correct?

If the metric is noisy (Tier 2), ask:

> This metric appears noisy. How many repetitions per experiment? (Median will be taken.)
> Suggested: 3 for moderate noise, 5 for high noise.

Record in `pillars.frozen.harness`:
- `command`
- `repetitions` (integer, default 1 for Tier 1, 3-5 for Tier 2)

---

## Step 7: Budget Setting

**Purpose:** Set the resource cap that makes experiments comparable. Unit depends on the domain — NOT always time.

Present the runtime analyst's budget recommendation:

> Based on your domain, I recommend this budget unit:
>
> **Recommended: [unit]** — [why this unit fits your domain]
>
> Available units:
> - `seconds` — wall-clock time (ML training, benchmarks)
> - `iterations` — step count (iterative algorithms)
> - `api_calls` — external API call count (LLM experiments)
> - `tokens` — LLM token count (cost-sensitive)
> - `dollars` — cloud cost (resource-heavy experiments)
> - `memory_mb` — memory budget
> - `requests` — request count (load tests)
>
> Which unit do you want? And what value (resource cap per experiment)?

Record in `pillars.mutable.budget`:
- `value` (number)
- `unit` (one of the enum above)

---

## Step 8: Keep Threshold

**Purpose:** Minimum metric improvement to count as "keep". Karpathy's original uses strict improvement; we allow a threshold to filter out noise.

> Every experiment either improves {{primary_metric.name}} or it doesn't.
> But tiny improvements may be noise. How much improvement counts as real?
>
> Recommended: 1% relative improvement (percent unit) — filters most noise without being too strict.
>
> You can use:
> - **percent** (1.0 = 1% improvement relative to current best)
> - **absolute** (0.001 = improvement of 0.001 in the metric's natural unit)
>
> Which unit and value?

Record in `pillars.mutable.keep_threshold`:
- `value` (number)
- `unit` (`percent` | `absolute`)

---

## Step 9: Tier Classification

**Purpose:** Determine loop mode.

Present expert consensus:

> Based on analysis:
> - Metric can be evaluated locally: [yes/no]
> - Metric is deterministic: [yes/no]
> - Feedback cycle: ~[time]
>
> I classify this as **Tier [1/2]**:
>
> | Tier | Mode | Meaning |
> |------|------|---------|
> | 1 | Autonomous, deterministic | Each experiment produces a clean scalar, no repetition needed |
> | 2 | Autonomous + noise handling | Scalar is noisy, N-run median used |
>
> Does Tier [N] sound right?

Record in `pillars.tier`: `1` or `2`.

**If the domain analyst suggests the project is inherently Tier 3 (interactive, slow feedback, e.g., requires real users / days-long A/B testing):**

> This domain's evaluation cycle appears to be [days/weeks], which requires
> interactive human-in-the-loop optimization. autoresearch-creator v0.5.0
> does not support interactive loops (Tier 3 is out of scope).
>
> Options:
> 1. Define a proxy metric that can be evaluated locally in minutes
>    (e.g., for conversion rate optimization, use a heuristic UX scoring
>    function instead of real conversion numbers)
> 2. Wait for a future version with Tier 3 support
> 3. Cancel this skill creation
>
> What would you like to do?

If the user chooses to define a proxy metric, go back to Step 2.

---

## Final Confirmation

Summarize all confirmed pillars:

> Here's the complete optimization configuration:
>
> **Domain:** [description]
> **Tier:** [1 or 2] — [description]
>
> **Frozen (cannot change):**
> - Primary Metric: [name] ([direction])
> - Eval Command: `[command]` (× [repetitions])
> - Guards: [list]
> - Frozen Files: [N] files
> - Protected Patterns: [N] patterns
>
> **Mutable (meta-review can propose changes):**
> - Budget: [value] [unit] per experiment
> - Surface: [N] mutable patterns
> - Keep Threshold: [value] [unit]
>
> I'll now:
> 1. Write `autoresearch/pillars.json` (source of truth)
> 2. Render `.claude/skills/autoresearch-[domain]/` from templates
> 3. Initialize `autoresearch/results.tsv` and `autoresearch/knowledge.md`
>
> Proceed to generation?

Only proceed to Phase 3 after explicit user approval.
