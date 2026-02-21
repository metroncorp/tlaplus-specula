# Code Analysis for Formal Verification

Investigate a system implementation to find bugs. Some bugs can be confirmed directly (copy-paste errors, data races). The most critical bugs — protocol safety violations — need TLA+ model checking to verify. The **Modeling Brief** captures findings and plans how to verify them.

## Input / Output

**Input**: Repository path + reference algorithm/paper (optional) + GitHub URL (optional)

**Output**: `modeling-brief.md` (handoff to Spec Generation) + `analysis-report.md` (optional audit trail)

**The user drives each phase.** You provide methodology and execute on their direction.

---

## Phase 1: Reconnaissance

**Goal**: Build a structural map — core modules, scale, concurrency model, atomicity boundaries.

---

## Phase 2: Bug Archaeology

**Goal**: Map historical bugs to identify error-prone areas and recurring patterns.

1. Git history mining by keyword (fix, bug, race, panic, deadlock, correctness)
2. Issue/PR verification — read full discussions, not just titles
3. Bug Family grouping — group by shared **mechanism**, not by file
4. Reference comparison (if applicable) — find deviations from paper/other implementations

**Read `references/bug-archaeology.md`** for detailed methodology.

---

## Phase 3: Deep Analysis

**Goal**: Find new issues through systematic code reading, guided by Phase 2 — code path inconsistencies, non-atomic operations, missing guards, reference deviations.

Every finding MUST be verified: re-read exact code lines, check for compensating mechanisms, trace execution path, check if it's an acknowledged design decision.

**Read `references/deep-analysis.md`** for the full methodology.

---

## Phase 4: Modeling Brief

**Goal**: Synthesize findings into an actionable document for Spec Generation — select top Bug Families, propose spec extensions, state what NOT to model and why, classify remaining findings by verification method.

**Read `references/modeling-brief-format.md`** for the format specification.
**See `examples/hashicorp-raft-modeling-brief.md`** for a complete example.

---

## Critical Rules

1. **VERIFY before reporting.** Re-read the code. Check for compensating mechanisms. No unverified suspicions.
2. **Read issue discussions, not just titles.** Confirm issues haven't been debunked by maintainers.
3. **Do NOT hallucinate code logic.** If unsure, READ IT. Cite `file:line` for every claim.
4. **Exclude false positives explicitly.** Explain WHY each exclusion was made.
5. **Use parallel subagents aggressively.** Essential for covering the codebase within context limits.
6. **Evidence-based claims only.** Show code, git commits, issue discussions, or code path inconsistencies.
7. **Bug Families over flat lists.** Group by mechanism. 5 actionable families > 17 flat findings.

---

## Reference Files

- **`references/bug-archaeology.md`** — Git mining and issue verification methodology
- **`references/deep-analysis.md`** — Code analysis patterns (path inconsistency, atomicity, missing guards)
- **`references/modeling-brief-format.md`** — Standard format for the handoff document
- **`examples/hashicorp-raft-modeling-brief.md`** — Complete real-world example

## Related Skills

- **spec-generation** — Next phase: produces TLA+ specs from the Modeling Brief
- **tla-trace-workflow** — Validates the trace spec against real traces
