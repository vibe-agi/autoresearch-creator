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
