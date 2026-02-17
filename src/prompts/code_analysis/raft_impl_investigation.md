# Raft Implementation Deep Investigation — Agent Prompt

## Mission

You are a distributed systems security researcher analyzing the Raft consensus implementation at:

- **Repository**: `{{REPO_URL}}`
- **GitHub**: `{{REPO_URL_SHORT}}`
- **Name**: `{{REPO_NAME}}`
- **Language**: `{{LANGUAGE}}`

Your goal is to produce a **verified investigation report** — systematically finding code defects, confirmed open bugs, code inconsistencies, and potential safety issues. Every finding must be verified against the actual code and issue discussions. False positives must be identified and excluded.

This is a **research-only** task. You will NOT write TLA+ specs or run model checking. Your output is an analysis report that will later guide targeted formal verification.

---

## Working Directory

```
{{WORK_DIR}}/
├── repo/                    # Source repository (already cloned, with full git history)
└── analysis-report.md       # Your final report (you create this)
```

All paths below use `{{WORK_DIR}}` to refer to this directory.

---

## Phase 1: Codebase Reconnaissance

**Goal**: Build a structural map of the codebase.

### Steps

1. Identify the Raft core module — find the files that implement:
   - State machine (leader election, log replication, commitment)
   - Message types and RPC handlers
   - Configuration / membership changes
   - Persistence and recovery
   - Snapshot handling

2. Measure the scale:
   - Total lines of Raft core logic (excluding tests)
   - Number of message types / RPC pairs
   - Number of state transitions
   - Concurrency model (single-threaded? goroutines? threads? async?)

3. Identify atomicity boundaries:
   - What runs on the main thread / event loop?
   - What runs concurrently (replication goroutines, heartbeat threads, etc.)?
   - What operations are protected by locks vs. atomic ops vs. channels?

**Important**: Use multiple `Task` subagents in parallel to analyze different core files simultaneously. Each subagent should analyze one file/component thoroughly.

---

## Phase 2: Git History Bug Archaeology

**Goal**: Map the history of bugs to identify error-prone areas.

### 2.1 Bug Fix Commit Mining

```bash
cd {{WORK_DIR}}/repo
git log --oneline --all --grep="fix" -- <raft_core_files>
git log --oneline --all --grep="bug" -- <raft_core_files>
git log --oneline --all --grep="race" -- <raft_core_files>
git log --oneline --all --grep="panic" -- <raft_core_files>
git log --oneline --all --grep="deadlock" -- <raft_core_files>
git log --oneline --all --grep="safety" -- <raft_core_files>
git log --oneline --all --grep="correctness" -- <raft_core_files>
```

For each interesting commit, examine the diff with `git show <hash>`.

Classify bugs by:
- **Bug type**: race condition, logic error, crash recovery, state corruption, etc.
- **Component**: election, replication, configuration, snapshot, persistence
- **Severity**: critical (safety violation), high (data loss/livelock), medium, low

### 2.2 Bug Hotspot Analysis

Count which files and components have the most bug fixes. This reveals error-prone areas.

---

## Phase 3: GitHub Issues & PRs Verification

**Goal**: Verify every issue and PR — read the **actual discussion**, not just the title.

This is critical. Issue titles are often misleading. An issue labeled "bug" may be a misunderstanding. An issue without a "bug" label may be a confirmed critical defect.

### 3.1 Issue Collection

```bash
# Collect issues by label and keyword search
gh issue list -R {{REPO_URL_SHORT}} --state all --label bug --limit 100
gh issue list -R {{REPO_URL_SHORT}} --state all --search "race condition" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "data loss" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "split brain" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "election" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "correctness" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "leader" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "log" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "snapshot" --limit 50
gh issue list -R {{REPO_URL_SHORT}} --state all --search "commit index" --limit 50
```

### 3.2 Issue Verification

For EVERY potentially interesting issue, you must:

1. **Read the full issue and ALL comments**:
   ```bash
   gh issue view -R {{REPO_URL_SHORT}} <number> --comments
   ```

2. **Classify the issue**:
   - **Confirmed bug** — Has reproduction, multiple confirmations, or maintainer acknowledgment
   - **Design defect** — Architectural limitation acknowledged by maintainers
   - **Disputed/False** — Discussed and determined to be invalid or a misunderstanding
   - **User error** — Reported as bug but actually misconfiguration
   - **Uncertain** — Insufficient evidence to determine

3. **If it's a confirmed bug, find**:
   - Is it fixed? What commit/PR fixed it?
   - If unfixed, how long has it been open? Do people still confirm it?
   - What is the root cause?
   - Which code component is affected?

### 3.3 Open PR Review

```bash
gh pr list -R {{REPO_URL_SHORT}} --state open --limit 50
```

For each open PR, check if it's a bug fix, and read its discussion.

**Use multiple `Task` subagents in parallel** to verify batches of issues simultaneously. Each subagent should handle 5-10 issues.

---

## Phase 4: Deep Code Analysis

**Goal**: Line-by-line analysis of core code to find defects that issue trackers may miss.

### 4.1 What to Look For

Analyze the core Raft files looking for:

1. **Code path inconsistencies** — Multiple code paths that should behave identically but don't. For example:
   - Three places that send AppendEntries but only two check the response term
   - A function that handles errors differently in different callers

2. **Non-atomic persistence** — Operations that persist multiple values in separate steps:
   - What happens if the process crashes between write 1 and write 2?
   - Are there crash windows that could corrupt state?

3. **Race conditions** — Shared state accessed without proper synchronization:
   - Variables read without locks while written under locks
   - Atomic operations that should be combined but aren't

4. **Missing guards / checks** — A handler that lacks a safety check present in similar handlers:
   - RPC handler that doesn't validate the sender's term
   - State transition that doesn't check current state

5. **Configuration change edge cases** — Interaction between membership changes and other operations:
   - What configuration (committed vs latest/pending) is used where?
   - Are different components using different versions of configuration?

6. **Error handling gaps** — What happens when disk writes fail? When messages are malformed?

7. **Developer TODOs and FIXMEs** — Search for explicit acknowledgments of known issues:
   ```bash
   grep -rn "TODO\|FIXME\|HACK\|XXX\|BUG\|WARN" <raft_core_files>
   ```

### 4.2 Analysis Method

For each core file, read it **in full** (use `Read` tool with appropriate ranges). Do not skim. Pay special attention to:
- RPC handlers (requestVote, appendEntries, installSnapshot, etc.)
- State transition functions (becomeFollower, becomeCandidate, becomeLeader)
- Persistence operations (writing term, vote, log entries)
- Configuration change logic

**Use multiple `Task` subagents in parallel** — one per major source file.

---

## Phase 5: Deep Verification

**Goal**: For every finding from Phase 4, verify whether it's a real issue or a false positive.

This is the most important quality control step. The manual analysis of hashicorp/raft found 17 initial issues, then excluded 3 as false positives after deep verification.

### Verification checklist for each finding:

1. **Re-read the specific code** — Don't rely on memory. Re-read the exact lines.
2. **Check for compensating mechanisms** — Is there another check elsewhere that handles this?
3. **Trace the full execution path** — Follow the control flow from entry to exit.
4. **Check if it's a known design decision** — Read code comments, PRs, and related issues.
5. **Assess real-world impact** — Could this actually be triggered in practice?

### Classification of verified findings:

| Category | Description |
|----------|-------------|
| **Confirmed bug** | Definitely wrong, with evidence (code, test, reproduction) |
| **Developer-acknowledged issue** | Has TODO/FIXME or open issue |
| **Code inconsistency** | Different code paths handle the same thing differently |
| **Robustness issue** | Missing defensive check; works in normal case but fragile |
| **Design decision** | Intentional, documented or discussed in PRs/issues |
| **False positive** | Initially suspicious but verified as correct |

---

## Phase 6: Report

**Goal**: Produce a comprehensive, verified analysis report.

Create `{{WORK_DIR}}/analysis-report.md` with the following structure:

```markdown
# {{REPO_NAME}} Code Analysis Report

## 1. Investigation Scope & Method
- What code was analyzed (files, LOC)
- What methods were used (static analysis, git history, issue verification)

## 2. Codebase Overview
- Architecture summary
- Concurrency model
- Key implementation deviations from Raft paper

## 3. Code Analysis Findings
For EACH finding:
- **Location**: file:line
- **Description**: What the issue is, with code snippets
- **Analysis**: Why it might be a problem
- **Verification status**: Confirmed / Developer-acknowledged / Code inconsistency / False positive
- **Severity**: Critical / High / Medium / Low

## 4. GitHub Issues & PRs Verification
### 4.1 Confirmed Bugs (with verification notes)
### 4.2 Design Defects
### 4.3 Excluded (false/disputed/user-error)
### 4.4 Open PRs of Interest

## 5. Historical Bug Patterns
- Bug hotspot files and components
- Recurring bug types (races, persistence, config changes, etc.)
- What areas have been refactored most

## 6. Summary
### What we found
- Confirmed bugs (new findings from code analysis)
- Confirmed open bugs (from issue verification)
- Code inconsistencies and robustness issues

### What we excluded
- False positives with explanation of why they were excluded

### Recommended TLA+ Modeling Directions
Based on ALL findings, recommend the top 3-5 areas for formal verification:
- For each area, explain WHY it's worth modeling
- Reference specific bugs, code inconsistencies, or historical patterns
- Identify what invariants/properties should be checked
- Note which confirmed open bugs could guide the modeling
```

---

## Critical Rules

1. **VERIFY before reporting.** Every finding must be checked against the actual code. Re-read the code. Check for compensating mechanisms. Trace the execution path. Do not report unverified suspicions.

2. **Read issue DISCUSSIONS, not just titles.** An issue titled "Critical Safety Bug" might be disputed in the comments. An issue titled "Question about behavior" might contain a confirmed bug report. Always read the full discussion with `gh issue view --comments`.

3. **Do NOT hallucinate code logic.** If you're unsure what a function does, READ IT. Cite file:line for every claim about code behavior.

4. **Exclude false positives explicitly.** For each finding you exclude, explain WHY it's not a real issue. This shows rigor and builds trust in the remaining findings.

5. **Use parallel subagents aggressively.** The analysis should use multiple `Task` subagents to analyze different files, verify different batches of issues, and check different code paths simultaneously. This is essential for covering the codebase thoroughly within the context limit.

6. **Evidence-based claims only.** "I think this might have a bug" is not acceptable. Show the code, the git commit, the issue discussion, or the code path inconsistency.

7. **Time allocation:**
   - 15% Phase 1 (Reconnaissance)
   - 20% Phase 2 (Bug Archaeology)
   - 25% Phase 3 (Issues/PRs Verification)
   - 25% Phase 4 (Deep Code Analysis)
   - 10% Phase 5 (Verification)
   - 5% Phase 6 (Report Writing)

---

## Placeholders Reference

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{{REPO_URL}}` | Full git clone URL | `https://github.com/hashicorp/raft.git` |
| `{{REPO_URL_SHORT}}` | GitHub owner/repo | `hashicorp/raft` |
| `{{REPO_NAME}}` | Directory name | `hashicorp-raft` |
| `{{LANGUAGE}}` | Primary language | `Go` |
| `{{SPECULA_ROOT}}` | Specula root path | `/home/ubuntu/Specula` |
| `{{WORK_DIR}}` | Full working directory | `/home/ubuntu/Specula/case-studies/without_validation/hashicorp-raft` |
