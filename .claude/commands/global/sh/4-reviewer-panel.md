---
name: sh-4-reviewer-panel
description: "4-reviewer parallel code review — code-reviewer + architecture-panel + analyze + codex — FIPD triage with auto-fix for Fix-* findings"
---

# /sh:4-reviewer-panel — Four-Reviewer Parallel Panel

Dispatches 4 reviewers in parallel against a commit range or diff, merges findings via FIPD classification, auto-applies Fix-* per governance §8, and surfaces Plan-* / Decide-* for human triage.

## Overview

**When to use:** After completing a US story or before merge to main — whenever you want adversarial coverage from multiple perspectives simultaneously.

**Why 4 reviewers:** Each has a distinct failure-mode radar:
- `code-reviewer` → implementation bugs, logic errors, security
- `sh:architecture-panel` → boundary violations, contract gaps, failure-mode blindspots
- `sh:analyze` → code smells, complexity, duplication, dead paths
- `codex exec` → style/convention drift, lint violations, naming

Running them in parallel prevents any single reviewer's blind spots from dominating the verdict.

---

## Usage

```
/sh:4-reviewer-panel [BASE_SHA] [HEAD_SHA]
/sh:4-reviewer-panel HEAD~3 HEAD
/sh:4-reviewer-panel --diff /path/to/file.patch
```

If SHAs are omitted, defaults to `HEAD~1..HEAD` (last commit).

---

## Pattern (follow exactly)

### Step 1 — Prepare diff context

```bash
BASE="${1:-HEAD~1}"
HEAD="${2:-HEAD}"
DIFF=$(git diff "$BASE" "$HEAD" --stat && git diff "$BASE" "$HEAD")
COMMIT_LOG=$(git log "$BASE..$HEAD" --oneline)
```

Produce a context block:
```
## Review Context
Commits: <COMMIT_LOG>
Files changed: <stat output>
<full diff>
```

### Step 2 — Dispatch 4 reviewers IN PARALLEL (single tool-use block)

Send all 4 in one message — never serial:

**Reviewer A — code-reviewer subagent:**
```
Agent(subagent_type="code-reviewer", prompt="""
Review this diff for bugs, security issues, and logic errors.
For each finding: FILE:LINE | SEVERITY (Critical/Major/Minor) | DESCRIPTION | FIPD-ACTION (Fix/Investigate/Plan/Decide)

<diff context>
""")
```

**Reviewer B — architecture review:**
```
Invoke sh:architecture-panel on the diff in critique mode.
Focus: boundaries, contracts, failure modes, integration seams.
End output with PANEL-VERDICT: <score>.
```

**Reviewer C — analyze:**
```
Invoke sh:analyze on the changed files.
Focus: complexity, duplication, dead code, naming, abstraction leaks.
Classify each finding as Fix / Investigate / Plan / Decide.
```

**Reviewer D — codex style check:**
```bash
# Run codex on the diff for style/convention violations
codex exec "Review this diff for style, naming, and convention issues.
Output: FILE:LINE | Minor | DESCRIPTION | Fix" -- <diff>
```

### Step 3 — Merge findings via FIPD triage table

Collect all findings from A–D. Deduplicate by file+line. Build:

| # | File:Line | Reviewer | Severity | Finding | FIPD |
|---|---|---|---|---|---|
| 1 | src/foo.py:42 | code-reviewer | Major | Off-by-one in loop | Fix |
| 2 | src/bar.py:18 | arch-panel | Minor | Missing null guard | Fix |
| 3 | src/baz.py:99 | analyze | Minor | Unused import | Fix |
| 4 | api/routes.py:55 | code-reviewer | Major | Auth bypass on edge path | Investigate |

### Step 4 — Auto-fix Fix-* findings (governance §8)

Per governance §8 panel-auto-fix policy: **all Fix-* findings are auto-applied without asking**.

For each Fix-* row:
- Apply the fix directly (Edit/Write tools)
- Mark row as ✓ Applied

Do NOT ask which Fix-* items to apply. Fix all of them.

### Step 5 — Surface Plan-* and Decide-* to user

If any Plan-* or Decide-* findings exist, surface them in a single batch:

```
## Panel Findings Requiring Human Input

The following findings need your decision before proceeding:

**Plan-* (architecture/design work needed):**
- [P1] api/routes.py:55 — Auth bypass requires redesign of the session scope model

**Decide-* (trade-off call):**
- [D1] src/foo.py:12 — Abstraction here could be premature; depends on whether X is reused

Please review and respond with which items to address this session vs. backlog.
```

### Step 6 — Emit final verdict

```
## 4-Reviewer Panel Verdict

Reviewers: code-reviewer | architecture-panel | analyze | codex
Findings: <N> total — <F> Fix (applied) | <I> Investigate | <P> Plan | <D> Decide
Auto-fixed: <F> item(s)
Pending human input: <P+D> item(s)

PANEL-VERDICT: <score>/10  (Fix items resolved: yes | Investigate/Plan/Decide: surfaced)
```

Score: 10 - (2 × Critical) - (1 × Major) - (0.5 × Minor) from unresolved findings.

---

## FIPD Triage Table

| Action | Meaning | Autopilot behavior |
|---|---|---|
| **Fix** | Clear defect, deterministic correction | Auto-apply immediately (gov §8) |
| **Investigate** | Root cause unclear, needs more context | Surface + pause if medium+ severity |
| **Plan** | Correct but requires design work | Add to backlog as P2 US |
| **Decide** | Trade-off call requiring human judgment | AskUserQuestion in next turn |

---

## Common Mistakes

- **Serial dispatch**: sending reviewers one-at-a-time doubles wall-clock time. Always parallel.
- **Asking before fixing Fix-* items**: governance §8 mandates auto-fix. Never ask.
- **Merging before dedup**: two reviewers often flag the same line. Deduplicate by file+line before building the FIPD table.
- **Mixing tooling-failure with veto**: if a reviewer fails (non-zero exit, timeout), mark it `tooling_failed` — never count it as a finding or a veto.

---

## Verification

After the panel run, confirm:
- [ ] All 4 reviewers dispatched in a single parallel tool-use block
- [ ] FIPD table built with all findings merged and deduped
- [ ] All Fix-* auto-applied without asking
- [ ] Plan-* → backlog US stubs created; Decide-* → AskUserQuestion issued
- [ ] PANEL-VERDICT line present as last output line

---

## Integration

- Called by: autopilot cleanup sub-loop (M+ effort tasks)
- Calls: `code-reviewer` subagent, `sh:architecture-panel`, `sh:analyze`, `codex exec`
- Output consumed by: quality gate Stage 3 (`parse_panel_verdict`)
- Synced via: `bash ~/projects/20_agentflow/scripts/sync-skills-global.sh`

---

## Rationalization Table (RED→GREEN iterations)

Biases caught during TDD baseline (AC-01):
| Rationalization | Counter |
|---|---|
| "The reviewer will catch this later" | All 4 run now — no deferral |
| "It's just style" | Style drift compounds; codex catches it deterministically |
| "This is probably fine" | FIPD-Fix means auto-apply, not human judgment |
| "Let me ask before fixing" | Gov §8: never ask for Fix-* |
| "One reviewer is enough" | Each has blind spots; 4-way coverage is the contract |
