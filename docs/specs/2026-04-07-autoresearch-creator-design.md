# Design Spec: create-autoresearch Skill

> A meta-skill that generates project-specific autonomous optimization skills,
> inspired by karpathy/autoresearch's engineering philosophy.

**Date:** 2026-04-07 (initial) / 2026-04-09 (v0.5.0 rewrite)
**Status:** v0.5.0 Draft
**Project:** autoresearch-creator (github.com/vibe-agi/autoresearch-creator)

---

## 0. Changelog

- **v0.5.0** — Major rewrite: pillars.json as single source of truth, pre-experiment sanity check, minimal external knowledge fallback, budget unit flexibility, git safety, defensive code user confirmation, upgrade mechanism, removed Tier 3 interactive loop, flattened knowledge lifecycle, removed Polanyi wording, fixed 23 consistency bugs.
- **v0.4.0** — Defensive code protection (AP-006, Defensive Code Inventory, L3 checklist).
- **v0.3.0** — Language rule (skill communicates in user's language).
- **v0.2.0** — Initial pillar dialogue flow.
- **v0.1.0** — Initial release.

---

## 1. Overview

### 1.1 What This Is

`create-autoresearch` is a factory skill for the Claude Code ecosystem. When invoked, it:

1. Analyzes the user's codebase with 4 specialized expert agents
2. Guides the user through defining an optimization problem
3. Generates a project-specific skill that runs an autonomous iterative optimization loop
4. Supports upgrade: when autoresearch-creator itself upgrades, generated skills can be re-rendered from their persisted pillar decisions

### 1.2 Core Philosophy

Generalize karpathy/autoresearch's engineering methodology into a domain-agnostic pattern:

```
Any optimization problem = State(code/config) x Metric(scalar) x Mutation(change) x Evaluation(run) x Selection(keep/discard)
```

The generated skill embodies this pattern for the user's specific domain, complete with:
- A frozen evaluation harness (ungameable)
- A mutable search surface (what the agent can change)
- A 3-layer judgment system (metric, compliance, secondary review)
- A pre-experiment sanity check (detect codebase drift before each run)
- A self-optimization mechanism (meta-review)
- A knowledge base (flat list of anti-patterns and proven patterns)
- An external knowledge fallback (WebSearch when truly stuck)
- Git safety (never operate on a dirty worktree)

### 1.3 Distribution

- Installed via marketplace: `/plugin marketplace add vibe-agi/vibe-marketplace` then `/plugin install autoresearch-creator@vibe-marketplace`
- Invoked as: `/create-autoresearch` or `/create-autoresearch <domain-hint>`
- Output: a `.claude/skills/autoresearch-<domain>/` directory plus `autoresearch/` state directory in the user's project

---

## 2. Architecture

### 2.1 Two-Layer Design

```
Layer 1: create-autoresearch (the factory)
  - Installed once via marketplace
  - Analyzes project, guides user, generates skill
  - Supports upgrade of existing generated skills

Layer 2: autoresearch-<domain> (the generated skill)
  - Lives in user's project .claude/skills/
  - Runs the autonomous optimization loop
  - Self-optimizes through meta-review
  - Accumulates knowledge over time
  - Reads pillars.json as its source of truth
```

### 2.2 Generated File Structure

In `.claude/skills/autoresearch-<domain>/` (all rendered from pillars.json):

```
SKILL.md              # Loop entry point, pre-experiment sanity check, execution loop
program.md            # Rendered experiment protocol (Frozen + Mutable rules)
meta-review.md        # Rendered self-optimization protocol
knowledge-seed.md     # Domain-general knowledge seed (initial)
```

In project root `autoresearch/`:

```
pillars.json          # SOURCE OF TRUTH — user's pillar decisions + meta-review history
results.tsv           # Experiment log (5 columns, karpathy-compatible format)
knowledge.md          # Growing flat knowledge base (anti-patterns + proven patterns)
run.log               # Most recent experiment output (gitignored)
```

### 2.3 The Single Source of Truth Principle

`pillars.json` is the canonical representation of an autoresearch configuration. All `.md` files are derived outputs. This enables:

1. **Clean upgrade**: re-render .md files from pillars.json with new templates.
2. **Safe meta-review**: meta-review modifies pillars.json, then triggers re-render.
3. **No merge conflicts**: there is no ambiguity about "what was the template vs what was the user's change".

**CRITICAL RULE: Users and agents MUST NOT edit generated .md files directly. All changes flow through pillars.json.**

---

## 3. Factory Skill Flow (`create-autoresearch`)

### 3.1 Invocation Modes

- `/create-autoresearch` — fresh generation, runs full dialogue
- `/create-autoresearch <domain-hint>` — fresh generation with focus hint
- `/create-autoresearch upgrade` — re-render existing skill using current templates

### 3.2 Fresh Generation Flow (4 Phases)

#### Phase 1: Project Analysis (Parallel)

Dispatch 4 expert agents simultaneously:

| Expert | Responsibility | Output |
|--------|---------------|--------|
| Architecture Analyst | Project structure, languages, frameworks, modules, dependencies | Technical landscape + Surface candidates |
| Metric Scout | Existing tests, benchmarks, CI evaluation, monitoring | Quantifiable metrics inventory + Tier hint |
| Runtime Analyst | Build/test/run commands, environment, evaluation commands | Executable command inventory + Budget unit recommendation |
| Domain Analyst | README, docs, business goals, defensive code patterns | Domain context + Defensive Code Inventory + Safety verification commands |

Each expert output is capped at ~20 lines (karpathy critique: don't write essays).

After all 4 return, run a **synthesis step** with explicit conflict resolution:

- **Architecture says X is frozen, Metric says X contains the metric:** prefer metric's view (that file is where the metric is measured, move it out of "frozen infrastructure" into "evaluation harness frozen").
- **Architecture says X is mutable, Runtime says X is part of the build:** prefer runtime's view, mark X as frozen.
- **Multiple metric candidates:** present all to user in Phase 2, do not auto-select.
- **Tier disagreement:** present all experts' tier assessments to user in Phase 2, let user decide.
- **Any unresolved conflict:** surface as a question in Phase 2.

#### Phase 2: Pillar Dialogue (Sequential)

Walk the user through confirming each pillar one at a time. Communication uses the user's language; technical terms stay in English.

**Steps (order matters — each builds on previous):**

1. **Domain Understanding** — Confirm what the project does and its core goal.
2. **Metric Selection** — Present discovered metrics, user picks primary metric + direction.
3. **Guard Metrics** — Experts simulate "what could go wrong without this guard", user selects.
4. **Surface Definition** — Propose mutable file scope, user adjusts.
5. **Protected Patterns Confirmation** — Display Defensive Code Inventory, user confirms each entry (or removes false positives).
6. **Harness Confirmation** — Confirm evaluation command and extraction logic.
7. **Budget Setting** — Propose budget with domain-appropriate unit (not always seconds).
8. **Keep Threshold** — Minimum metric improvement to keep an experiment.
9. **Tier Classification** — Tier 1 (deterministic) or Tier 2 (noisy) — Tier 3 removed from v0.5.0.

After all pillars confirmed, summarize the complete configuration for final approval.

#### Phase 3: Skill Generation

1. **Write `autoresearch/pillars.json`** — the source of truth containing all confirmed pillars.
2. **Render `.claude/skills/autoresearch-<domain>/*.md`** from templates + pillars.json.
3. **Initialize `autoresearch/results.tsv`** with header (5 columns).
4. **Initialize `autoresearch/knowledge.md`** from the appropriate knowledge-seed.
5. **Do NOT create git branch** — that happens at first run of the generated skill.

#### Phase 4: Review and Commit

1. Present generated files for user review.
2. If user requests changes → modify pillars.json and re-render.
3. Once approved, check working tree state:
   - If clean except for generated files: ask "commit now?"
   - If user has unrelated changes: explicitly state we will only stage generated files, then ask "commit the generated files only?"
4. Only commit after explicit user confirmation.
5. Commit command stages only specific paths:
   ```bash
   git add .claude/skills/autoresearch-<domain>/ autoresearch/pillars.json autoresearch/results.tsv autoresearch/knowledge.md
   git commit -m "feat: generate autoresearch skill for <domain>"
   ```

### 3.3 Upgrade Flow (`/create-autoresearch upgrade`)

1. Read `autoresearch/pillars.json` to get the saved pillar decisions.
2. Compare `pillars.json.generator_version` with the current autoresearch-creator version.
3. If same version: report "Already up to date" and exit.
4. If different: read the templates from the current autoresearch-creator version.
5. Re-render all generated .md files from templates + pillars.json.
6. Update `pillars.json.generator_version` and `last_updated_at`.
7. Show git diff of the re-rendered files.
8. Ask user to confirm before writing.
9. On confirmation: commit the changes with message `"chore: upgrade autoresearch-<domain> to vX.Y.Z"`.

The upgrade preserves `results.tsv` and `knowledge.md` — they are user-accumulated assets.

---

## 4. The Five Pillars

### 4.1 Metric

A single primary scalar with clear direction (`higher-is-better` or `lower-is-better`).

Plus optional guard metrics that act as hard constraints:

```
maximize(primary_metric) subject to:
  guard_1 operator floor_1
  guard_2 operator floor_2
  ...
```

Guard metrics are selected during skill creation via expert simulation ("what could go wrong without this guard").

### 4.2 Surface

The files/artifacts the agent is allowed to modify. Everything outside the surface is frozen.

### 4.3 Budget

Fixed resource cap per experiment that makes results comparable. **Unit is domain-dependent:**

| Unit | Applicable Domains |
|------|-------------------|
| `seconds` | ML training, compiler benchmarks, general wall-clock scenarios |
| `iterations` | Iterative algorithms, training step counts |
| `api_calls` | LLM experiments, rate-limited APIs |
| `tokens` | LLM cost-sensitive experiments |
| `dollars` | Cloud resource-heavy experiments |
| `memory_mb` | Resource-constrained optimization |
| `requests` | Load testing, HTTP throughput |

Domain Analyst recommends a default unit; user confirms or changes it.

Budget also includes `repetitions` for noisy metrics (Tier 2): run N times, take median.

### 4.4 Harness

Frozen evaluation code/commands:
- The evaluation command itself
- Metric extraction logic (grep/jq/parse)
- Guard metric extraction commands
- Repetition count

The agent cannot modify any of these.

### 4.5 Protocol

The rules governing the experiment loop:
- Keep threshold (minimum metric improvement)
- Mutation strategy preferences
- Loop mode: Tier 1 (autonomous, deterministic) or Tier 2 (autonomous, noisy with repetitions)
- Meta-review trigger thresholds (with sensible defaults)

---

## 5. The Experiment Loop

### 5.1 Pre-Experiment Sanity Check (runs before EVERY experiment)

Four cheap checks to detect codebase drift:

1. **File existence check** — every file in Surface and Frozen lists still exists.
2. **Protected pattern integrity** — for each Protected Pattern, grep verifies the pattern is still present in its declared file.
3. **Evaluation command startup** — smoke test that the eval command can at least launch (not full run).
4. **Metric extraction health** — baseline eval output still parses to a finite, non-zero number consistent with previous baseline (within 10x).

If ANY check fails:
- PAUSE the loop.
- Report specific failure(s) to the user.
- Offer options:
  - (1) User will fix manually and resume
  - (2) Run `/create-autoresearch upgrade` to re-sync
  - (3) Abort

Do NOT auto-recover. Do NOT auto-stash.

### 5.2 Git Safety Check (at first run only)

Before creating the `autoresearch/<domain>` branch, verify the working tree is clean:

```bash
git status --porcelain
```

If non-empty:
- Show the dirty files.
- Require user to either commit, stash, or explicitly abort.
- Do NOT auto-commit or auto-stash.
- Only proceed after a re-check shows clean.

### 5.3 The Autonomous Loop

```
NEVER STOP. Run continuously until user interrupts or external knowledge fallback exhausts.
If out of ideas: think harder, re-read in-scope files, try combining near-misses, try radical changes.
Boredom is not a stopping condition.

loop:
  0. Pre-Experiment Sanity Check (section 5.1)
     If any check fails → PAUSE and report

  1. Read autoresearch/knowledge.md — check anti-patterns, reference proven patterns
  2. Propose ONE improvement hypothesis based on analysis + knowledge
  3. Modify files within Surface
  4. git commit with descriptive message
  5. Run evaluation:
     - Run the eval command (with <repetitions> runs if noisy, take median)
     - Extract primary metric and guard metrics
  6. Three-Layer Judgment (section 6)
  7. Log result to results.tsv (always BEFORE any git reset)
  8. Decision:
     - KEEP (all layers pass) → commit stays, advance
     - DISCARD → git reset --hard HEAD~1
  9. Update autoresearch/knowledge.md with outcome (append to flat list)
 10. Check meta-review triggers (section 7)
 11. If plateau detected AND meta-review proposes no direction → trigger external knowledge fallback (section 8)
 12. Back to 0
```

**Key ordering:** results.tsv logging MUST happen before `git reset --hard`, otherwise the commit hash is lost.

---

## 6. Three-Layer Judgment System

### 6.1 Layer 1: Metric Check (Automatic, Every Round)

```
1. Extract primary_metric from evaluation output using metric_extraction command.
2. If repetitions > 1: run <repetitions> times, take median.
3. Compare: new_value improves over current best by at least keep_threshold?
4. Extract each guard metric and verify it meets its floor.

PASS if: primary metric improved AND all guard metrics satisfied.
FAIL otherwise.
```

### 6.2 Layer 2: Compliance Check (Automatic, Every Round)

Domain-specific mechanical checks defined at skill creation time. Examples:

| Domain | L2 Checks |
|--------|-----------|
| ML Training | No new dependencies, VRAM within limit, no val data leakage |
| Web Performance | Visual regression test, e2e tests pass, no feature removal |
| API Latency | Response body validation, integration tests pass |
| Test Coverage | Mutation testing score, assertion density check |
| Compiler | Output correctness diff, sanitizer clean |
| Database | Query result-set diff, safe config allowlist |
| Game Rendering | Visual reference comparison (SSIM/LPIPS) |
| Go Concurrency | `go test -race` passes |

L2 checks also run **mechanical Protected Pattern verification**: for each declared Protected Pattern, grep to confirm it's still present in its file. This is the automated counterpart to L3's judgment.

### 6.3 Layer 3: Secondary Review (Conditional)

**Mechanical trigger conditions** (any one triggers L3):

1. Primary metric improvement > 2x rolling average improvement (suspiciously good)
2. Git diff shows > 50 lines deleted (code removal is a red flag)
3. Git diff matches any "defensive code removal" grep pattern (see below)
4. Every 10 accumulated keeps (periodic audit, hardcoded default)

**Defensive code removal grep patterns:**

```
^-.*\.Clone\(\)
^-.*sync\.(Mutex|RWMutex)
^-.*\b(validate|sanitize|escape)
^-.*\b(try|catch|rescue|recover)
^-.*rate[_-]?limit
^-.*(encrypt|decrypt|hash|verify)
```

These are mechanical heuristics — if a diff removes a line matching any of these, L3 is triggered.

**Expert review prompt:**

```
You are a senior engineer reviewing a proposed change.

Change:
[git diff]

Metric change: old → new
Guard values: [...]

Review checklist:
1. Does this improvement come from real optimization or from removing a protection?
2. For each removed line, ask: why did this line exist?
3. Could this introduce: concurrency issues, security holes, correctness bugs, compliance violations?
4. Would a reasonable code reviewer merge this PR?

For EACH line the diff removes:
- What was it protecting?
- Has that protection been preserved elsewhere in the diff?

Common defensive patterns to watch:
- Cloning/copying shared data (concurrency)
- Locks and synchronization (thread safety)
- Input validation and sanitization (security)
- Error handling and fallbacks (fault tolerance)
- Rate limiting and circuit breakers (stability)
- Checksums and assertions (correctness)

If ANY removed line served a protective purpose, REJECT.

Verdict: APPROVE | REJECT <reason> | FLAG <observation>
```

Note: the word "Polanyi" is intentionally absent. This is a secondary LLM review with an explicit checklist, not magical intuition.

---

## 7. Meta-Review

### 7.1 Triggers (hardcoded defaults)

- 5 consecutive discards
- 3 consecutive crashes
- User manually interrupts the loop
- Every 20 experiments completed (periodic health check)
- Rolling average improvement < 0.1% for 10 rounds (convergence)

### 7.2 Analysis Dimensions

1. **Hit rate analysis**: keep / total
   - \> 40%: strategy too conservative, suggest aggression
   - 10-40%: healthy
   - < 10%: strategy too aggressive or direction wrong
2. **Diminishing returns**: trend of improvement magnitude in recent keeps
3. **Crash pattern analysis**: group by error type
4. **Mutation direction retrospective**: which change categories have highest keep rate
5. **Knowledge distillation**: review flat entries in knowledge.md and suggest cleanup

### 7.3 Output: Modifications to pillars.json

Meta-review proposes changes to `pillars.json.mutable` fields. Examples:

- Lower keep_threshold
- Expand surface_patterns
- Adjust mutation_strategy_notes
- Add new domain-specific warnings

**Meta-review CANNOT propose changes to `pillars.json.frozen`.** If it believes frozen fields need changing, it can only report to user and recommend re-running `/create-autoresearch` from scratch.

### 7.4 User Approval Flow

Meta-review presents each proposed change item-by-item:

```
Meta-Review Proposal N/M:

Change: pillars.mutable.keep_threshold: 0.01 → 0.005
Rationale: Diminishing returns detected in last 10 keeps (avg improvement dropped from 2.1% to 0.3%)
Evidence: Experiments #N, #M, #K showed <0.5% improvements

Approve / Reject / Modify?
```

### 7.5 Application

After user approval:
1. Apply changes to `autoresearch/pillars.json` (mutable section only).
2. Append entry to `autoresearch/pillars.json.meta_review_history` with timestamp, rationale, change diff.
3. Re-render .md files from updated pillars.json.
4. Commit: `git commit -am "autoresearch: meta-review #N"`.
5. Resume the loop.

**Note:** Re-rendering is mechanical. Meta-review never touches .md files directly.

---

## 8. External Knowledge Fallback

### 8.1 Trigger

Only as a fallback after meta-review has run and produced no useful direction:
- Plateau detected (meta-review convergence)
- knowledge.md has no relevant pattern for the current bottleneck
- Meta-review analysis yielded no actionable changes

### 8.2 Protocol

```
When triggered:
  1. Formulate ONE targeted search query based on current bottleneck
     Example: "FlashAttention-3 benchmark 2025 pytorch"
     Example: "Go sync.Pool alternatives high contention"
  2. Use WebSearch to fetch results (max 3 searches per fallback session)
  3. For each relevant finding:
     - Extract the actionable technique
     - Add it as a candidate experiment with source URL
  4. Run these candidate experiments through the normal loop
  5. Log source URL in results.tsv description field
  6. External findings become "proven patterns" only after local validation passes
```

### 8.3 Constraints

- Hard cap: 3 WebSearch calls per fallback session.
- Prefer official docs, arXiv papers, GitHub issues over random blogs.
- All external findings must pass the full 3-layer judgment before being trusted.
- Record source URL in knowledge.md entries derived from external knowledge.

### 8.4 Why This Is Minimal

~20 lines added to generated SKILL.md. No tiered source rankings, no domain-specific source lists, no citation format system. WebSearch + common sense. Complexity grows only if observed failures require it.

---

## 9. Knowledge Base

### 9.1 Structure (Flattened in v0.5.0)

Two flat lists — no lifecycle, no promotion/demotion:

```markdown
# Knowledge Base: <domain>

## Anti-Patterns

### AP-NNN: <title>
- Evidence: experiments #N, #M (or: universal principle)
- What failed: <brief description>
- Why: <root cause>
- Rule: <generalized lesson>

## Proven Patterns

### PP-NNN: <title>
- Evidence: experiments #N, #M
- What worked: <brief description>
- Why: <root cause>
- Rule: <generalized lesson>
- Source: <URL if from external knowledge>
```

### 9.2 Integration with Loop

**Pre-mutation:** read knowledge.md, consult anti-patterns and proven patterns when proposing a hypothesis.

**Post-experiment:** append a new entry describing the outcome. No "candidate" state, no promotion pipeline. Just append.

**During meta-review:** review entries, consolidate duplicates, remove contradicted entries. This is the only cleanup mechanism.

### 9.3 Knowledge Seed

`knowledge-seed-generic.md` (domain-general) is bundled with autoresearch-creator. It contains universal anti-patterns (AP-001 to AP-006) and proven patterns (PP-001 to PP-003).

At skill creation time, the Domain Analyst produces domain-specific entries which are appended to the seed, producing the project's initial `knowledge.md`.

---

## 10. Domain Tier Classification

### 10.1 Tiers (v0.5.0)

| Tier | Feedback | Mode | Examples |
|------|---------|------|---------|
| Tier 1 | Seconds-to-minutes, deterministic | Autonomous | ML training, compiler benchmarks, DB queries |
| Tier 2 | Seconds-to-minutes, noisy | Autonomous with N-repetition median | Web performance, API latency, game FPS |

**Tier 3 (interactive, slow feedback) is removed from v0.5.0.** Use cases like A/B testing or SEO ranking fundamentally differ from the fast-loop model. They will be considered in a future version if real demand emerges.

### 10.2 Tier Detection

Domain Analyst assesses:
- Can the metric be evaluated locally in under 10 minutes? (no → not supported in v0.5.0)
- Is the metric deterministic? (no → Tier 2)
- Feedback cycle time?

If the domain is inherently slow-feedback, the factory reports: "This domain's evaluation cycle is [time] which exceeds our current supported range. autoresearch-creator v0.5.0 does not support interactive loops. Consider a proxy metric that can be evaluated locally, or wait for a future version with interactive support."

### 10.3 Tier-Specific Adaptations

**Tier 1:**
- `repetitions = 1` in harness
- Keep threshold directly compared

**Tier 2:**
- `repetitions = 3..5` in harness (recommended by Domain Analyst)
- Median or mean taken before comparison
- Keep threshold may require statistical significance check (optional)

Only ONE of these blocks is rendered into the generated SKILL.md — never both.

---

## 11. pillars.json Schema

### 11.1 Complete Schema

```json
{
  "schema_version": "1.0",
  "generator_version": "0.5.0",
  "domain": "go-memory",
  "created_at": "2026-04-09T10:00:00Z",
  "last_updated_at": "2026-04-09T10:00:00Z",
  "project_context": "Brief summary of what the project does and its goals",

  "frozen": {
    "primary_metric": {
      "name": "heap_alloc_bytes",
      "direction": "lower-is-better",
      "extraction": "grep 'heap_alloc_bytes' bench.log | awk '{print $2}'"
    },
    "guard_metrics": [
      {
        "name": "race_tests_pass",
        "floor": "all_pass",
        "command": "go test -race ./..."
      }
    ],
    "harness": {
      "command": "go test -bench . -benchmem",
      "repetitions": 3
    },
    "frozen_files": [
      {"path": "testdata/", "reason": "evaluation data"},
      {"path": "bench_test.go", "reason": "benchmark harness"}
    ],
    "protected_patterns": [
      {
        "file": "handler.go",
        "grep_pattern": "header\\.Clone\\(\\)",
        "protects": "concurrency safety",
        "risk_if_removed": "data race under concurrent HTTP requests"
      }
    ],
    "prohibited_actions": [
      "Do not add external dependencies without L3 expert approval",
      "Do not modify benchmark data"
    ]
  },

  "mutable": {
    "budget": {
      "value": 300,
      "unit": "seconds"
    },
    "surface_patterns": [
      {"pattern": "src/**/*.go", "purpose": "all application Go code"}
    ],
    "keep_threshold": {
      "value": 1.0,
      "unit": "percent"
    },
    "mutation_strategy_notes": [
      "Prefer pool reuse over allocation",
      "Profile before optimizing"
    ],
    "meta_review_triggers": {
      "consecutive_discards": 5,
      "consecutive_crashes": 3,
      "periodic_interval": 20,
      "convergence_threshold_percent": 0.1,
      "convergence_window": 10
    }
  },

  "tier": 1,

  "meta_review_history": [
    {
      "timestamp": "2026-04-09T12:00:00Z",
      "generator_version": "0.5.0",
      "changes": [
        {
          "field": "mutable.keep_threshold.value",
          "from": 1.0,
          "to": 0.5,
          "reason": "Diminishing returns detected"
        }
      ]
    }
  ]
}
```

### 11.2 Rendering Rules

1. All `.frozen.*` fields render into the "Frozen Rules" section of program.md.
2. All `.mutable.*` fields render into the "Mutable Rules" section of program.md.
3. Tier-specific templates conditionally render only the relevant loop variant.
4. `meta_review_history` is informational only, not rendered into any .md file (git log is the official history).

### 11.3 Invariants

- `schema_version` is the pillars.json format version (not the autoresearch-creator version).
- `generator_version` is the autoresearch-creator version that last rendered the .md files.
- After upgrade, `generator_version` is updated but other fields are unchanged unless explicitly modified.

---

## 12. Git Safety

### 12.1 Principles

Autoresearch inherently performs `git commit` and `git reset --hard` operations. Because these can destroy uncommitted user work, the skill must operate with strong safety guarantees:

- **Never operate on a dirty worktree.** Always pre-check with `git status --porcelain`.
- **Never auto-stash.** Stashing is user state; only users should decide to stash.
- **Never auto-commit user changes.** Only commit files the skill itself produced.
- **Stage files explicitly by path.** Never use `git add .` or `git add -A`.

### 12.2 Enforcement Points

Three places the check must happen:

1. **Factory Phase 4 (before committing generated files):** if user has unrelated changes, explicitly tell them we will only stage generated files, then ask permission.
2. **Generated skill first run (before creating autoresearch branch):** if dirty, require user to commit/stash/abort manually.
3. **Every experiment iteration:** the pre-experiment sanity check includes verifying the worktree is clean (except for the in-progress experiment commit).

### 12.3 What the User Sees on Dirty Worktree

```
⚠ Working tree is not clean:

  M  src/handler.go
  ?? notes.txt

The autoresearch loop auto-commits and runs `git reset --hard`.
Running with dirty state risks data loss.

Options:
  1. I will commit/stash my changes, then tell you to re-check
  2. Abort this run

(I will NOT auto-commit or auto-stash your changes.)
```

---

## 13. Upgrade Mechanism

### 13.1 When the User Should Run Upgrade

- After `/plugin update autoresearch-creator` shows a version bump.
- When the generated SKILL.md's startup self-check warns about version drift.

### 13.2 Self-Check at Generated Skill Start

Every invocation of `/autoresearch-<domain>` first runs:

```
1. Read autoresearch/pillars.json
2. Compare pillars.generator_version with current autoresearch-creator version
3. If current > saved:
   - Show a brief notice: "autoresearch-creator has upgraded from vX.Y.Z to vA.B.C."
   - Ask: "Run /create-autoresearch upgrade to refresh this skill with new features?"
4. User can: upgrade now / skip this session / continue with outdated skill
```

### 13.3 Upgrade Command Flow

```
/create-autoresearch upgrade

1. Load autoresearch/pillars.json
2. Validate: schema_version is compatible with current autoresearch-creator
3. Get the current version's templates
4. Re-render all .md files from pillars + current templates
5. Compute diff vs existing files
6. Show diff summary to user:
   "SKILL.md: 23 lines changed (new pre-experiment sanity check added)
    program.md: 5 lines changed (Protected Patterns now use grep patterns)
    meta-review.md: 10 lines changed (analysis dimensions updated)"
7. Ask user to approve
8. On approval:
   - Write new .md files
   - Update pillars.json.generator_version and last_updated_at
   - Stage only the affected paths
   - Commit: "chore: upgrade autoresearch-<domain> to vA.B.C"
9. Never touch results.tsv, knowledge.md
```

### 13.4 Schema Version Incompatibility

If `pillars.json.schema_version` is older than the current autoresearch-creator's supported minimum:
- Report the incompatibility clearly.
- Offer a schema migration (future work).
- For v0.5.0: print manual migration steps and exit.

---

## 14. Complete Lifecycle

```
/create-autoresearch <domain-hint>
      │
      ▼
Phase 1: 4 experts analyze codebase in parallel
      │
      ▼
Phase 2: Pillar dialogue (9 steps)
      │
      ▼
Phase 3: Write pillars.json + render .md files
      │
      ▼
Phase 4: Review, git safety check, commit (with explicit approval)
      │
      ▼
/autoresearch-<domain>  ──────────────────────────┐
      │                                            │
      ▼                                            │
Version self-check                                 │
      │                                            │
      ▼                                            │
Git safety check (first run)                       │
      │                                            │
      ▼                                            │
Pre-experiment sanity check  ◄────────────────────┐│
      │ pass                                      ││
      ▼                                           ││
Read knowledge.md                                 ││
      │                                           ││
      ▼                                           ││
Propose & execute experiment                      ││
      │                                           ││
      ▼                                           ││
3-layer judgment → log → keep/discard             ││
      │                                           ││
      ▼                                           ││
Update knowledge.md                               ││
      │                                           ││
      ▼ (trigger?)                                ││
Meta-review → propose pillar changes → approve    ││
      │                                           ││
      ▼ (if meta-review stuck)                    ││
External knowledge fallback (WebSearch)           ││
      │                                           ││
      └──────────────────────────────────────────┘│
                                                  │
      [auto-creator version bump detected]        │
      │                                           │
      ▼                                           │
/create-autoresearch upgrade                      │
      │                                           │
      ▼                                           │
Re-render .md from pillars.json + new templates   │
      │                                           │
      └──────────────────────────────────────────┘
```

---

## 15. Success Criteria (Empirical)

v0.5.0 ships only if ALL the following hold:

### 15.1 Static Correctness
- All 23 identified consistency bugs fixed.
- No placeholder variables remain in generated .md files.
- All variables used in templates are either collected from dialogue or have explicit defaults.

### 15.2 Multi-Domain Simulation (Coverage)
Successful dry-run skill generation against 3 real repos:
1. `karpathy/autoresearch` — regression test (can we recreate the original setup?)
2. A real Go project (Web backend) — verify Protected Patterns detection
3. A real Next.js project (Web frontend) — verify different tier handling

For each: generated pillars.json + .md files are manually inspected for reasonableness.

### 15.3 Real End-to-End Execution (Depth)
Full run on a controlled benchmark project at `/Users/null/Code/github/vibe-agi/testapp`:
- 20 experiments completed
- Baseline → final metric shows measurable improvement
- results.tsv and knowledge.md properly updated
- No git state corruption
- Pre-experiment sanity check has been exercised (at least one drift scenario)

### 15.4 Gaming Attempt Rejection
Manually constructed "cheating" experiment (defensive code removal) is rejected by:
- Protected Pattern grep check in L2, OR
- Secondary review in L3

Both paths should be exercised at least once.

### 15.5 Git Safety
- Dirty worktree is detected and refused at factory Phase 4.
- Dirty worktree is detected and refused at generated skill first run.
- User's uncommitted code is never lost or overwritten during any step.

---

## 16. Out of Scope (v0.5.0)

- **Tier 3 interactive loop**: removed until real demand emerges.
- **Cross-project knowledge sharing**: each project's knowledge.md is isolated.
- **Multi-objective Pareto optimization**: primary + guards is the model.
- **Distributed/multi-GPU experiment execution**.
- **Integration with external experiment tracking** (W&B, MLflow).
- **Automated tier detection**: v0.5.0 relies on Domain Analyst recommendation + user confirmation.
- **Automatic schema migration**: if `pillars.json.schema_version` is incompatible, user must manually migrate.
- **Re-analysis triggered drift detection**: v0.5.0 detects drift via sanity check only; full re-analysis on drift is future work.
- **External knowledge rating / source credibility system**: v0.5.0 uses simple WebSearch without tiered source rankings.

---

## 17. Non-Goals (Never)

- **Editing the evaluation harness during optimization**: metric and eval command are frozen, always.
- **Auto-stash or auto-commit user changes**: git state is user state.
- **Removing defensive code to improve metrics**: this is the defining failure mode we guard against.
- **Self-modification of L1/L2/L3 judgment logic**: meta-review only touches mutable strategy, never the judgment framework.
- **Unbounded autonomous operation without sanity checks**: every experiment has a pre-check.
