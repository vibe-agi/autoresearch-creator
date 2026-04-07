# Design Spec: create-autoresearch Skill

> A meta-skill that generates project-specific autonomous optimization skills,
> inspired by karpathy/autoresearch's engineering philosophy.

**Date:** 2026-04-07
**Status:** Draft
**Project:** autoresearch-creator (github.com/vibe-agi/autoresearch-creator)

---

## 1. Overview

### 1.1 What This Is

`create-autoresearch` is a factory skill for the Claude Code ecosystem. When invoked, it:

1. Analyzes the user's codebase with 4 specialized expert agents
2. Guides the user through defining an optimization problem
3. Generates a project-specific skill that runs an autonomous iterative optimization loop

### 1.2 Core Philosophy

Generalize karpathy/autoresearch's engineering methodology into a domain-agnostic pattern:

```
Any optimization problem = State(code/config) x Metric(scalar) x Mutation(change) x Evaluation(run) x Selection(keep/discard)
```

The generated skill embodies this pattern for the user's specific domain, complete with:
- A frozen evaluation harness (ungameable)
- A mutable search surface (what the agent can change)
- A 3-layer judgment system (metric + compliance + tacit knowledge)
- A self-optimization mechanism (meta-review)
- A knowledge base (error book + proven patterns)

### 1.3 Distribution

- Installed via Claude Code marketplace: `/plugin install autoresearch-creator`
- Invoked as: `/create-autoresearch`
- Output: a `.claude/skills/autoresearch-<domain>/` directory in the user's project

---

## 2. Architecture

### 2.1 Two-Layer Design

```
Layer 1: create-autoresearch (the factory)
  - Installed once via marketplace
  - Analyzes project, guides user, generates skill
  - Run once per optimization goal

Layer 2: autoresearch-<domain> (the generated skill)
  - Lives in user's project .claude/skills/
  - Runs the autonomous optimization loop
  - Self-optimizes through meta-review
  - Accumulates knowledge over time
```

### 2.2 Generated File Structure

In `.claude/skills/autoresearch-<domain>/`:

```
SKILL.md              # Loop entry point: loads program.md, executes the experiment loop
program.md            # Domain-specific experiment protocol (frozen + mutable rules)
meta-review.md        # Self-optimization protocol (referenced by SKILL.md)
knowledge-seed.md     # Domain-general knowledge base (initial seed from factory)
```

In project root `autoresearch/`:

```
results.tsv           # Raw experiment data (commit, metric, guard_metrics, status, description)
knowledge.md          # Project-level knowledge base (grows from seed over time)
run.log               # Most recent experiment output
```

---

## 3. Factory Skill Flow (`create-autoresearch`)

### 3.1 Phase 1: Project Analysis (Parallel)

Dispatch 4 expert agents simultaneously:

| Expert | Responsibility | Output |
|--------|---------------|--------|
| Architecture Analyst | Project structure, languages, frameworks, modules, dependencies | Technical landscape map |
| Metric Scout | Existing tests, benchmarks, performance metrics, monitoring, CI evaluation steps | Quantifiable metrics inventory |
| Runtime Analyst | Build/test/run commands, environment dependencies, how to launch and evaluate | Executable command inventory |
| Domain Analyst | README, docs, comments, business goals, user value proposition | Domain context summary |

### 3.2 Phase 2: User Dialogue (Sequential)

After expert analysis, confirm with the user one pillar at a time:

1. **Domain understanding**: "I understand your project does X, with core goal Y. Correct?"
2. **Metric selection**: "I found these quantifiable metrics: [list]. Which do you want to optimize? Direction (higher/lower)?"
3. **Guard metrics** (simulation-driven):
   - Experts recommend 2-4 candidate guard metrics
   - For each, simulate: "Without this guard, the agent might do X"
   - Present simulation results to user for selection
4. **Surface definition**: "I suggest these files as the mutable scope: [list]. Add/remove?"
5. **Harness confirmation**: "Evaluation can run via `[command]`, estimated [N]s per run. Correct?"
6. **Budget setting**: "Suggested resource cap per experiment: [X]. Reasonable?"
7. **Tier classification**: Experts assess feedback cycle speed, present to user:
   - Tier 1-2 (fast, seconds-to-minutes): autonomous loop
   - Tier 3 (slow, days-to-weeks): interactive loop

### 3.3 Phase 3: Skill Generation

Generate all files from templates, customized with confirmed pillars.

### 3.4 Phase 4: User Approval

Present generated skill for review. User approves or requests changes.

---

## 4. The Five Pillars

Every generated skill is defined by 5 pillars:

### 4.1 Metric

A single primary scalar with clear direction (higher-is-better or lower-is-better).

Plus optional guard metrics (护栏指标) that act as hard constraints:
```
maximize(primary_metric) subject to:
  guard_1 >= floor_1
  guard_2 >= floor_2
  ...
```

Guard metrics are selected during skill creation via expert simulation.

### 4.2 Surface

The files/artifacts the agent is allowed to modify. Everything outside the surface is frozen.

### 4.3 Budget

Fixed resource cap per experiment that makes results comparable:
- Wall-clock time (e.g., 300s for ML training)
- Or iteration count, API call count, etc.
- Includes repetition count for noisy metrics (e.g., "run 3x, take median")

### 4.4 Harness

Frozen evaluation code/commands. The agent cannot modify how success is measured.

Includes:
- The evaluation command itself
- Metric extraction logic
- Guard metric checks
- Repetition and noise-reduction strategy

### 4.5 Protocol

The rules governing the experiment loop:
- Keep/discard threshold
- Mutation strategy preferences
- Loop mode (autonomous vs interactive)
- Convergence detection parameters

---

## 5. The Experiment Loop

### 5.1 Tier 1-2: Autonomous Loop

```
loop:
  1. Check git state (current branch, latest commit)
  2. Read knowledge.md — avoid known anti-patterns, leverage proven patterns
  3. Analyze codebase, propose one improvement hypothesis
  4. Modify files within Surface
  5. Git commit
  6. Run Harness (N times if noisy metric, take median)
  7. Extract primary metric + guard metrics
  8. 3-Layer Judgment:
     - L1: Primary metric improved >= threshold?
     - L2: Compliance checks pass? (tests, lint, guards, frozen files)
     - L3: Expert review if triggered? (tacit knowledge assessment)
  9. If all pass → KEEP (commit stays, advance branch)
     If any fail → DISCARD (git reset --hard HEAD~1, revert to last known-good commit)
  10. Log to results.tsv
  11. Update knowledge.md (candidate entry)
  12. Check meta-review triggers
  13. Back to 1
```

### 5.2 Tier 3: Interactive Loop

For domains with slow feedback cycles (A/B testing, SEO ranking, etc.):

```
loop:
  1-4. Same as autonomous (propose + modify)
  5. Use proxy metric for initial screening
  6. Generate N variant proposals with proxy scores
  7. Output to user:
     "Generated N variants:
      Variant A: [description] — proxy score X
      Variant B: [description] — proxy score Y
      ...
      Please deploy/test and input real results when available."
  8. WAIT for user input (may be days/weeks)
  9. User inputs real-world results
  10. Record both proxy and real scores to results.tsv
  11. Keep best real-performing variant
  12. Meta-review can analyze proxy-vs-real correlation over time
  13. Back to 1
```

---

## 6. Three-Layer Judgment System

### 6.1 Layer 1: Metric Check (Automatic, Every Round)

```
primary_metric improved >= threshold?
all guard_metrics >= their floors?
```

For noisy metrics: run N times, take median, optionally use statistical significance test.

### 6.2 Layer 2: Compliance Check (Automatic, Every Round)

Domain-specific mechanical checks. Examples by domain:

| Domain | L2 Checks |
|--------|-----------|
| ML Training | No new dependencies, VRAM within limit, no data leakage |
| Web Performance | Visual regression test, e2e tests pass, no feature removal |
| API Latency | Response body validation, integration tests pass |
| Test Coverage | Mutation testing score, assertion density |
| Compiler | Output correctness diff, sanitizer clean |
| Database | Query result-set diff, safe config allowlist |
| Game Rendering | Visual reference comparison (SSIM/LPIPS) |

L2 checks are defined in `program.md` during skill creation, tailored to the domain.

### 6.3 Layer 3: Expert Review (Conditional, Tacit Knowledge)

Triggered when:
- Metric improvement is abnormally large (> 2x average improvement)
- Significant code deletion
- Changes near Surface boundary
- Every N keeps (periodic audit)

The expert prompt follows Michael Polanyi's tacit knowledge principle:

```
You are a senior expert in [domain]. Here is the experiment change:

[git diff]
[metric change: X -> Y]
[domain context]

Use your professional intuition to review this change.

Don't just check surface-level correctness — ask yourself:
- Is this improvement "genuine progress" or "clever evasion"?
- As a senior engineer on this project, would you let this PR merge?
- Is there anything the data isn't telling you, but your experience
  makes you uneasy about?

Your judgment can go beyond any written rule. If your intuition says
something is wrong, say so, even if you can't fully articulate why.

Verdict: APPROVE / REJECT (with reason) / FLAG (mark for meta-review)
```

Key principle: **L2 guards explicit knowledge (rules, tests, lint). L3 guards tacit knowledge (intuition, experience, pattern recognition). They complement, not overlap.**

---

## 7. Meta-Review: Self-Optimization

### 7.1 Triggers

Signal-driven, not fixed-interval:
- 5 consecutive discards (stuck in a rut)
- 3 consecutive crashes (engineering method issue)
- User manually interrupts the loop
- Every 20 experiments (periodic health check)
- Convergence detected (rolling avg improvement < threshold for 10 rounds)

### 7.2 Analysis Dimensions

1. **Hit rate analysis**: keep/total ratio
   - \> 40%: strategy too conservative, suggest more aggressive mutations
   - < 10%: strategy too aggressive or wrong direction, suggest convergence
   - 10-40%: healthy range

2. **Diminishing returns**: trend of improvement magnitude in recent keeps
   - If shrinking: approaching local optimum
   - Suggest: expand Surface, try structural changes, or declare converged

3. **Crash pattern analysis**: categorize crash causes
   - Recurring crash type: add prohibition rule to domain guidance

4. **Mutation direction retrospective**: which categories of changes have highest keep rate
   - Reinforce effective directions, de-prioritize ineffective ones

5. **Knowledge distillation**: review candidate entries in knowledge.md
   - Promote recurring patterns to formal entries
   - Extract domain heuristics from multiple entries

### 7.3 Output: Optimization Proposal

Specific proposed changes to `program.md`'s Mutable Rules section:
- Budget adjustments
- Surface expansion/contraction
- Threshold tuning
- Strategy preference updates
- Domain guidance additions

### 7.4 Safety Boundary

Meta-review **CANNOT** propose changes to:
- Metric definition (changing the metric = changing the game)
- Evaluation command (changing evaluation = cheating)
- Frozen constraints (safety boundary)

If meta-review concludes that Frozen rules need changing, it can only **report to the user** with rationale, recommending a re-run of `/create-autoresearch`.

### 7.5 User Approval Flow

```
Meta-review completes
  |
  v
Present proposals item by item
  |
  v
User approves / rejects / modifies each item
  |
  v
Update program.md Mutable Rules
  |
  v
Git commit "autoresearch: meta-review evolution #N"
  |
  v
Resume experiment loop with updated protocol
```

---

## 8. Knowledge Base System

### 8.1 Structure

Two-level knowledge system:

**Level 1: results.tsv** (raw experiment data)
```
commit | primary_metric | guard_metrics | status | description
a1b2c3 | 0.331         | vram:12.1     | keep   | Changed lr schedule to cosine warmup
d4e5f6 | 0.345         | vram:11.8     | discard| Doubled model width
```

**Level 2: knowledge.md** (extracted wisdom)

```markdown
# Knowledge Base

## Anti-Patterns

### AP-001: [title]
- Evidence: experiments #N, #M
- Failure reason: [why it failed]
- Lesson: [what to do instead]
- Generalized rule: [broader principle]

## Proven Patterns

### PP-001: [title]
- Evidence: experiments #N, #M
- Success reason: [why it worked]
- Applicable when: [conditions]
- Generalized rule: [broader principle]

## Domain Heuristics

### DH-001: [rule]
- Derived from: PP-001, AP-003
- Confidence: high/medium/low
```

### 8.2 Knowledge Lifecycle

```
Candidate entry (single experiment observation)
  | appears 2+ times
  v
Formal entry (Anti-Pattern / Proven Pattern)
  | multiple formal entries generalize
  v
Domain Heuristic (generalized rule)
  | disproven by new evidence
  v
Deprecated (not deleted — record WHY it was disproven)
```

Deprecated entries are retained with disproval reason — knowing why a rule no longer holds is itself knowledge.

### 8.3 Integration with Experiment Loop

**Pre-Mutation:** Read knowledge.md. Check candidate approach against anti-patterns. Reference proven patterns and heuristics for direction.

**Post-Experiment:** Record candidate entry (discard → candidate anti-pattern, keep → candidate proven pattern, crash → error pattern with stack trace).

**During Meta-Review:** Review all candidate entries. Promote recurring patterns to formal entries. Extract domain heuristics. Clean contradictions.

### 8.4 Knowledge Seed

`create-autoresearch` embeds a `knowledge-seed.md` with domain-general knowledge when generating the skill. This gives the loop a head start with known best practices for the domain.

The project-level `knowledge.md` starts from this seed and grows through experimentation.

### 8.5 Sharing Potential (Future)

Project-level knowledge can potentially be contributed back to domain-level seeds:
```
Project knowledge.md (project-specific)
  <-> meta-review can propose
Domain knowledge-seed.md (shared via skill distribution)
```

This is a future enhancement, not part of the initial implementation.

---

## 9. Domain Tier Classification

### 9.1 Tier Definitions

| Tier | Feedback Cycle | Loop Mode | Example Domains |
|------|---------------|-----------|-----------------|
| Tier 1 | Seconds to minutes, deterministic | Fully autonomous | ML training, compiler optimization, DB query performance |
| Tier 2 | Seconds to minutes, noisy | Autonomous with noise handling | Web performance, API latency, game FPS, test coverage |
| Tier 3 | Days to weeks, requires real-world data | Interactive, user-in-the-loop | Conversion rate, SEO ranking, A/B testing scenarios |

### 9.2 Tier Detection

During Phase 2 of `create-autoresearch`, experts assess:
- Can the metric be evaluated locally? (no → Tier 3)
- Is the metric deterministic? (no → Tier 2)
- Is the evaluation fast (< 10 min)? (no → consider Tier 2-3)

Present classification to user for confirmation.

### 9.3 Tier-Specific Adaptations

**Tier 2 additions:**
- Repetition count in Budget (run N times, take median)
- Statistical significance check in L1 (optional)
- Noise-aware keep threshold

**Tier 3 additions:**
- Proxy metric for fast screening
- Variant generation mode (propose N options)
- User input wait state
- Proxy-vs-real correlation tracking in results.tsv
- Proxy model calibration in meta-review

---

## 10. program.md Template

The core of every generated skill. Template structure:

```markdown
# AutoResearch Protocol: <domain>

## Project Context
<Domain analyst output: what the project does, business goals>

## Frozen Rules (NEVER modify)
- Primary Metric: <name> (<direction>)
- Evaluation Command: `<command>`
- Guard Metrics: <name> >= <floor>, ...
- Constraints: <files/behaviors that must not be touched>
- Repetitions: <N runs per experiment, median>

## Mutable Rules (optimizable via meta-review, with user approval)
- Budget: <resource cap per experiment>
- Surface: <mutable file patterns>
- Keep Threshold: <minimum improvement to keep>
- Mutation Strategy: <preferences and priorities>

## The Loop
<Tier-appropriate loop algorithm — see Section 5>

## Three-Layer Judgment
### L1: Metric Check
<specific metric comparison logic>

### L2: Compliance Check
<domain-specific checks — see Section 6.2>

### L3: Expert Review
<trigger conditions for this domain>
<tacit knowledge prompt customized to domain>

## Domain-Specific Guidance
<engineering methodology, common optimization directions,
known pitfalls, and domain heuristics for this specific field>

## Meta-Review Triggers
<configured trigger conditions for this domain>
```

---

## 11. Complete Lifecycle

```
/create-autoresearch
      |
      v
Phase 1: 4 experts analyze codebase in parallel
      |
      v
Phase 2: User dialogue — confirm 5 pillars + guard metrics + tier
      |
      v
Phase 3: Generate .claude/skills/autoresearch-<domain>/
      |
      v
Phase 4: User approves generated skill
      |
      v
/autoresearch-<domain>  <------------------------+
      |                                           |
      v                                           |
Pre-Mutation: consult knowledge.md                |
      |                                           |
      v                                           |
Experiment: mutate -> evaluate -> 3-layer judge   |
      |                                           |
      v                                           |
Post-Experiment: log results + knowledge entry    |
      |                                           |
      v (trigger conditions met?)                  |
Meta-Review: analyze -> propose -> user approves  |
      |                                           |
      v                                           |
Knowledge Distillation: update knowledge.md       |
      |                                           |
      v                                           |
Update program.md mutable rules ------------------+
```

---

## 12. Success Criteria

The skill is successful if:

1. A user with no prior autoresearch experience can invoke `/create-autoresearch` and have a working optimization loop within 30 minutes
2. The generated skill produces measurable improvement on the primary metric within the first 10 experiments
3. The knowledge base prevents the agent from repeating the same failed experiment
4. Meta-review produces actionable suggestions that the user finds valuable
5. The 3-layer judgment system prevents metric gaming (no "cheating" improvements get kept)
6. The skill works across Tier 1, 2, and 3 domains with appropriate adaptations

---

## 13. Out of Scope (v1)

- Cross-project knowledge sharing (future enhancement)
- Multi-objective Pareto optimization (v1 uses primary + guards)
- Distributed/multi-GPU experiment execution
- Integration with external experiment tracking (W&B, MLflow)
- Automated tier detection (v1 relies on expert recommendation + user confirmation)
