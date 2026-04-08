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
6. Defensive code patterns — identify code that exists for non-functional reasons (concurrency safety, security, compliance, fault tolerance) that must NOT be removed even if it looks "expensive" by the metric. This is CRITICAL: scan the mutable Surface files for cloning/copying of shared data, locks/mutexes, input validation, error handling, rate limiting, checksums, and similar protective patterns. These are load-bearing even when the metric says they are dead weight.

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

## Defensive Code Inventory
For each defensive pattern found in the mutable Surface files:
- **File:line:** [location]
- **Pattern:** [what it does — e.g., "clones http.Header before passing to goroutine"]
- **Protects:** [what non-functional property — e.g., "concurrency safety"]
- **Risk if removed:** [what breaks — e.g., "data race under concurrent requests"]

This inventory will be injected into the generated program.md as protected
code that the agent must not remove for metric gains.

## Engineering Methodology
- **Recommended approach:** [how to iterate in this domain]
- **Single-variable principle:** [how to isolate changes in this domain]
- **Domain heuristics:** [rules of thumb from this field]
- **Safety verification commands:** [e.g., `go test -race`, `valgrind`, `npm audit`, linters with safety rules]

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
