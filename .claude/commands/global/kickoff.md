---
name: kickoff
description: "Use at session start or when resuming work — before reading pipeline files or picking the next task"
---

# /kickoff — Daily Kickoff

Automated daily kickoff sequence. Scans all pipeline files, auto-cleans shipped items,
and produces a single actionable summary.
Replaces manually reading 4+ files at session start.

**Target:** Session start → first productive action in <2 minutes.

---

## Protocol

### Step −1.5 — Consult session stamp (always runs first)

```bash
source "$HOME/.claude/scripts/stamp-context.sh"
# Provides: $SID, $STAMP_FILE, $STAMP, $GOAL_STOP
echo "STAMP=${STAMP:-<none>}"
```

- `$STAMP` empty → unstamped behavior; continue Step −1 as written, including the Step 3b session-type prompt.
- `$STAMP` set → stamped behavior: this session is committed to `$STAMP`. Skip Step 3b entirely (no `execution|clearing|meta` prompt, no US-ID question). The stamp IS the commitment. Auto-write `~/.local/state/claude/session-type`:
  ```bash
  mkdir -p ~/.local/state/claude
  printf '%s\n%s\n' "execution" "$STAMP" > ~/.local/state/claude/session-type
  ```
  Then, at Step 5, override the "next action" suggestion to point at the stamp's critical-path item: read `$GOAL_STOP` if it exists and its `project` matches `$STAMP`, take `scope_uss[]` filtered by `status != "complete"` sorted by `order` ASC — first entry is the next action. If no goal-stop, suggest `/resume-handover` (which will itself consult the stamp).

  **Mission branch (spec §6, after the goal-stop check — fail-closed):** if
  `active.json` is absent or its goal complete AND a mission resolves
  (`$MISSION_FILE` non-empty, from `stamp-context.sh`) AND that mission's `status`
  is `active`, suggest `/mission next` as the next action. Any of {missing/invalid
  mission, status `blocked`/`complete`} → fall through to the existing behavior. This
  is suggest-only; never auto-drives.

### Step −1 — Validate CLAUDE_SESSION_ID (always runs first)

Before any other step, validate that `CLAUDE_SESSION_ID` is available and well-formed:

```bash
SID="${CLAUDE_SESSION_ID:-}"
UUID_RE='^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$'
# Fallback: read from persisted file written by session-id-persist.sh SessionStart hook.
# CLAUDE_SESSION_ID is injected into hook subprocesses but NOT into Bash tool subprocesses
# (a Claude Code harness gap in VS Code extension mode). The hook writes it to disk at
# session start so scripts can recover it.
if [ -z "$SID" ]; then
  PERSIST_FILE="$HOME/.local/state/claude/current-session-id"
  [ -f "$PERSIST_FILE" ] && SID="$(cat "$PERSIST_FILE" | tr -d '[:space:]')"
fi
if [ -z "$SID" ]; then
  echo "CLAUDE_SESSION_ID not set — session-id-persist.sh hook not yet fired (new session?). Proceeding without SID isolation." >&2
  SID8="unknown0"
  echo "SID_WARN=$SID8"
elif ! echo "$SID" | grep -qE "$UUID_RE"; then
  echo "CLAUDE_SESSION_ID malformed: $SID — proceeding without SID isolation." >&2
  SID8="unknown0"
  echo "SID_WARN=$SID8"
else
  SID8="${SID:0:8}"
  echo "SID_OK=$SID8"
fi
```

If SID is unavailable the kickoff continues in degraded mode (uses `TODO-Today.md` fallback, no per-session isolation). Only abort if a hard infrastructure failure prevents any pipeline reads.

### Step −0 — ICS open-incident check (US-ICS-9 AC-01/02)

Per spec `docs/specs/2026-05-27-ics-agent-governance.md` §11 ICS-9: **before any other action**, /kickoff MUST read `~/.local/state/ics/INCIDENTS.md`, print the open-incident summary, and for each open incident print the last SITREP status and prompt for read-back of the active contract. This is the session-bracket integration that lets ICS state survive across sessions through the same skill that already enforces other governance invariants.

Why before Step 0 (path discovery): ICS runtime state lives at a fixed absolute path (`~/.local/state/ics/`), independent of `$ROOT`. Running the read-back gate first satisfies the "before any other action" contract — operator acknowledges open incidents before any pipeline file is read.

```bash
INCIDENTS="$HOME/.local/state/ics/INCIDENTS.md"
if [ ! -f "$INCIDENTS" ]; then
  echo "ICS=no-runtime"     # no ICS state on this machine yet — proceed silently
else
  # Open-incident summary: each "### " heading under "## Active" is one open incident_id.
  ACTIVE_BLOCK=$(awk '/^## Active$/{flag=1; next} /^## /{flag=0} flag' "$INCIDENTS")
  OPEN_IDS=$(echo "$ACTIVE_BLOCK" | grep -E '^### [^<]' | sed 's/^### //' || true)
  OPEN_N=$(echo "$OPEN_IDS" | grep -c . || true)
  echo "ICS_OPEN=${OPEN_N:-0}"
  if [ "${OPEN_N:-0}" -gt 0 ]; then
    echo "--- Open ICS incidents ---"
    echo "$OPEN_IDS"
    # Per-incident: last SITREP status + read-back prompt for the active contract.
    while IFS= read -r INCIDENT_ID; do
      [ -n "$INCIDENT_ID" ] || continue
      SITREP_DIR="$HOME/.local/state/ics/sitreps/$INCIDENT_ID"
      LAST_SITREP=$(ls -t "$SITREP_DIR"/period-*.md 2>/dev/null | head -1)
      if [ -n "$LAST_SITREP" ]; then
        STATUS=$(grep -m1 -oE 'status:[[:space:]]*[A-Z]+' "$LAST_SITREP" | head -1 | awk '{print $2}')
        echo "  $INCIDENT_ID — last SITREP: ${LAST_SITREP##*/} — status: ${STATUS:-UNKNOWN}"
      else
        echo "  $INCIDENT_ID — last SITREP: (none yet)"
      fi
      CONTRACT="$HOME/.local/state/ics/contracts/$INCIDENT_ID.json"
      if [ -f "$CONTRACT" ]; then
        echo "  → read-back required for active contract: $CONTRACT"
      fi
    done <<< "$OPEN_IDS"
    echo "--------------------------"
  fi
fi
```

**Operator action — required before any other action when `ICS_OPEN > 0`:** for each open incident, run `/ics-readback <incident_id>` (lands in ICS-4) and acknowledge the read-back before proceeding to Step 0. Until ICS-4 ships, perform the read-back as a conversation convention: state objective + exit_criteria + escalation.to from the contract file, then proceed.

If `ICS_OPEN=0` or `ICS=no-runtime`, this step is a no-op and kickoff continues immediately.

### Step 0 — Discover pipeline root (1 bash call)

Run this single command. It checks known project locations first (fast path) and only falls back to `find` if none match. When the location changes, update `reference_central_backlog.md` in session memory.

```bash
ROOT=""
# Fast path: current dir, then known project backlogs (ordered by frequency of use)
for KNOWN in \
  "." \
  "$HOME/projects/00_Governance" \
  "$HOME/projects/20_CONSIGLIERE" \
  "$HOME/projects/30_SVG-PAINT" \
  "$HOME/projects/50_KETO" \
  "$HOME/projects/60_skillengineering" \
  "$HOME/projects/15_SAAS/20_PosterEngine" \
  "$HOME/projects/10_Sidequest" \
  "$HOME/projects/30_briefing-publisher" \
  "$HOME/projects/75_Coaching/10_Consulting/syllabus" \
  "$HOME/projects/75_Coaching/10_Consulting/MeetingProcessor"; do
  [ -f "$KNOWN/BACKLOG.md" ] && ROOT="$KNOWN" && break
done
# Fallback: filesystem scan (slow — only hits when CWD is unknown project)
if [ -z "$ROOT" ]; then
  FOUND=$(find . -maxdepth 3 -name "BACKLOG.md" 2>/dev/null | head -1)
  ROOT="${FOUND:+$(dirname "$FOUND")}"
  ROOT="${ROOT:-.}"
fi
echo "ROOT=$ROOT"
```

Store the result as `$ROOT`. All subsequent file paths use `$ROOT/filename`. If a new project path was found via the fallback, add it to the fast-path list above and update `reference_central_backlog.md` in session memory.

At the same time (parallel), load the QMD schema:
```
ToolSearch query="select:mcp__qmd__query"
```

### Step 0b — Generate TODAY-view.md (merged session queue)

After discovering ROOT, generate the daily merged view of all active session queues:

```bash
ROOT=<value from Step 0> python3 - <<'EOF'
import sys, os
sys.path.insert(0, os.path.join(os.environ.get('ROOT', '.'), 'scripts'))
from pathlib import Path
from datetime import date
try:
    from session_utils import merge_todo_files, migrate_todo_today
    root = Path(os.environ.get('ROOT', '.'))
    migrate_todo_today(root)
    content = merge_todo_files(root, date.today().isoformat())
    view = root / 'TODAY-view.md'
    view.write_text(content, encoding='utf-8')
    session_files = list(root.glob('TODO-????????-????-????-????-????????????.md'))
    print(f"TODAY_VIEW_SESSIONS={len(session_files)}")
except ImportError:
    print("TODAY_VIEW_SESSIONS=0 (session_utils not available)")
EOF
```

This file is gitignored — it is a derived read-only aggregate. Sessions never write TODAY-view.md directly.

### Step 0c — Check Dreaming FAILED sentinels

```bash
SENTINEL_DIR="$HOME/.local/state/dreaming"
# Use find, not a shell glob: zsh's nomatch error bypasses `2>/dev/null`.
FAILED_FILES=$(find "$SENTINEL_DIR" -maxdepth 1 -name 'FAILED-*' -type f 2>/dev/null)
FAILED_COUNT=$(printf '%s\n' "$FAILED_FILES" | grep -c . || true)
if [ "${FAILED_COUNT:-0}" -gt 0 ]; then
  while IFS= read -r F; do
    [ -n "$F" ] || continue
    RUN_ID=$(basename "$F" | sed 's/FAILED-//')
    echo "WARNING: Dreaming run $RUN_ID failed — review $SENTINEL_DIR/run-$RUN_ID.log" >&2
  done <<< "$FAILED_FILES"
fi
```

If any FAILED sentinels exist, add a `DREAMING FAILURES: {N} run(s) failed` line to the kickoff summary in Step 4. Do not block kickoff — this is a warning only.

### Step 0d — Refresh BACKLOG age badges

```bash
REFRESH="$ROOT/scripts/refresh_backlog_ages.py"
if [ -f "$REFRESH" ]; then
  python3 "$REFRESH" 2>/dev/null || echo "WARN: refresh_backlog_ages.py failed (non-blocking)"
fi
```

Runs silently on success. If the script is missing or fails, kickoff continues unaffected. Output (e.g. `Updated 3 age badge(s)`) is discarded — badges are refreshed in the file regardless.

### Step 0e — Stash backlog check (AC-05, US-GOV-JANITOR-ALLOWLIST-AUDIT)

Surfaces accumulated `auto-janitor-*` stashes per active repo so the operator sees the debt independent of any janitor change. Threshold: >5 stashes OR any stash older than 7 days → emit warning line.

```bash
NOW=$(date +%s)
CUTOFF=$((NOW - 7*86400))
for repo in ~/projects/00_Governance ~/projects/30_SVG-PAINT ~/projects/20_CONSIGLIERE ~/projects/50_KETO; do
  [ -d "$repo/.git" ] || [ -e "$repo/.git" ] || continue
  COUNT=$(git -C "$repo" stash list 2>/dev/null | grep -c "auto-janitor-")
  if [ "$COUNT" -gt 5 ]; then
    echo "⚠ $repo: $COUNT auto-janitor-* stashes accumulated. Review: git -C $repo stash list | grep auto-janitor- | head -10"
    continue
  fi
  OLD=$(git -C "$repo" stash list --format='%ci %gd %s' 2>/dev/null \
    | awk -v cut="$CUTOFF" '/auto-janitor-/ {
        cmd = "date -j -f \"%Y-%m-%d %H:%M:%S %z\" \"" $1 " " $2 " " $3 "\" +%s 2>/dev/null"
        cmd | getline ts; close(cmd)
        if (ts && ts < cut) print $0
      }' | wc -l | tr -d ' ')
  if [ "$OLD" -gt 0 ]; then
    echo "⚠ $repo: $OLD auto-janitor-* stash(es) older than 7 days. Review: git -C $repo stash list | grep auto-janitor-"
  fi
done
```

Silent when there's no backlog. After the janitor allowlist audit (US-GOV-JANITOR-ALLOWLIST-AUDIT) ships and the CODE→stash path is removed, this signal will only show stashes accumulated *before* the fix landed.

### Step 1 — Scan all pipeline files (1 bash call)

Run this single inline Python script — it reads all pipeline files and emits structured output in one round-trip:

```bash
python3 - <<'EOF'
import re, os
from datetime import date
from pathlib import Path

ROOT = os.environ.get('ROOT', '.')
today = date.today().isoformat()

def slurp(p):
    try: return Path(p).read_text()
    except: return ''

# INBOX: count bullets below last ---
def inbox_count(content):
    parts = content.split('---')
    if len(parts) < 2: return 0
    return sum(1 for l in parts[-1].splitlines() if l.strip().startswith('-'))

# BACKLOG: section counts, items grouped by state, struck-through
def analyze_backlog(content):
    struck_n = len(re.findall(r'^### ~~', content, re.MULTILINE))
    by_state = {'Ideation': [], 'Refining': [], 'Ready': []}
    for sec in re.split(r'^### ', content, flags=re.MULTILINE):
        us_m = re.match(r'(US-\S+)', sec)
        if not us_m:
            continue
        us = us_m.group(1).rstrip(':')
        state_m = re.search(r'State:\*\* (\w+)', sec)
        prio_m  = re.search(r'Priority:\*\* (P\d)', sec)
        state = state_m.group(1) if state_m else None
        prio  = prio_m.group(1) if prio_m else '--'
        if state in by_state:
            by_state[state].append((us, prio))
    ready_n    = len(by_state['Ready'])
    refining_n = len(by_state['Refining'])
    ideation_n = len(by_state['Ideation'])
    return ready_n, refining_n, ideation_n, struck_n, by_state

# TODO-Today: unchecked count + first unchecked task
def todo_stats(content):
    lines = content.splitlines()
    unchecked = [l for l in lines if re.match(r'- \[ \]', l)]
    checked   = [l for l in lines if re.match(r'- \[x\]', l, re.I)]
    first = unchecked[0].strip() if unchecked else 'none'
    return len(unchecked), len(checked), first

# DONE-Today: count entries with today's date
def done_today(content):
    return content.count(today)

# Autopilot
try:
    import sys as _sys
    _sys.path.insert(0, f'{ROOT}/scripts')
    from session_utils import autopilot_check as _ap_check, get_sid8 as _get_sid8
    _SID8 = _get_sid8()
    def autopilot(p):
        try:
            content = Path(p).read_text().strip()
            result = _ap_check(content, _SID8)
            if result.startswith('skip:'):
                return f'SKIPPED (owned by {result[5:]})'
            return result or 'empty'
        except Exception:
            return 'missing'
except ImportError:
    def autopilot(p):
        try: return Path(p).read_text().strip() or 'empty'
        except: return 'missing'

# MEMORY: retro counter
def retro_count(content):
    m = re.search(r'retro_stories_since_last[:\s]+(\d+)', content)
    return m.group(1) if m else '?'

# CONTEXT.md: term count
def context_terms(content):
    if not content: return 0, 0
    terms     = len(re.findall(r'^- ', content, re.MULTILINE))
    unresolved = content.count('[unresolved]')
    return terms, unresolved

# TODO Batch Classifier: group items by pattern
def classify_batches(todo_content):
    """Group unchecked TODO items into logical batches for batch-first processing."""
    batches = {
        'Debt Registration': [],
        'Debt Skill/Orchestration': [],
        'Debt Inline-Script': [],
        'Network/Graph': [],
        'Features': [],
    }
    lines = [l for l in todo_content.split('\n') if l.strip().startswith('- [')]
    for line in lines:
        if '[x]' in line or '✓' in line:
            continue
        # Classify by pattern
        if 'US-GOV-DEBT' in line and 'Registration' in line:
            batches['Debt Registration'].append(line)
        elif 'DEBT' in line and ('Skill' in line or 'orchestration' in line or 'service-call' in line):
            batches['Debt Skill/Orchestration'].append(line)
        elif 'DEBT' in line and 'inline-script' in line:
            batches['Debt Inline-Script'].append(line)
        elif 'NETWORK' in line or 'MASTODON' in line or 'GOOGLE-DEVS' in line or 'SPEAKER-NETWORK' in line:
            batches['Network/Graph'].append(line)
        else:
            batches['Features'].append(line)
    # Remove empty batches
    return {k: v for k, v in batches.items() if v}

inbox_c   = inbox_count(slurp(f'{ROOT}/INBOX.md'))
bl_r, bl_f, bl_i, struck, by_state = analyze_backlog(slurp(f'{ROOT}/BACKLOG.md'))
import os as _os
_sid = _os.environ.get('CLAUDE_SESSION_ID', '')
_todo_file = f'TODO-{_sid}.md' if _sid else 'TODO-Today.md'
unc, chk, nxt = todo_stats(slurp(f'{ROOT}/{_todo_file}'))
done_c    = done_today(slurp(f'{ROOT}/DONE-Today.md'))
ap        = autopilot(f'{ROOT}/.autopilot')
retro     = retro_count(slurp(f'{ROOT}/MEMORY.md'))
ctx_terms, ctx_unres = context_terms(slurp(f'{ROOT}/CONTEXT.md'))
batches = classify_batches(slurp(f'{ROOT}/{_todo_file}'))

print(f"INBOX={inbox_c}")
print(f"BACKLOG={bl_i}i/{bl_f}r/{bl_r}R  struck={struck}")
print(f"TODO={unc} unchecked / {chk} checked")
print(f"DONE_TODAY={done_c}")
print(f"AUTOPILOT={ap}")
print(f"RETRO={retro}")
print(f"CONTEXT_TERMS={ctx_terms}  unresolved={ctx_unres}")
print(f"NEXT={nxt}")
for batch_name, items in batches.items():
    print(f"BATCH_{batch_name.upper().replace('/', '_').replace('-', '_')}={len(items)}")
for state in ('Ideation', 'Refining', 'Ready'):
    for us, prio in by_state[state]:
        print(f"{state.upper():<10}  {prio:<3}  {us}")
EOF
```

Set `ROOT` from Step 0 before running: `ROOT=<discovered-root> python3 - <<'EOF' ...`

### Step 2 — Auto-groom backlog

Lightweight grooming pass that runs automatically every kickoff. Prevents zombie accumulation.

**2a. Archive shipped items (write):**
Scan `$ROOT/BACKLOG.md` for struck-through (`~~`) items and items marked SHIPPED/FIXED/RESOLVED/SUPERSEDED/DONE/Parked.
For each found:
- Append a 1-line summary to `$ROOT/done/BACKLOG-ARCHIVE.md` under a `### Grooming Pass {YYYY-MM-DD}` header
- Remove the item (and all its indented sub-lines) from `BACKLOG.md`
- Count removals per section

**Two archive files exist — always check both:**
- `$ROOT/BACKLOG-ARCHIVE.md` — written by the sweep pipeline (`backlog_add.py`, `rudi_heartbeat`) and manual operator deletions
- `$ROOT/done/BACKLOG-ARCHIVE.md` — written by this Step 2a grooming pass

When verifying whether a TODO-queue item is already archived (auto-drain check), grep BOTH files. An item found in either counts as archived. A check against only `done/BACKLOG-ARCHIVE.md` will produce false-negatives for everything archived via the sweep pipeline.

**2b. Duplicate scan (read-only):**
Within each section, check for items that share a title keyword match. Flag duplicates in the report — do NOT auto-merge.

**2c. Quick staleness flag (read-only):**
From the Step 1 output:
- Ideation items with a date > 14 days ago → flag as stale
- Refining items with a date > 14 days ago and no spec link → flag as stale
- Items referencing deprecated platforms → flag as platform-blocked

**2c.5. Stale-claim scan (event-log backed):**
Run `python3 ~/projects/00_Governance/scripts/backlog_add.py stale-scan` and read its
JSON `count`. This is additive to 2c (which owns Ideation/Refining date-staleness) — 2c.5
owns *claimed WIP gone quiet*. It files at most one deduped rollup US (`[stale-claim]`);
skip-if-open means re-runs never churn. Surface the count on the 2d GROOMING line.

**2d. Report:**
```
GROOMING:    {N} archived ({N} Ideation, {N} Refining, {N} Ready)
             {N} stale · {N} stale-claim · {N} duplicates · {N} platform-blocked
             (or "Clean — no action needed")
```

### Step 2f — Paperclip ↔ BACKLOG sync (write, guarded)

Only run if the script exists:

```bash
SYNC=~/projects/00_Governance/scripts/paperclip_backlog_sync.py
[ -f "$SYNC" ] && python3 "$SYNC" || echo "paperclip sync: script not found, skipping"
```

Include reconcile count in summary if non-zero.

### Step 2e + Step 3 — QMD context + cross-project priority (2 parallel calls)

Send both QMD queries in the same parallel tool-use block:

**Query A — project context:**
```
mcp__qmd__query searches=[
  {"type": "lex", "query": "spec plan design"},
  {"type": "vec", "query": "recent implementation plan or spec for current sprint"}
] collections=["<active-project-collection>"] limit=3
```

**Query B — cross-project priority:**
```
mcp__qmd__query searches=[
  {"type": "lex", "query": "Ready priority cross-project"},
  {"type": "vec", "query": "high priority cross-project backlog items that outrank local work"}
] collections=["governance"] limit=3
```

Use the project's QMD collection name (e.g., `svg-paint`, `consigliere`, `keto`, `poster-engine`, `governance`).
If the project has no QMD collection, send only Query B.

### Step 3b — Session type declaration

**If `$STAMP` was set in Step −1.5: SKIP this step entirely.** The stamp already committed the session to `execution` against `$STAMP`, and `~/.local/state/claude/session-type` was already written. Do not prompt.

**Unstamped only:** Before producing the summary, ask the user one line:

> **Session type: `execution` | `clearing` | `meta`?**

Required follow-up depending on type:
- `execution` — name the BACKLOG `US-ID` being advanced. Refuse to start an execution session without one (the failure-mode this rule targets is "executed nothing, shipped nothing").
- `clearing` — name the blocker being cleared AND the downstream task it unblocks (so /lightsout can verify the unblock happened).
- `meta` — name the artifact being built (skill, hook, governance edit, spec).

Write the declaration so /lightsout Step S can label the audit:

```bash
mkdir -p ~/.local/state/claude
{
  echo "$SESSION_TYPE"     # first line: type
  echo "$SESSION_TASK"     # second line: US-ID, blocker name, or artifact name
} > ~/.local/state/claude/session-type
```

This step is the structural commitment. Mid-session drift to a different type (e.g. starting `execution` then pivoting to skill-building) means `/lightsout` immediately and a fresh session with the new type — do not extend the current session under the wrong type.

### Step 4 — Produce summary

Print this summary (adapt counts from Step 1 output). **NEW: If batches exist, show them before QUEUE details:**

```
KICKOFF — {PROJECT} — {YYYY-MM-DD}
════════════════════════════════════════
INBOX:       {N} untriaged items
BACKLOG:     {N} Ideation · {N} Refining · {N} Ready
             Ideation:  {prio}  {US-ID}
                        {prio}  {US-ID}  (one line per item)
             Refining:  {prio}  {US-ID}
             Ready:     {prio}  {US-ID}
GROOMING:    {grooming summary from Step 2d}
BATCHES:     Batch 1 (Debt Registration): {N} items
             Batch 2 (Debt Skill/Orch): {N} items
             Batch 3 (Debt Inline-Script): {N} items
             Batch 4 (Network/Graph): {N} items
             Batch 5 (Features): {N} items
QUEUE:       {N} unchecked ({N} done today)
AUTOPILOT:   {state}
RETRO:       {N}/10 stories until next retro
CONTEXT:     {top 3 QMD results: title (file), or "No recent docs"}
DOMAIN:      {N terms in CONTEXT.md, or "No CONTEXT.md — create during next /sh:brainstorm"}
════════════════════════════════════════
```

**Batch summary rules:**
- Show BATCHES section only if 2+ batches exist (single batch = no need for batch framing)
- Order batches in priority (Debt Registration first, Features last)
- Omit zero-count batches entirely
- Omit other state blocks if they have 0 items

### Step 5 — Suggest next action

**BATCH-FIRST default:** If Step 4 shows 2+ batches, recommend processing the highest-priority batch before individual items.

Based on the state, suggest exactly ONE next action:

| Condition | Suggestion |
|-----------|-----------|
| Queue has 2+ batches | "→ **Batch-first:** Process Batch 1 ({name}, {N} items) before other work. Within batch: `/sh:execute` with autopilot enabled." |
| INBOX has items | "→ Run `/triage` to process {N} INBOX items" |
| INBOX empty + Ready items exist + Queue empty | "→ Run `/workflow` to queue {N} Ready items (#{N}: {title})" |
| Queue has unchecked items (single batch or no batching) | "→ Resume queue. Next: **{task description}**" |
| Queue has unchecked + autopilot=run | "→ Autopilot is active. Next task: **{task description}**" |
| Queue has unchecked + autopilot=pause | "→ Autopilot paused. Write `run` to `.autopilot` to resume, or pick up manually" |
| Everything empty | "→ Check ROADMAP.md for next initiative to refine" |
| Retro counter >= 10 | "→ **Retro due!** Run `/kaizen` before starting next task" |
| Governance BACKLOG has higher-priority items | "→ **Cross-project priority:** {item} outranks local queue" |
| Stale items > 5 | "→ Run `/backlog-grooming` for full staleness + orphan review" |

If multiple conditions apply, prioritize: retro due > cross-project > batches > INBOX > Queue > Ready > stale > empty.

### Step 6 — STOP

Never start implementation. The user decides what to do based on the summary.

---

## Rules

- Step 2a (archive) is the ONLY write operation — all other steps are read-only
- Step 0 (path discovery) ALWAYS runs first — never hardcode paths
- Step 1 (analysis) is ONE bash call — never split into multiple separate reads or greps
- Steps 2e + Step 3 QMD queries are sent in ONE parallel tool-use block
- Summary format is fixed — don't add prose or explanations
- Exactly ONE suggested next action — don't overwhelm with choices
- If a file doesn't exist or is empty, report 0 for that section (the script handles this)
- Archive pass is idempotent — running twice produces the same result
- Never modify active (non-struck-through) items
- Never change priorities (`#N` ordering is human-owned)
- For full staleness/orphan/dependency analysis, use `/backlog-grooming`

## Dry Run

When `--dry-run` is passed, **do not scan files or archive items**. Instead, output a synopsis:

| Action | Target | What Would Change |
|--------|--------|-------------------|
| discover | `find . -maxdepth 3 -name BACKLOG.md` | Locate pipeline root (Step 0) |
| analyze | `$ROOT/{INBOX,BACKLOG,TODO-Today,DONE-Today,MEMORY,CONTEXT}.md` | Single-script stats (Step 1) |
| archive | `BACKLOG.md` → `done/BACKLOG-ARCHIVE.md` | Remove shipped/struck-through items (Step 2a — only write operation). Auto-drain check greps BOTH `BACKLOG-ARCHIVE.md` (root, sweep pipeline) AND `done/BACKLOG-ARCHIVE.md` (grooming pass). |
| sync | `paperclip_backlog_sync.py` (if exists) | Reconcile Paperclip → BACKLOG |
| query | QMD context + cross-project (parallel) | Surface relevant specs and cross-project priorities |
| report | stdout | Summary + single next-action suggestion |

Include which files exist and estimated item counts per section.
End with confidence: **High** (all files present), **Medium** (some files missing), or **Low** (critical files missing).
