---
name: code-analysis
description: "Code analysis for formal verification. Use when: (1) investigating a system implementation to identify what to model in TLA+, (2) performing bug archaeology on a codebase, (3) producing a modeling brief that guides spec generation."
---

# Code Analysis for Formal Verification

This skill guides the **Code Analysis** phase of the Specula workflow: investigating a system implementation to find bugs.

The goal is to find as many bugs as possible. Some bugs can be confirmed directly through code analysis (e.g., copy-paste errors, data races). But the most critical bugs — protocol safety violations with severe consequences — often have extremely tricky trigger paths that are impossible to confirm by manual reasoning alone. These suspected-but-unproven bugs need TLA+ model checking to verify. The **Modeling Brief** captures these findings and plans how to verify them.


## Input / Output

**Input**:
- Repository path (already cloned, with full git history)
- Reference algorithm/paper (optional, e.g., Raft paper)
- GitHub URL (optional, for issue/PR mining)

**Output**:
- `modeling-brief.md` — Standard handoff document for Spec Generation
- `analysis-report.md` — Detailed findings (optional, for full audit trail)

## Workflow Overview

```
Phase 1: Reconnaissance ──> Phase 2: Bug Archaeology ──> Phase 3: Deep Analysis ──> Phase 4: Modeling Brief
     (structure)                (history)                   (code logic)              (synthesis)
```

This is a **reference guide**, not an auto-pilot script. The user drives each phase. You provide methodology and execute on their direction.

---

## Phase 1: Reconnaissance

**Goal**: Build a structural map of the codebase.

**Steps**:
1. Identify core modules — state machine, message handling, persistence, config changes, etc.
2. Measure scale — LOC, message types, state transitions
3. Identify concurrency model — what runs on the main thread vs. separate goroutines/threads
4. Identify atomicity boundaries — what's protected by locks, what can interleave

**Key output**: A mental model of "what's where" that guides the rest of the analysis.

---

## Phase 2: Bug Archaeology

**Goal**: Map historical bugs to identify error-prone areas and recurring patterns.

**Steps**:
1. **Git history mining** — Search for bug-fix commits by keyword (fix, bug, race, panic, deadlock, correctness)
2. **Issue/PR verification** — Read full discussions, not just titles. Classify: confirmed bug / design defect / user error / false
3. **Bug Family grouping** — Group bugs by shared mechanism, not by file
4. **Reference comparison** (if applicable) — First compare with the reference algorithm/paper to find protocol-level deviations, then compare with other implementations to find implementation-level differences

**Key output**: Bug Families with evidence, ranked by severity and TLA+ modeling suitability.

**Read `references/bug-archaeology.md`** for detailed methodology.

### Bug Family Grouping

This is the most important analytical step. Instead of listing bugs flat, group them by **mechanism**:

| Grouping criterion | Example |
|-------------------|---------|
| Same root cause pattern | "Multiple code paths should behave identically but don't" |
| Same interaction pattern | "Config change + election interleaving" |
| Same architectural decision | "Independent heartbeat goroutine" |

Each Bug Family directly maps to a potential spec extension.

---

## Phase 3: Deep Analysis

**Goal**: Find new potential issues through systematic code reading, guided by Phase 2.

**Steps**:
1. **Code path inconsistency** — Find handlers that should be symmetric but aren't
2. **Non-atomic operations** — Find multi-step persistence with crash windows
3. **Missing guards** — Find checks present in some handlers but absent in similar ones
4. **Reference deviation** — Compare with the reference algorithm/paper/implementation

**Key output**: New findings, each linked to an existing Bug Family or forming a new one.

**Read `references/deep-analysis.md`** for the full analysis methodology.

### Verification of Findings

Every finding MUST be verified before inclusion. Check:
1. Re-read the exact code lines (don't rely on memory)
2. Look for compensating mechanisms elsewhere
3. Trace the full execution path
4. Check if it's an acknowledged design decision (comments, PRs, issues)

Classify each finding:

| Category | Description |
|----------|-------------|
| Confirmed bug | Definitely wrong, with evidence |
| Developer-acknowledged | Has TODO/FIXME or open issue |
| Code inconsistency | Different paths handle the same thing differently |
| Robustness issue | Missing defensive check, works normally but fragile |
| Design decision | Intentional, documented in PRs/comments |

---

## Phase 4: Modeling Brief

**Goal**: Synthesize findings into an actionable document for Spec Generation.

**Steps**:
1. Select top Bug Families by TLA+ modeling suitability
2. For each, propose concrete spec extensions (variables, actions, invariants)
3. Explicitly state what NOT to model and why
4. Classify remaining findings by verification method (model-check / test / code-review)

**Key output**: `modeling-brief.md` following the standard format.

**Read `references/modeling-brief-format.md`** for the format specification.
**See `examples/hashicorp-raft-modeling-brief.md`** for a complete real-world example.

### Bug Classification for Verification

| Method | Suitable for | Not suitable for |
|--------|-------------|-----------------|
| **TLA+ Model Checking** | Protocol logic errors, state interaction bugs, crash-recovery issues, invariant violations | Copy-paste errors, performance issues, API design, Go-specific concurrency (channels, goroutine semantics) |
| **Testing** | Data races, specific crash sequences, error handling, boundary conditions | State space explosion scenarios, subtle protocol interactions |
| **Code Review** | Copy-paste inconsistencies, metrics errors, missing defensive checks | Complex multi-step protocol issues |

---

## Handling Different System Types

- **With reference algorithm/paper** (e.g., Raft, Paxos): Find implementation deviations from the paper — historical bugs often cluster around deviations.
- **With reference implementation** (e.g., etcd-raft vs hashicorp-raft): Compare architectural decisions and check if bugs fixed in one implementation exist in another.
- **Without reference** (custom protocol): Rely on bug archaeology, concurrency analysis, and finding internal inconsistencies across code paths.

---

## Critical Rules

1. **VERIFY before reporting.** Every finding must be checked against actual code. Re-read the code. Check for compensating mechanisms. Trace the execution path. Do not report unverified suspicions.

2. **Verify important issues by reading their discussions.** Do not cite an issue as evidence based on its title alone. For any issue you plan to reference in your findings, read the comments to confirm it hasn't been debunked by maintainers or determined to be a non-issue.

3. **Do NOT hallucinate code logic.** If you're unsure what a function does, READ IT. Cite file:line for every claim about code behavior.

4. **Exclude false positives explicitly.** For each finding you exclude, explain WHY. This shows rigor and builds trust in the remaining findings.

5. **Use parallel subagents aggressively.** Analyze different files in parallel. Verify batches of issues in parallel. This is essential for covering the codebase within context limits.

6. **Evidence-based claims only.** "I think this might have a bug" is not acceptable. Show the code, the git commit, the issue discussion, or the code path inconsistency.

7. **Bug Families over flat lists.** Always group findings by mechanism. A flat list of 17 findings is hard to act on. 5 Bug Families with clear modeling implications are actionable.

---

## Reference Files

- **`references/bug-archaeology.md`** — Detailed git mining and issue verification methodology
- **`references/deep-analysis.md`** — Code analysis patterns (path inconsistency, atomicity, missing guards)
- **`references/modeling-brief-format.md`** — Standard format for the handoff document
- **`examples/hashicorp-raft-modeling-brief.md`** — Complete real-world example from the hashicorp/raft case study

## Related Skills

- **spec-generation** — Next phase: translating the modeling brief into a TLA+ specification
- **tla-trace-workflow** — Phase after spec generation: validating the spec against real traces
- **tla-spec-review** — Reviewing completed specifications for correctness
