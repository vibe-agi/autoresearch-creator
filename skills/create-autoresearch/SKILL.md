---
name: create-autoresearch
description: Use when user wants to create an autonomous optimization loop for their project. Analyzes codebase with expert agents, defines optimization pillars, generates a self-improving experiment skill.
argument-hint: domain hint or "upgrade"
---

# create-autoresearch

Generate a project-specific autonomous optimization skill for this codebase.

Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch): any optimization problem can be expressed as iterative hill-climbing over a mutable code surface, evaluated against a frozen metric, with a ratchet mechanism that only advances on improvement.

## Core Formula

```
Optimization = State(code) x Metric(scalar) x Mutation(change) x Evaluation(run) x Selection(keep/discard)
```

## Two Modes

This skill has two invocation modes:

1. **Fresh generation**: `/create-autoresearch` or `/create-autoresearch <domain-hint>` — runs the full 4-phase flow
2. **Upgrade**: `/create-autoresearch upgrade` — re-renders existing generated skill from its pillars.json using the current templates

If `$ARGUMENTS` is exactly the string `upgrade`, go to the Upgrade Flow section. Otherwise, run Fresh Generation.

---

# Fresh Generation (4 Phases)

## Phase 1: Project Analysis

Read `expert-analysis.md` for the full expert prompts and dispatch instructions.

Dispatch 4 agents in parallel using the Agent tool (single message, 4 tool calls). Use `subagent_type: Explore` for the first three, `general-purpose` for Domain Analyst.

| Expert | Focus |
|--------|-------|
| **Architecture Analyst** | Project structure, languages, frameworks, modules, dependencies |
| **Metric Scout** | Existing tests, benchmarks, metrics, CI evaluation, monitoring |
| **Runtime Analyst** | Build/test/run commands, environment, how to launch and evaluate, budget unit recommendation |
| **Domain Analyst** | README, docs, comments, business goals, defensive code patterns |

Wait for all 4 to complete. Follow the Synthesis section of `expert-analysis.md` (explicit conflict resolution rules) to produce a unified project understanding.

If `$ARGUMENTS` contains a domain hint (e.g., "web-performance"), pass it to all experts to focus their analysis.

## Phase 2: Pillar Dialogue

Read `pillar-dialogue.md` for the full dialogue protocol.

Walk the user through confirming each pillar **one at a time**. Do NOT batch questions. Communicate in the user's language (detect from messages). Technical terms (metric names, commands, file paths) stay in English.

**9 steps (order matters — each builds on the previous):**

1. **Domain Understanding** — Confirm what the project does and its core goal
2. **Metric Selection** — Present discovered metrics, user picks primary + direction
3. **Guard Metrics** — Simulate "what could go wrong without this guard", user selects
4. **Surface Definition** — Propose mutable file scope, user adjusts
5. **Protected Patterns Confirmation** — Display Defensive Code Inventory, user confirms/removes/modifies
6. **Harness Confirmation** — Confirm evaluation command and extraction
7. **Budget Setting** — Propose budget with domain-appropriate unit (not always seconds)
8. **Keep Threshold** — Minimum improvement to keep an experiment
9. **Tier Classification** — Tier 1 (deterministic) or Tier 2 (noisy); Tier 3 unsupported

After all 9 steps, summarize the complete configuration and get final approval before Phase 3.

## Phase 3: Skill Generation

Read `skill-generation.md` for the full templates and rendering rules. Pay special attention to the "Rendering Rules (Step-by-Step)" section and the worked pillars.json example.

### Step 1: Build the in-memory pillars object

Assemble all decisions from Phase 2 into a single pillars object matching the schema in skill-generation.md Template 0:

```
pillars = {
  schema_version: "1.0",
  generator_version: <current autoresearch-creator version>,
  domain: <domain slug from user>,
  created_at: <now in ISO 8601>,
  last_updated_at: <same as created_at>,
  project_context: <confirmed text from Step 1>,
  tier: <1 or 2 from Step 9>,
  frozen: {
    primary_metric: <from Step 2>,
    guard_metrics: <from Step 3>,
    harness: <from Step 6>,
    frozen_files: <from Step 4>,
    protected_patterns: <from Step 5>,
    prohibited_actions: <domain-specific, from Domain Analyst>
  },
  mutable: {
    budget: <from Step 7>,
    surface_patterns: <from Step 4>,
    keep_threshold: <from Step 8>,
    mutation_strategy_notes: <from Domain Analyst>,
    meta_review_triggers: <use defaults unless user overrode>
  },
  meta_review_history: []
}
```

### Step 2: Write pillars.json

Serialize the pillars object and write to `autoresearch/pillars.json`. Use 2-space indentation for readability. This file is the source of truth — every subsequent file is rendered from it.

### Step 3: Render the generated .md files

In order, render each template from skill-generation.md using the pillars object:

1. `.claude/skills/autoresearch-<domain>/SKILL.md` ← Template 1
2. `.claude/skills/autoresearch-<domain>/program.md` ← Template 2
3. `.claude/skills/autoresearch-<domain>/meta-review.md` ← Template 3
4. `.claude/skills/autoresearch-<domain>/knowledge-seed.md` ← Template 4 (generic seed + domain-specific entries from Domain Analyst output)

Follow the rendering rules in skill-generation.md section "Rendering Rules (Step-by-Step)":
- Substitute `{{field.path}}` with literal values from pillars.json
- Iterate `{{#each array}}` blocks for list fields
- Render tier-conditional blocks based on `pillars.tier`
- Skip optional sections whose fields are null/empty
- Do NOT paraphrase or simplify templates — substitute values only

### Step 4: Validate rendered output

For each rendered .md file, run these checks:

```
grep -n '{{' <file>     → must be empty
grep -n '<[a-z_]' <file> → must be empty (legacy angle-bracket placeholders)
```

If either grep returns matches:
- Identify which field is missing from pillars.json
- Go back to Step 1 and fix the gap, OR go back to Phase 2 and collect the missing field from the user
- Do NOT ship a skill with unfilled placeholders

### Step 5: Initialize state files and .gitignore

```
autoresearch/results.tsv  ← Write just the 5-column header row, no data
autoresearch/knowledge.md ← Copy knowledge-seed.md content with a header
```

See Template 5 and Template 6 in skill-generation.md for exact content.

**CRITICAL: `.gitignore` handling.** The following files MUST be gitignored to survive `git reset --hard` during the experiment loop:

```
run.log                         # eval output, recreated each experiment
autoresearch/results.tsv        # append-only experiment log
autoresearch/knowledge.md       # append-only knowledge base
```

These files are **intentionally not committed** — if they were, `git reset --hard HEAD~1` on discard would erase log entries for the discarded experiment. This design follows karpathy/autoresearch's original approach.

Only `autoresearch/pillars.json` is committed (it's the configuration, not the log).

**Check if `.gitignore` exists in the project root:**
- If it exists, append any missing entries from the list above (don't duplicate existing entries).
- If it does not exist, create it with:
  ```
  # autoresearch runtime outputs
  run.log

  # autoresearch accumulating logs (must survive git reset)
  autoresearch/results.tsv
  autoresearch/knowledge.md
  ```

### Step 6: Do NOT create git branches

Branch creation happens at the generated skill's FIRST RUN, not during factory generation. Phase 3 leaves the user on their current branch.

## Phase 4: User Approval and Commit

Present the generated files to the user for review:

```
Generated autoresearch skill for: <domain>

Files created:
  autoresearch/pillars.json                                 — Source of truth
  autoresearch/results.tsv                                  — Experiment log
  autoresearch/knowledge.md                                 — Knowledge base
  .claude/skills/autoresearch-<domain>/SKILL.md             — Loop entry (invoke with /autoresearch-<domain>)
  .claude/skills/autoresearch-<domain>/program.md           — Experiment protocol
  .claude/skills/autoresearch-<domain>/meta-review.md       — Self-optimization protocol
  .claude/skills/autoresearch-<domain>/knowledge-seed.md    — Initial domain knowledge

Configuration (from pillars.json):
  Tier: <1|2>
  Primary Metric: <name> (<direction>)
  Guards: <list>
  Surface: <summary>
  Budget: <value> <unit>
  Keep Threshold: <value> <unit>
  Protected Patterns: <N> entries

Review the files. When you're ready, I'll commit them.
```

If the user requests changes, **modify pillars.json and re-render** — never edit the generated .md files directly. Then re-present.

**Git safety check before commit:**

```
1. Run: git status --porcelain
2. Categorize the output:
   - Files we just generated (expected): .claude/skills/autoresearch-<domain>/*, autoresearch/*
   - Any other modified/untracked files: UNRELATED to this work
3. If there are UNRELATED files:
     Tell user:
       "I notice you have other uncommitted changes:
        [list of unrelated files]

        I will ONLY stage the files I just generated, not your other changes.
        Should I commit the generated files now?"
4. If no unrelated files:
     Tell user: "The generated files are ready. Should I commit them?"
5. Wait for explicit confirmation ("yes" / "commit" / 等)
```

Only after explicit confirmation, commit. **Do NOT stage `autoresearch/results.tsv` or `autoresearch/knowledge.md`** — these are gitignored (see Phase 3 Step 5):

```bash
git add .claude/skills/autoresearch-<domain>/ \
        autoresearch/pillars.json \
        .gitignore
git commit -m "feat: generate autoresearch skill for <domain>"
```

After commit, tell the user: "Skill generated. Invoke `/autoresearch-<domain>` to start optimizing."

---

# Upgrade Flow (`/create-autoresearch upgrade`)

Invoked as: `/create-autoresearch upgrade`

## Step 1: Locate the existing skill

Check the user's project for `autoresearch/pillars.json`:

```
1. If autoresearch/pillars.json does not exist:
     Check if .claude/skills/autoresearch-*/ exists:
       
       a. If no such directory exists:
            Tell user: "No autoresearch skill found in this project.
                        Run /create-autoresearch (without 'upgrade') to create one."
            Exit.
       
       b. If a legacy (pre-v0.5.0) skill directory exists:
            Tell user: "Found a legacy autoresearch skill that was generated
                        before v0.5.0. Legacy skills cannot be automatically
                        upgraded because they lack pillars.json.
                        
                        To migrate:
                        1. Back up your experiment data:
                           cp autoresearch/results.tsv autoresearch/results.tsv.bak
                           cp autoresearch/knowledge.md autoresearch/knowledge.md.bak
                        2. Delete the legacy skill:
                           rm -rf .claude/skills/autoresearch-<domain>/
                        3. Run: /create-autoresearch <domain>
                           (you'll need to re-answer the pillar dialogue —
                            your original decisions should still be fresh)
                        4. Restore your experiment data:
                           cp autoresearch/results.tsv.bak autoresearch/results.tsv
                           cp autoresearch/knowledge.md.bak autoresearch/knowledge.md
                        
                        Future upgrades (v0.5.0 onward) will be automatic."
            Exit.

2. If there are multiple .claude/skills/autoresearch-*/ directories:
     Ask the user: "Multiple autoresearch skills found: [list].
                    Which one to upgrade?"
```

## Step 2: Read and validate pillars.json

```
1. Load autoresearch/pillars.json
2. Check pillars.schema_version — if incompatible with this autoresearch-creator version:
     Report the incompatibility and exit with manual migration instructions.
3. Read pillars.generator_version — this is the version that last rendered the .md files.
4. Compare with the current autoresearch-creator version.
5. If same: "Already up to date (version X.Y.Z)." Exit.
6. If different: proceed to Step 3.
```

## Step 3: Re-render from current templates

```
1. Read the current templates from skill-generation.md.
2. Re-render SKILL.md, program.md, meta-review.md, knowledge-seed.md
   from pillars.json + current templates.
3. Compute diff against existing files.
4. Update pillars.json:
     - generator_version = current autoresearch-creator version
     - last_updated_at = now
```

## Step 4: Show diff and ask approval

```
Tell user:

  Upgrading autoresearch-<domain> from vX.Y.Z to vA.B.C.

  Changes:
    SKILL.md: <N lines changed> — <summary of what's new>
    program.md: <N lines changed> — <summary>
    meta-review.md: <N lines changed> — <summary>
    pillars.json: generator_version updated

  Preserved (untouched):
    autoresearch/results.tsv
    autoresearch/knowledge.md
    pillars.json.frozen.*
    pillars.json.mutable.*
    pillars.json.meta_review_history

  Apply the upgrade?
```

Wait for explicit user confirmation.

## Step 5: Git safety check + commit

Same git safety pattern as Fresh Generation Phase 4:

```
1. git status --porcelain
2. If unrelated changes exist, warn user and confirm stage-only-our-files intent.
3. Stage explicitly:
     git add autoresearch/pillars.json \
             .claude/skills/autoresearch-<domain>/SKILL.md \
             .claude/skills/autoresearch-<domain>/program.md \
             .claude/skills/autoresearch-<domain>/meta-review.md \
             .claude/skills/autoresearch-<domain>/knowledge-seed.md
4. Commit:
     git commit -m "chore: upgrade autoresearch-<domain> to vA.B.C"
```

---

# Important Rules

- **ALWAYS communicate in the user's language.** Detect the language from the user's messages and use it throughout all phases. Technical terms (metric names, commands, file paths) stay in English; explanations, questions, summaries, and status updates use the user's language.
- **NEVER skip Phase 2 (pillar dialogue).** Even if the domain seems obvious, the user must confirm each pillar.
- **NEVER generate a skill with unfilled placeholder values.** Every `{{...}}` or `<...>` in templates must be replaced with a concrete value from pillars.json.
- **NEVER manually edit generated .md files.** All changes flow through pillars.json.
- **NEVER auto-commit to a dirty branch without explicit user permission.** Git state is user state.
- **NEVER create git branches in Phase 3.** Branch creation is the generated skill's responsibility at its first run.
- **The Metric and Evaluation Command in Frozen Rules are the user's decision.** Suggest, but do not override.
- If the user's project has no obvious quantifiable metric, help them define one. Every domain has something measurable.
- **Tier 3 is unsupported in v0.5.0.** If the domain is inherently slow-feedback (A/B tests, SEO rankings), tell the user and offer to define a proxy metric or abort.
