---
name: lightsout
description: "Use at end of any session, after every major deliverable (PR, panel review, batch completion), or before pivoting to a new logical task"
---

# /lightsout — End-of-Session Wrap-Up

End-of-session pipeline. Default mode is **checkpoint** (fast: promote + handover + attribution). Use `--full` for publishing, overnight DAGs, and detailed summaries.

## Modes

| Mode | Steps | When to use |
|------|-------|-------------|
| `/lightsout` (default) | 0, 1-lite, W, M, 5, ⏚, D, 5b, 6, 7, 8 | After any session — cheap, target <3 min |
| `/lightsout --full` | 0, 1, 1b, 1c, 2, 2b, 3, 4, W, W2, M, S, R, 5, ⏚, D, 5b, 6, 7, 8 | End-of-day or after major deliverables |
| `/lightsout --dry-run` | Show what would happen, no writes | Debugging |
| `/lightsout [date]` | Full pipeline for a specific date | Retroactive |

**Cost discipline.** Default mode dropped W2/S/R/1c/2b after a 15-min wrap-up revealed checkpoint had accreted ceremony. Hourly launchd covers W2; per-commit hooks cover 2b; R/S/1c run end-of-day in `--full`. Target default cost: <3 min, ≤6 tool calls.

---

## Step −3 — Consult session stamp (always runs first)

```bash
source "$HOME/.claude/scripts/stamp-context.sh"
# Provides: $SID, $STAMP_FILE, $STAMP, $GOAL_STOP
```

`$STAMP` is consulted by Step 5 (handover filename + funnel-empty check) and Step 8 (clipboard hint). All other steps are stamp-agnostic — the lightsout pipeline runs identically. When `$STAMP` is set:

- **Step 5 PROJECT derivation:** override the "primary project from session work" heuristic — filename becomes `HANDOVER-${STAMP}-{YYYY-MM-DD-HHmm}.md` directly.
- **Funnel-empty signal:** after writing the handover, check whether the stamp's funnel is empty (see resolution order in `~/.claude/commands/stamp.md`). If empty, append a final section to the handover:
  ```markdown
  ## Stamp Funnel
  Empty — `/lightsout --full` recommended for this stamp before pivoting.
  ```
  This is the signal `resume-handover` reads on the next session to recognize "this topic is done."

---

## Step −2 — Acquire Lightsout Lock (always runs first)

Only one `/lightsout` should write to shared governance files (`KNOWN_PATTERNS.md`, `BACKLOG.md`, `pending.jsonl`, `HANDOVER-*.md`) at a time. This step blocks until any sibling lightsout exits, with a 5-minute timeout (TTL-aligned — past 5 min the lock is stale by definition and reclaimed). Skipped on `--dry-run`.

**IMPORTANT:** Invoke the bash block below with `timeout: 360000` (6 min — covers the 5-min wait plus reclaim overhead). The wait was reduced from 9 min after a 2026-05-16 orphan-lock incident; TTL=5 min means a stale lock is guaranteed reclaimable within 5 min.

The lock is a sentinel file (not `flock(1)`) because kernel locks die with each bash process and Claude skills span many bash calls. Sentinel-as-state with mtime TTL gives self-healing across the full skill duration.

```bash
mkdir -p ~/.local/state/lightsout
LOCK=~/.local/state/lightsout/active.session
SESSION_ID="${CLAUDE_SESSION_ID:-$(date +%s)-$$}"
TTL_MIN=5            # stale-lock auto-release window (was 30 — orphans now self-heal fast)
WAIT_SEC=300         # 5 min — past TTL, sibling lock is reclaimed anyway
POLL_SEC=5           # tighter polling (was 10) — sibling exit picked up faster

# Trap: if this very block exits non-cleanly between acquire and "echo OK",
# release the lock so the next /lightsout doesn't see a partial-acquire orphan.
# Only protects this block; later blocks rely on the 5-min TTL.
trap 'if [ "$(cat "$LOCK" 2>/dev/null)" = "$SESSION_ID" ] && [ "${LOCK_CONFIRMED:-0}" = "0" ]; then rm -f "$LOCK"; fi' EXIT INT TERM

# --dry-run skips the lock entirely (read-only, no contention)
if [ "${LIGHTSOUT_DRY_RUN:-0}" = "1" ]; then
  echo "LIGHTSOUT_LOCK_OK session=$SESSION_ID waited=0s mode=dry-run"
  exit 0
fi

is_stale() {
  local mtime age_min
  mtime=$(stat -f %m "$LOCK" 2>/dev/null || stat -c %Y "$LOCK" 2>/dev/null)
  [ -z "$mtime" ] && return 0
  age_min=$(( ($(date +%s) - mtime) / 60 ))
  [ "$age_min" -ge "$TTL_MIN" ]
}

acquire() {
  if [ -s "$LOCK" ]; then
    local holder
    holder=$(cat "$LOCK")
    [ "$holder" = "$SESSION_ID" ] && return 0       # idempotent re-entry
    if is_stale; then
      echo "Stale lock ($holder) — reclaiming" >&2
      rm -f "$LOCK"                                 # required so noclobber write succeeds
    else
      return 1                                      # active sibling — wait
    fi
  fi
  # noclobber atomic write — first writer wins on concurrent attempts
  ( set -C; echo "$SESSION_ID" > "$LOCK" ) 2>/dev/null && return 0 || return 1
}

WAITED=0
until acquire; do
  HOLDER=$(cat "$LOCK" 2>/dev/null || echo "?")
  echo "Waiting for /lightsout lock (holder=$HOLDER, waited=${WAITED}s/${WAIT_SEC}s)" >&2
  sleep "$POLL_SEC"
  WAITED=$((WAITED + POLL_SEC))
  if [ "$WAITED" -ge "$WAIT_SEC" ]; then
    echo "LIGHTSOUT_TIMEOUT waited=${WAITED}s holder=$HOLDER"
    exit 99
  fi
done

LOCK_CONFIRMED=1
echo "LIGHTSOUT_LOCK_OK session=$SESSION_ID waited=${WAITED}s"
```

**If the output contains `LIGHTSOUT_TIMEOUT`**: STOP. Do not proceed to Step −1. Report to the user that the sibling lightsout did not finish within 5 min — they should investigate whether the holder is stuck (check `~/.local/state/lightsout/active.session` and the holder's session) or run `/lightsout` again later. End the skill.

**If the output contains `LIGHTSOUT_LOCK_OK`**: continue to Step −1.

---

## Step −1 — Mark Lightsout Active (always runs after lock acquired)

Drop the sentinel file the session-length-guard hook checks. Without this, a high-count session that hits `/lightsout` near the EXIT threshold (count=60) gets halted by the guard mid-handover — the very wrap-up the guard was telling the user to run.

```bash
touch /tmp/claude-lightsout-active
```

The sentinel has a 30-min mtime TTL inside the guard — abandoned lightsouts self-heal so the guard re-arms automatically. Step 7 clears it on clean exit.

---

## Step 0 — Promote Pending Insights (always runs)

Single CLI call — `promote_insights.py` owns the deterministic state machine (read → filter `kp_candidate` → fingerprint-dedupe vs KNOWN_PATTERNS.md → FIPD-classify → append KP-N → archive → clear). Stdlib-only, atomic writes.

```bash
python3 ~/projects/00_Governance/scripts/promote_insights.py
```

The prod (option-2) `promote_insights.py` scans `KNOWN_PATTERNS.md` directly on every run, so there is no sidecar to rebuild — `max_kp` cannot drift behind file edits because the file *is* the source of truth.

Add `--dry-run` to preview, `--no-dedupe` to bypass the duplicate fingerprint check, `--max=N` to cap promotions per run (default 50). The script handles "Last updated" date stamping and archives all entries (KP + non-KP) to `~/.local/state/insights/archive/YYYY-MM.jsonl`.

**Behavior:**
- Empty/missing pending → prints "No pending insights" and exits 0
- Non-candidate entries are archived but not promoted
- FIPD classification is rule-based heuristic (Fix/Investigate/Decide/Plan in priority order); ambiguous entries default to Plan
- Section assignment is deferred — entries append to end of file under their KP heading; manual re-categorization happens during quarterly KP review
- Title derived from first sentence (truncated 80 chars)

Report what the script printed.

---

## Step 1 — Gather Completed Work

### Default mode (lite)

Single combined bash call — no separate file reads:

```bash
# Today's entries from DONE-Today (not the whole file)
grep -A 100 "^## $(date +%Y-%m-%d)" ~/projects/00_Governance/DONE-Today.md 2>/dev/null | head -100

# Git activity across ALL governed projects (auto-discovered, not hardcoded)
find ~/projects -maxdepth 2 -name ".git" -type d 2>/dev/null | while read gitdir; do
  d="$(dirname "$gitdir")"
  echo "=== $(basename "$d") ===" && git -C "$d" log --oneline --since="midnight" 2>/dev/null
done

# Always exit 0 — empty grep / empty find above would otherwise return non-zero
# and cancel any sibling Bash tools running in the same parallel batch.
exit 0
```

If both return empty, note "No tracked completions today — session work was ad-hoc" and continue.

### Full mode (`--full`)

Read ALL of the following:

1. **Global DONE-Today.md** — `~/projects/00_Governance/DONE-Today.md` — **only read from today's date heading forward**, not the entire file. Use `grep -A` or read with offset.
2. **Weekly archives** — `~/projects/00_Governance/done/DONE-*.md` — scan for today's date only
3. **Git log** — `git log --oneline --since="midnight"` across governed project directories
4. **Todo list** — any completed items from the current session's task tracking
5. **Task inbox** — recently processed items in `~/projects/00_Governance/task-inbox/`

### Step 1b — Backlog Reconciliation (--full only)

```bash
python3 ~/projects/00_Governance/scripts/backlog_reconciler.py ~/projects/00_Governance --days 7
```

- Exit 2 (stale stories): include in **Open Items** with `[RECONCILER]` tag
- Exit 0: note "Backlog reconciler: clean"

### Step 1c — Paperclip ↔ BACKLOG sync (--full only)

```bash
python3 ~/projects/00_Governance/scripts/paperclip_backlog_sync.py
```

Reads Paperclip for `done` issues with `US-XX-NN` ids, ticks matching ACs in `BACKLOG.md`, and rewrites the State line to `SHIPPED (GET-N done YYYY-MM-DD)`. Idempotent — skips blocks already containing `SHIPPED`.

- Reports `N block(s)` reconciled or "already in sync"
- Include the count in **Open Items** if non-zero so the next session sees it

---

## Step 2 — Summarize Session

### Default mode (lite)

Brief summary — 5-10 lines max:

```
## Session Summary — YYYY-MM-DD
- [Project]: [what was done, 1 line per item]
- Decisions: [key decisions, if any]
- KPs promoted: [list]
- Open: [carry-forward items]
```

### Full mode (`--full`)

Full structured summary with commit hashes, grouped by project:

```
## Session Summary — YYYY-MM-DD

### Completed
#### [Project Name]
- [US-XX: story title] — [brief outcome] (commit: `abc1234`)

### Decisions Made
- [decision with rationale]

### Open Items / Carry Forward
- [anything unfinished]

### Memories Updated
- [list any memory files created/updated]

### Governance State
- Pending suggestions: [count from ~/.local/state/governance/suggestions.jsonl]
- Last scan: [age from ~/.local/state/governance/last-scan.txt]
- Last retro: [age from ~/.local/state/governance/last-retro.txt]
- Pending insights: [count from ~/.local/state/insights/pending.jsonl]
```

---

## Step 2b — Automation Debt Sweep (--full only)

> Moved from default to `--full` on 2026-05-16. Per-commit `automation-debt-check.sh` is now the primary producer, so default-mode /lightsout no longer needs this sweep. Run end-of-day via `--full` to catch any uncommitted-session debt.

Sweep accumulated automation debt from `/reflect` scans into Paperclip tickets.

**Step 2b.0 — Safety-net catch-up scan (only if needed):**
Per-commit scans are now the primary producer: `automation-debt-check.sh` fires after every git commit and outputs an inline 6-question scan prompt that is filled out as part of the next response. Each commit's debt should already be in `pending.jsonl` by the time `/lightsout` runs.

This step is the safety net for work that didn't go through a commit (untracked sessions, discussion-only work that produced manual orchestration, etc.):

- **Skip if recent activity is fully committed** — if every meaningful action since the last `/lightsout` ended in a commit, the per-commit hook already captured it. Report `reflect: covered by per-commit scans` and continue.
- **Otherwise**, scan only the uncommitted portion of the session for the six patterns in `~/.claude/commands/reflect.md` (inline scripts, manual service calls, data transformation, manual orchestration, Skill-Unused, Skill-Missing).
- Append findings as JSON lines to `~/.local/state/automation-debt/pending.jsonl` in the schema defined by the reflect skill.
- Do NOT brainstorm fixes here — just log gaps. `/analyze-debt` does the root-cause work later.

Single CLI call — `sweep_automation_debt.py` is a thin wrapper around `paperclip.sh debt-sweep` (which already groups by project + creates issues + resolves project IDs via the live `/projects` API). The wrapper adds the archive/clear lifecycle so Step 2b is one atomic call.

```bash
python3 ~/projects/00_Governance/scripts/sweep_automation_debt.py
```

Flags: `--dry-run` (preview only — no Paperclip writes, no archive), `--no-paperclip` (archive without dispatching, useful when offline), `--assignee <agent>`, `--status <s>`, `--priority <p>`.

**Behavior:**
- Empty/missing pending → prints "No automation debt pending" and exits 0
- `kind: clean` / `gap: clean` markers are filtered (counted as "clean markers" in stdout, archived but not dispatched)
- Project-name → Paperclip-project-id resolution lives in `paperclip.sh` (`resolve` against live API), not in any local mapping table
- On Paperclip API failure: archive is **skipped** so pending.jsonl is preserved for retry. Exit code 4 signals partial run

Report what the script printed.

---

## Step 3 — Overnight DAGs (--full only)

**Pre-check before any DAG calls:**
```bash
# Single call: check all DAG states at once
for dag in phantom-autoresearch-panels phantom-autoresearch-memory phantom-autoresearch-router; do
  /opt/homebrew/opt/dagu/bin/dagu status "$dag" 2>/dev/null | head -1
done
```

**Rules:**
- **ppv-sweep**: NEVER start (user policy: stopped permanently)
- **phantom-autoresearch-***: Only start if the preflight step passes (currently blocked — scripts not implemented, US-PH-08/09/10). If all three show "not yet implemented" in status, report once: "Phantom autoresearch DAGs not yet built (US-PH-08/09/10)" and skip.
- **Other overnight DAGs**: Check `~/.config/dagu/dags/` for DAGs tagged `overnight` without `schedule:`. Start if not already running.

Log: "Started N overnight DAGs: [names]" or "No overnight DAGs to start"

---

## Step 4 — Publish to gtxs.eu (--full only)

Config: `~/projects/deploy/lightsout-config.json`

**4a + 4b — Prep manifest (filter, classify, secret-scan source)**

```bash
python3 ~/projects/00_Governance/scripts/publish_today.py prep --out /tmp/publish-manifest.json
```

The CLI does Step 4a (cross-reference DONE-Today against `allow_list.allowed_topics` / `blocked_topics`, classify learning/use-case/analysis, set target_sites per `routing_rule`, pick accent color) and Step 4b (regex secret scan over source content).

**Outcomes:**
- Exit 0, "Publishable: 0 item(s)" → "No publishable items today" and skip to Step 5
- Exit 0, "Publishable: N item(s)" → manifest at `/tmp/publish-manifest.json`, continue to 4c
- Exit 1, "BLOCKING: secret patterns" → STOP. Inspect `manifest.secret_hits`, fix source, re-run

**Manifest schema** (each item):
```json
{ "date": "...", "title": "...", "project": "...", "body": "...",
  "classification": "learning|use-case|analysis", "target_sites": ["..."], "accent": "..." }
```

**4c — Generate HTML**

*For gtxs.eu (learnings):*
- Dark theme: `#0a0c10` deep, `#12151c` card, `#1a1e28` elevated
- Fonts: DM Serif Display / Source Sans 3 / IBM Plex Mono
- Structure: fixed nav → hero → content sections → "Key Prompts" → related links → footer

*For getaccess.net:*
- Use nav pattern from existing `~/projects/deploy/site-getaccess/articles/`
- Add entry to `content.json`

**4d — Write & Update Indexes**
Write HTML files. Update index pages for both sites.

**4e — Final Secret Scan (BLOCKING GATE)**

```bash
python3 ~/projects/00_Governance/scripts/publish_today.py scan-html <path-to-each-generated-html>...
```

Re-scans rendered HTML against the same `secret_patterns` from config. Exit 1 = secret found in output → STOP, remove the file, fix the template, regenerate.

**4f — Deploy**
```bash
cd ~/projects/deploy && bash deploy.sh push-all
```

---

## Step W — Wiki Update (always runs)

Update the governance wiki (localhost:3100) for any project that was touched this session. The wiki is internal documentation — the threshold is **"did anything change?"**, not "is this publishable?"

### What triggers a wiki update

Any of these in the session's completed work:
- App changes (new features, bug fixes, UI changes, new pages/routes)
- Infrastructure/environment changes (deploy config, ports, Docker, VPS)
- Architecture changes (new modules, schema migrations, dependency changes)
- Skill/command changes (new skills, modified skills, new hooks)
- Claude Code config changes (CLAUDE.md, settings.json, MCP servers)
- New integrations or API endpoints

Routine operations that do NOT trigger wiki updates: running existing tests, reading code without changing it, pure backlog/planning work with no implementation.

### Wiki page routing

Use `wiki.py` CLI for all writes — never inline content in bash (escaping fails with special chars).

```
~/projects/00_Governance/00_Governance/wiki/scripts/wiki.py set-section <page-path> "<section heading>" <file.md>
```

**Discover the wiki page via QMD** — do NOT use a hardcoded mapping table. Query the `governance` collection for the wiki export file:

```
mcp__qmd__query searches=[
  {"type": "lex", "query": "wiki_path \"<project-keyword>\""}
] collections=["governance"] limit=3
```

The wiki export files at `governance/wiki/export-md/` contain frontmatter with `wiki_path` and `wiki_id`. Extract `wiki_path` from the QMD result to use with `wiki.py set-section`.

**Examples:** searching `"paperclip"` returns `wiki_path: "infrastructure/paperclip"`, `"svgpaint"` returns `wiki_path: "svgpaint"`, `"keto"` returns `wiki_path: "keto"`.

If QMD returns no wiki page for a touched project, **CREATE the page first**, then update it. Never skip a wiki update because no page exists. Use `wiki.py create "<title>" --path "<path>"` (or equivalent wiki.py create command) to register the page, then use `set-section` as normal. Note "Wiki page created: [path]" in the report.

### How to update

1. **Identify touched projects** from Step 1/2 completed work
2. **For each project with wiki page:**
   - Write a markdown snippet to `/tmp/wiki-<project>-update.md` with the changes
   - Use `set-section` to replace or append a dated section heading (e.g., "Recent Changes (2026-04-15)")
   - For new features: add to the relevant existing section instead of always creating "Recent Changes"
3. **Report:** "Wiki updated: [page-path] — [brief description]" per page, or "No wiki-eligible changes"

### Section format

```markdown
## Recent Changes (YYYY-MM-DD)

### [Feature/Fix Name]
[1-3 sentences describing what changed and why]

| Detail | Value |
|--------|-------|
| Files changed | `file1.py`, `file2.html` |
| Deployment | [deployed/not deployed] |
| Known issues | [if any] |
```

For schema/migration changes, update the existing schema section rather than appending.
For new routes/endpoints, update the existing API/Pages table.

---

## Step W2 — Ingest Prompt Usage (--full only, pre-HANDOVER)

> Moved from default to `--full` on 2026-05-16. Hourly launchd already keeps the prompt-usage DB current (AC-13); running it every checkpoint is redundant cost.

Refresh the claude-usage-systray Habits tab by ingesting new user prompts from `~/.claude/projects/*/conversations/*.jsonl` into the prompt-usage DB. Runs **before** HANDOVER is written so the post-run log state is visible to the next session. Non-blocking — ingest failure never aborts `/lightsout` (AC-14).

**Rules:**
- **30-second hard timeout** — if ingest stalls (large backlog, locked DB), we abort and move on.
- **Failure is logged, not raised** — on non-zero exit or timeout, append a one-line warning to HANDOVER.md so the next session sees the signal, then continue.
- **Log sink is `~/.local/state/prompt-usage-ingest.log`** — same path the hourly launchd job writes to (AC-13), so history is unified.
- **Never block session end.**

```bash
# Ingest prompt usage (non-blocking — 30s timeout, log-only failure)
# HANDOVER_PATH is set in Step 5 — ingest warnings are appended there if needed
mkdir -p "$HOME/.local/state"
cd /Users/jcords-macmini/projects/claude-usage-systray
if command -v timeout >/dev/null 2>&1; then
  timeout 30 python3 -m engine.ingest_prompts 2>&1 \
    | tee -a "$HOME/.local/state/prompt-usage-ingest.log"
  STATUS=${PIPESTATUS[0]}
  [ "$STATUS" -eq 124 ] && INGEST_WARN="⚠ ingest timed out at 30s"
  [ "$STATUS" -ne 0 ] && [ "$STATUS" -ne 124 ] && INGEST_WARN="⚠ ingest failed (see log)"
else
  python3 -m engine.ingest_prompts 2>&1 \
    | tee -a "$HOME/.local/state/prompt-usage-ingest.log"
  STATUS=${PIPESTATUS[0]}
  [ "$STATUS" -ne 0 ] && INGEST_WARN="⚠ ingest failed (see log)"
fi
# $INGEST_WARN (if set) is appended to the HANDOVER file written in Step 5
```

Report: `ingest: <N> msgs, <P>% matched, <M> unmatched` (from script stdout) or `ingest: skipped — <reason>`.

---

## Step M — Memory Index Compact (always runs, pre-HANDOVER)

Keep MEMORY.md within the ~24.4KB context load limit by trimming hooks to ≤100 chars. The full detail already lives in each topic file — the hook is just the index pointer.

```bash
python3 ~/projects/00_Governance/scripts/compact_memory_index.py
```

Add `--dry-run` to preview without writing. The script:
- Skips entries already within the limit (idempotent)
- Flags entries whose topic file is missing (does NOT truncate — would lose data)
- Appends `…` when truncating so the pointer is clearly incomplete
- Reports estimated bytes saved and new index size

Report what the script printed. If any `WARNING: topic file missing` lines appear, note them in the HANDOVER for manual cleanup.

---

## Step S — Session Shape Audit (--full only, pre-HANDOVER)

> **Migration note (2026-05-16):** Moved out of default. Shape audit is diagnostic, not load-bearing — running it every session costs ~4 tool calls for an output the user almost never reads in-line. Stays in `--full` for end-of-day review.

Classify this session's tool calls and report the realized shape. Read-only — never blocks /lightsout. The output is appended to the HANDOVER's "Session Shape" section.

```bash
# SESSION_TYPE is set by kickoff Step 3b; defaults to execution if absent
TYPE_FILE=~/.local/state/claude/session-type
SESSION_TYPE=$(head -1 "$TYPE_FILE" 2>/dev/null || echo "execution")
python3 ~/projects/00_Governance/scripts/session_shape_audit.py --type "$SESSION_TYPE"
```

The audit prints one line like:

```
sid8       tc  cer met dis imp ver glu oth  ratio              shape
abc12345   47   8   4  13  12   1   8   1  impl= 26% cere= 17%  execution-ok
```

**Shape labels:**
- `execution-ok` — declared execution + impl ≥25%
- `execution-empty` — declared execution + impl <25% (the failure mode this system targets)
- `meta (declared)` — declared meta, no impl (clean)
- `meta-creep` — declared meta but did impl anyway (mixed)
- `clearing` / `clearing-overrun` — declared clearing, ≤20 / >20 TC
- `shipped` / `mixed` / `no-impl` — undeclared sessions, descriptive only

Append the line to HANDOVER under `## Session Shape`. If the label is `execution-empty`, also append:

> WARNING: declared execution but did not ship — next session should re-attempt without governance side-quests.

**Calibration note:** thresholds (impl ≥25%, cere ≤20%) are empirically grounded against 26 audited sessions (impl median 8%, cere median 25%). Aspirational thresholds shame the rule; empirical ones enforce it. Revisit after ≥30 days of typed sessions.

---

## Step R — Backlog Roll-Call (--full only, pre-HANDOVER)

> **Migration note (2026-05-16):** Moved out of default. Portfolio-wide Ready sweep is expensive (multi-project grep + sort) and the data only matters at end-of-day or before /lightsout-driven planning. The next-session handover can re-query on demand.

Portfolio-wide answer to **"what is still left to implement overall?"**. Single batched bash block that mirrors Step 1's multi-project sweep but on the "what's left" side. Read-only — never blocks. Output is appended to HANDOVER under `## Backlog Roll-Call` so the next session sees portfolio state without re-querying.

The roll-call answers three questions in one pass:
1. **Central BACKLOG.md `## Ready`** — items cleared for execution
2. **Per-project `BACKLOG.md` `## Ready`** — project-scoped Ready counts
3. **TODO-Today.md leftovers** — unchecked `- [ ]` tasks the session didn't finish

```bash
# Single batched roll-call — all reads in one parallel-safe block. Never fails the skill.
ROLLCALL_OUT=/tmp/lightsout-rollcall.md
GW=~/projects/00_Governance

{
  echo "## Backlog Roll-Call"
  echo ""

  # --- Central BACKLOG.md Ready items (the canonical "what's left") ---
  CENTRAL="$GW/BACKLOG.md"
  if [ -f "$CENTRAL" ]; then
    READY_TITLES=$(awk '
      /^## Ready/ {flag=1; next}
      /^## (Refining|Ideation|Done|Critical|In Progress)/ {flag=0}
      /^# / {flag=0}
      flag && /^### US-/ {print}
    ' "$CENTRAL")
    READY_COUNT=$(printf "%s\n" "$READY_TITLES" | grep -c "^### US-" || echo 0)
    echo "### Central — $READY_COUNT Ready"
    if [ "$READY_COUNT" -gt 0 ]; then
      printf "%s\n" "$READY_TITLES" | sed 's/^### /- /' | head -20
      [ "$READY_COUNT" -gt 20 ] && echo "- … (+$((READY_COUNT - 20)) more)"
    fi
    echo ""
  fi

  # --- Per-project BACKLOG.md Ready counts ---
  echo "### By Project"
  echo ""
  echo "| Project | Ready | Top item |"
  echo "|---------|------:|----------|"
  for bk in ~/projects/*/BACKLOG.md; do
    [ -f "$bk" ] || continue
    proj=$(basename "$(dirname "$bk")")
    titles=$(awk '
      /^## Ready/ {flag=1; next}
      /^## (Refining|Ideation|Done|Critical|In Progress)/ {flag=0}
      /^# / {flag=0}
      flag && /^### US-/ {print}
    ' "$bk")
    n=$(printf "%s\n" "$titles" | grep -c "^### US-" || echo 0)
    [ "$n" -eq 0 ] && continue
    top=$(printf "%s\n" "$titles" | head -1 | sed 's/^### //' | cut -c1-70)
    echo "| $proj | $n | $top |"
  done
  echo ""

  # --- TODO-Today.md leftovers (unchecked tasks from active queues) ---
  TODO_LEFT=0
  TODO_DETAILS=""
  for todo in "$GW"/TODO-Today.md ~/projects/*/TODO-Today.md; do
    [ -f "$todo" ] || continue
    # Count unchecked top-level checkboxes
    unchecked=$(grep -cE "^- \[ \]" "$todo" 2>/dev/null || echo 0)
    [ "$unchecked" -eq 0 ] && continue
    TODO_LEFT=$((TODO_LEFT + unchecked))
    label=$(echo "$todo" | sed "s|$HOME/||")
    TODO_DETAILS="${TODO_DETAILS}- $label: $unchecked open\n"
  done
  if [ "$TODO_LEFT" -gt 0 ]; then
    echo "### TODO-Today leftovers — $TODO_LEFT total"
    printf "$TODO_DETAILS"
  else
    echo "### TODO-Today leftovers — none"
  fi
  echo ""
} > "$ROLLCALL_OUT" 2>/dev/null

# Print to stdout so the report shows it, and keep the file for Step 5 to append.
cat "$ROLLCALL_OUT"
exit 0
```

The output file `/tmp/lightsout-rollcall.md` is consumed by **Step 5** — append its contents verbatim under a `## Backlog Roll-Call` section in HANDOVER. If the file is missing or empty, skip the section silently (never fail the handover on roll-call issues).

**Why batched, not split:** sweeping 10+ BACKLOG.md files plus TODO-Today files via separate Read calls would cost ~12 tool calls and bloat context with full-file content. The grep/awk approach extracts only the Ready titles (<2KB total) in one bash invocation.

---

## Step 5 — Handoff Note (always runs)

Write a uniquely named handover file — no prompt, no confirmation.

**Filename pattern:** `HANDOVER-{PROJECT}-{YYYY-MM-DD-HHmm}.md`
**Location:** `~/projects/00_Governance/`
**Example:** `HANDOVER-75_Coaching-2026-04-27-1557.md`

**Output discipline:** after the file is written, the immediate user-facing line MUST be the full absolute path (e.g. `Handover written to: /Users/jcords-macmini/projects/00_Governance/HANDOVER-…md`). Never report only the basename — the absolute path is what makes the line a clickable IDE link and what lets the user `Read` it directly. This rule applies to every step in `/lightsout` that writes a handover (Step 5, Step ⏚ merge output, Step 8 clipboard echo).

Derive `{PROJECT}`:
- **Stamped session** (`$STAMP` non-empty from Step −3) → `{PROJECT} = $STAMP`. No heuristic, no scan. The stamp is the truth.
- **Unstamped session** → derive from the session's primary project (same as Step 6 attribution — use canonical names: `30_SVG-PAINT`, `50_KETO`, `20_CONSIGLIERE`, `75_Coaching`, `00_Governance`, etc.). If multiple projects were touched, use the one with the most DONE-Today entries or most commits.

Include:
- Last task and queue position
- All completed work (same detail as Step 2 summary)
- Decisions made
- Open items / carry-forwards
- **Every User Story title must carry a `*(Nd)*` age suffix** (days since creation — look up in BACKLOG.md, do not omit)
- Overnight DAGs started (if --full)
- Workspace state: last KP number, pending suggestions, scan/retro age, pending insights
- **Backlog Roll-Call** — append the contents of `/tmp/lightsout-rollcall.md` verbatim (produced by Step R). Skip the section silently if the file is missing or empty.
- Resume checklist

## Step ⏚ — Parallel-Session Merge (always runs, after HANDOVER write)

Detect sibling HANDOVER files written by parallel Claude sessions since this session started. If any are found, merge them into a single `HANDOVER-merged-{timestamp}.md` so the next session has one canonical resume pointer instead of N rival ones.

**Signal:** a handover is "parallel" iff it was created after `CLAUDE_SESSION_START_EPOCH`. Anything older belongs to a prior wrap-up and is left alone.

**Survivor selection (3-by-content-size):** the file with the largest byte count becomes the **primary**. Its body is copied verbatim into the merged file. Siblings contribute only their `Open Items`, `Resume Checklist`, `Backlog Roll-Call`, and `Next Tasks` sections under a `## Cross-Session Additions` block, deduped by US-id against the primary.

**Originals are preserved.** The merge file is additive — no source handover is moved or deleted. Audit trail stays intact, and `git log` of the worktree shows every per-session handover plus the merged consolidation.

```bash
# Derive the just-written handover (newest non-merged) — Step 8 uses the same
# pattern. The merge script discovers parallel siblings via the detect script
# using CLAUDE_SESSION_START_EPOCH.
GW=~/projects/00_Governance
SELF=$(ls -t "$GW"/HANDOVER-*.md 2>/dev/null | grep -v '/HANDOVER-merged-' | head -1)
if [ -z "$SELF" ]; then
  echo "Step ⏚: no handover to merge against — skipping"
else
  python3 "$GW/scripts/parallel_session_merge.py" --auto --self "$SELF"
fi
exit 0
```

**Outcomes:**
- "No parallel handovers to merge." → continue silently to Step D
- "Merged handover written: HANDOVER-merged-YYYY-MM-DD-HHmm.md" → report the path. The merged file goes through Step 5b's governance commit alongside the original.
- Script failure (non-zero) → log warning, continue. Never blocks `/lightsout`.

**Why this step lives after Step 5 and before Step 5b:** the merged file must exist on disk before the governance commit fires, so the commit captures the consolidation atomically with the primary handover.

---

## Step D — Spawn Dreaming (non-blocking, AC-06)

After writing HANDOVER.md, spawn Dreaming as a background process. This is non-blocking — /lightsout returns immediately without waiting for Dreaming to complete.

> [!IMPORTANT]
> **Step D MUST run in its own turn — never batch with Step 5b or any other Bash call.**
> Step 5b's governance commit can legitimately exit non-zero (pre-commit hook fail, empty diff, hook signing error). If Step D shares a parallel Bash batch with a sibling that errors, the harness cancels the spawn with `Cancelled: parallel tool call X errored` and Dreaming silently never starts.
> Step D runs **before** Step 5b precisely so its execution is independent of governance-commit outcome.

```bash
DREAMING_SCRIPT="$HOME/projects/00_Governance/scripts/dreaming.py"
DREAMING_LOG_DIR="$HOME/.local/state/dreaming"
mkdir -p "$DREAMING_LOG_DIR"
DREAMING_LOG="$DREAMING_LOG_DIR/run-$(date +%s%3N).log"

if [ -f "$DREAMING_SCRIPT" ]; then
  nohup python3 "$DREAMING_SCRIPT" --mode lightsout >> "$DREAMING_LOG" 2>&1 &
  DREAMING_PID=$!
  echo "Dreaming spawned (pid=$DREAMING_PID, log=$DREAMING_LOG)"
else
  echo "Dreaming script not found at $DREAMING_SCRIPT — skipping"
fi
```

Do NOT wait for Dreaming to complete. The session exits immediately after this step.

---

### Step 5a — ICS SITREP for touched incidents (US-ICS-9 AC-03/04)

Per spec `docs/specs/2026-05-27-ics-agent-governance.md` §11 ICS-9: before commit, /lightsout MUST run `/ics-sitrep` for any open incident the session touched, which writes the SITREP, appends an IC-LOG line, and updates `INCIDENTS.md` (the three operations are already a single transaction inside the `ics sitrep --write` CLI, see US-ICS-5). If the resulting status is `CLOSED`, run `ics close <incident_id>` — that CLI atomically writes the archive under `ics/archive/<incident_id>/`, deletes the runtime contract (per US-ICS-3A), and **`git add`s + commits the archive itself** (git cwd = `_REPO_ROOT` = `~/projects/00_Governance`) at close time. **`ics close` owns the single transaction** the spec demands. Step 5b's `ics/archive/**` allowlist (US-ICS-9 AC-05) is a **defensive backstop**, not the transaction boundary: because `ics close` already committed the archive, the Step 5b sweep normally finds nothing new (a benign no-op); it only catches the archive if `ics close`'s own commit failed to land.

**Touched incident set (Claude determines):** enumerate the open `incident_id`s this session interacted with — read the contract, ran `ics readback`, wrote a manual SITREP, or modified files referenced by the contract. If the session did not touch any open incidents, this step is a no-op. Do NOT iterate over all open incidents indiscriminately — that would emit empty SITREPs and pollute IC-LOG.

```bash
GW=~/projects/00_Governance
ICS_BIN="$GW/bin/ics"

# TOUCHED_INCIDENTS is set by Claude based on session context — populate as a
# space-separated list of incident_ids the session worked with, or leave empty
# if none. Example: TOUCHED_INCIDENTS="ics-2026-05-27-foo ics-2026-05-28-bar"
TOUCHED_INCIDENTS="${TOUCHED_INCIDENTS:-}"

if [ -z "$TOUCHED_INCIDENTS" ]; then
  echo "ICS_SITREP=no-touched-incidents"
elif [ ! -x "$ICS_BIN" ]; then
  echo "ICS_SITREP=skipped (bin/ics not executable)" >&2
else
  for INCIDENT_ID in $TOUCHED_INCIDENTS; do
    CONTRACT="$HOME/.local/state/ics/contracts/$INCIDENT_ID.json"
    if [ ! -f "$CONTRACT" ]; then
      echo "ICS_SITREP_SKIP=$INCIDENT_ID (no runtime contract; already closed?)" >&2
      continue
    fi
    # Emit SITREP — --write performs the atomic (sitrep + IC-LOG + INCIDENTS.md)
    # bundle inside the CLI, under portalocker advisory lock (US-ICS-2).
    "$ICS_BIN" sitrep "$INCIDENT_ID" --write || {
      echo "ICS_SITREP_FAIL=$INCIDENT_ID" >&2
      continue
    }
    # Read the resulting status from the contract (status field, NOT the SITREP
    # body — the contract is the source of truth post-sitrep).
    STATUS=$(python3 -c "import json,sys; print(json.load(open(sys.argv[1])).get('status','UNKNOWN'))" "$CONTRACT" 2>/dev/null)
    echo "ICS_SITREP_OK=$INCIDENT_ID status=$STATUS"
    if [ "$STATUS" = "CLOSED" ]; then
      # transactional close (US-ICS-9 AC-03/04) — `ics close` atomically writes
      # the archive tree under ics/archive/<incident_id>/, deletes the runtime
      # ~/.local/state/ics/ contract+sitreps for this incident, AND commits the
      # archive itself (git cwd = _REPO_ROOT; US-ICS-3A `cmd_close`). That commit
      # IS the single transaction. Step 5b's ics/archive/** sweep (US-ICS-9
      # AC-05) is only a backstop for the rare case ics close's commit didn't land.
      "$ICS_BIN" close "$INCIDENT_ID" || echo "ICS_CLOSE_FAIL=$INCIDENT_ID" >&2
      echo "ICS_CLOSED=$INCIDENT_ID (archive committed by ics close; Step 5b sweep is a backstop)"
    fi
  done
fi
```

Rationale for placement (between Step 5 Dreaming and Step 5b commit): Step 5a runs **before** Step 5b so that *if* `ics close`'s own commit failed to land, Step 5b's `ics/archive/**` backstop sweep can still pick up the orphaned archive files in the same session. (In the normal path `ics close` has already committed the archive, so ordering relative to Step 5b does **not** affect the single-transaction promise — that promise is kept by `ics close` itself, not by Step 5b.) Running it before Step 5 (Dreaming) would couple SITREP emission to background Dreaming success/failure, which is independent of session wrap-up.

If any incident's SITREP or close fails, Step 5a logs the failure to stderr and continues — Step 5b still proceeds, but the manually-flagged failures should be reviewed before the next session opens with that incident in the open set (the next /kickoff Step −0 will surface it).

---

### Step 5a-hooks — Hook Archive Drift Gate (always runs)

Hooks (`~/.claude/hooks/*.sh`) are edited live and were historically un-archived — a one-char path typo in `automation-debt-check.sh` rotted silently for ~weeks because the hook code had no diff/review/history (origin of the detector-trust spec, `specs/2026-06-02-debt-detector-trust-and-prohibition-design.md` §3.1). This gate fails loud on any runtime hook that drifts from its git-tracked archive (`00_Governance/.claude/hooks/`), so no un-mirrored hook edit survives the session that wrote it (P3 — fail loud, not silent).

**Direction of truth is INVERTED vs commands:** runtime is SoT, archive is the copy (`sync-global-hooks.sh` copies runtime → archive). The gate auto-mirrors any drift it finds; the **nightly janitor** commits `00_Governance/.claude/hooks/` (it is intentionally OUTSIDE Step 5b's commit allowlist — widening that allowlist would launder the KP-744 permission gate).

```bash
# AC-02: fail loud when a runtime hook is un-mirrored, then remediate by syncing
# (runtime -> archive). exit 0 at the end so a non-zero --check never cancels a
# sibling tool in a parallel Bash batch (same guard as Step 5b line ~796).
HSYNC=~/projects/00_Governance/scripts/sync-global-hooks.sh
if "$HSYNC" --check; then
  echo "hooks: archive in sync — no drift"
else
  echo "!! hooks: DRIFT — runtime hook(s) un-mirrored vs archive. Mirroring now:"
  "$HSYNC"
  "$HSYNC" --check && echo "hooks: re-check clean; nightly janitor will commit 00_Governance/.claude/hooks/"
fi
exit 0
```

---

### Step 5a-githooks — CC Commit-Guard Archive Drift Gate (always runs)

The global cc-commit-guard (`~/.config/git/hooks/pre-commit`, KP-1558) lives outside any repo, so it has the same un-versioned-rot exposure that motivated the hook-archive gate above. Its byte-identical archive lives at `00_Governance/scripts/git-hooks/pre-commit` with installer `scripts/git-hooks/cc-guard-sync.sh`. This gate fails loud if the live guard drifts from its archive.

**Direction of truth (same inversion):** runtime is SoT, archive is the copy. On drift the gate re-archives (runtime → repo). Unlike the `.claude/hooks/` archive, `scripts/git-hooks/` is **not** auto-committed by the nightly janitor — it is reviewed code, so a re-archive is left dirty for explicit commit (the loud message says so). Disarm check entirely with `git config --global --unset core.hooksPath`.

```bash
# Fail loud when the live commit-guard drifts from its versioned archive, then
# re-archive (runtime -> repo) and surface it for review. exit 0 so a non-zero
# --check never cancels a sibling tool in a parallel Bash batch.
GHOOK=~/projects/00_Governance/scripts/git-hooks/cc-guard-sync.sh
if [ -x "$GHOOK" ]; then
  if "$GHOOK" --check; then
    :
  else
    echo "!! cc-commit-guard: DRIFT vs archive. Re-archiving (runtime -> repo):"
    "$GHOOK"
    echo "   review + commit 00_Governance/scripts/git-hooks/ (not auto-committed)."
  fi
fi
exit 0
```

---

### Step 5b — Commit Governance Changes (always runs)

Per the global CLAUDE.md allowlist (pre-authorized: `KNOWN_PATTERNS.md`, `HANDOVER-*.md`, `BACKLOG.md` + project-scoped variants), Step 5b auto-commits **only those files**. Everything else in `00_Governance` stays dirty until explicit user review — splitting the staging to sneak unapproved paths in would launder the permission gate (explicitly prohibited by KP-744).

**Exception — `ics/archive/**` (US-ICS-9 AC-05).** Closed-incident archives written by `/ics-sitrep` (on contract status `CLOSED`) are added to this auto-commit allowlist as a **deliberate governance change**, not as an implementation detail. Rationale: archived incidents are immutable audit artifacts (one-shot writes at incident close), so the laundering risk KP-744 guards against does not apply — there is no "future edit to sneak through." The pattern is narrowly scoped (`ics/archive/**` only, never `ics/contracts/` runtime tree which lives in `~/.local/state/`). Spec: `docs/specs/2026-05-27-ics-agent-governance.md` §11 ICS-9.

**Decision (2026-06-01, US-GOV-ICS-SPEC-TXN-DOC-01 — FOR-DEC light): KEEP this sweep + allowlist entry as a backstop.** `ics close` already `git add`s + commits the archive at close time (`ics_lifecycle.py` `close`, git cwd = `_REPO_ROOT`), so in the normal path this Step 5b sweep is a no-op. Options weighed: **(A)** keep the `ics/archive/**` sweep + allowlist entry as a belt-and-suspenders backstop; **(B)** remove both as dead code. Chose **A**. Trade-off: `ics close`'s commit is best-effort — if its `git commit` fails (pre-commit hook rejection, signing error, lock contention) the archive is written to disk but left uncommitted, and this sweep is then the only thing that commits it on session wrap-up. Cost of keeping is ~10 lines of no-op sweep + one narrowly-scoped allowlist entry over immutable audit artifacts (KP-744 laundering risk does not apply, per the rationale above); benefit is converting a silent-uncommitted-archive failure into self-healing on the next `/lightsout`. Reversible — revisit if US-ICS-11 telemetry shows the sweep never fires in practice.

```bash
# All governance files live in the git-backed repo at GW (~/projects/00_Governance).
# The legacy hermes governance-worktree is a frozen archive mirror — never commit there.
GW=~/projects/00_Governance

CHANGED=()

# Modified allowlist files
for f in BACKLOG.md KNOWN_PATTERNS.md; do
  git -C "$GW" diff --quiet -- "$f" 2>/dev/null || CHANGED+=("$f")
done

# New HANDOVER-*.md files (untracked) — read line-by-line so each filename lands
# in its own array slot. CHANGED+=($HANDOVER_NEW) word-splits at the array-append
# boundary but later "${CHANGED[@]}" preserves a multi-line slot as one joined arg,
# making `git add` see "file1\nfile2" as a single pathspec.
HANDOVER_NEW=$(git -C "$GW" ls-files --others --exclude-standard 'HANDOVER-*.md')
while IFS= read -r f; do
  [ -n "$f" ] && CHANGED+=("$f")
done <<< "$HANDOVER_NEW"

# Closed-incident archives under ics/archive/** (US-ICS-9 AC-05). Both tracked
# changes and new untracked files — closing an incident writes new archive files,
# subsequent edits are not expected (immutable audit artifacts) but tolerated.
ICS_ARCHIVE_TRACKED=$(git -C "$GW" diff --name-only -- 'ics/archive/**' 2>/dev/null)
ICS_ARCHIVE_NEW=$(git -C "$GW" ls-files --others --exclude-standard 'ics/archive/**')
while IFS= read -r f; do
  [ -n "$f" ] && CHANGED+=("$f")
done <<< "$ICS_ARCHIVE_TRACKED"
while IFS= read -r f; do
  [ -n "$f" ] && CHANGED+=("$f")
done <<< "$ICS_ARCHIVE_NEW"

if [ ${#CHANGED[@]} -eq 0 ]; then
  echo "governance-worktree allowlist clean — no commit needed"
else
  git -C "$GW" add "${CHANGED[@]}"
  # Pathspec-scoped commit (KP-744/KP-1558): `git commit -- <paths>` restricts the
  # commit to ONLY the allowlist, regardless of what else is staged in the index.
  # A bare `git commit` here would serialize the whole index — sweeping in any
  # pre-existing staged runtime/spec files and laundering the permission gate.
  # The preceding `git add` stays: untracked HANDOVER/ics-archive files must be
  # staged first for the pathspec commit to include them.
  git -C "$GW" commit -m "chore(governance): /lightsout session wrap-up $(date +%Y-%m-%d)" -- "${CHANGED[@]}"
  echo "governance-worktree committed: ${CHANGED[*]}"
fi

OTHER=$(git -C "$GW" status --porcelain | wc -l | tr -d ' ')
[ "$OTHER" -gt 0 ] && echo "Note: $OTHER other governance files dirty — review manually"
# Guard final conditional — when OTHER=0 the `&&` returns false and would
# cancel any sibling Bash tools running in the same parallel batch.
exit 0
```

Rules:
- **HANDOVER files are never overwritten** — each session produces a unique timestamped file.
- **Strict allowlist, not pattern-based** — only `KNOWN_PATTERNS.md`, `BACKLOG.md`, new `HANDOVER-*.md` files (+ per-project variants), and `ics/archive/**` (the one sanctioned pattern-based exception per US-ICS-9 AC-05; rationale in Step 5b body). Adding other files violates the policy Stream A set 2026-04-20.
- **No split-commit workaround** — if you're tempted to stage a non-allowlist file alongside, STOP and ask the user. Splitting the commit launders permission.
- **Scope is `00_Governance` only** — project repos (SVG-PAINT, Consigliere, KETO, etc.) never auto-commit here.
- **No `--no-verify`** — honor pre-commit hooks.
- **Push is not part of this step** — local commit only.
- **Skip when clean** — idempotent when re-run in a clean tree.

---

## Step 6 — Session Attribution (always runs)

```bash
# Find session UUID
ls -t ~/.claude/projects/-Users-jcords-macmini-projects/*.jsonl | head -1
```

Determine primary project from session work. Append to `~/.local/state/codeburn/session-projects.jsonl`:
```bash
mkdir -p ~/.local/state/codeburn
echo '{"session_id": "<uuid>", "project": "<canonical name>", "date": "YYYY-MM-DD"}' >> ~/.local/state/codeburn/session-projects.jsonl
```

---

## Step 7 — Release Lightsout Locks (always runs last)

Remove both the guard sentinel (Step −1) and the cross-session lock (Step −2) so the next `/lightsout` can proceed without waiting. Idempotent — `rm -f` is safe if either file is already gone (partial run, manual cleanup, stale-reclaim by a sibling).

The lock release is **last** so Steps 0–6 retain exclusive access to the shared governance files for the entire wrap-up. If a sibling is queued (polling), it acquires within `POLL_SEC` (~10s) of this step.

```bash
rm -f /tmp/claude-lightsout-active
rm -f ~/.local/state/lightsout/active.session
```

---

## Step 8 — Clipboard Export (always runs, after lock release)

Copy the handover filename and key next-session context to the macOS clipboard so the user can paste directly into a new blank session window — no search, no manual copy.

```bash
HANDOVER_FILE=$(ls -t ~/projects/00_Governance/HANDOVER-*.md 2>/dev/null | head -1)
if [ -z "$HANDOVER_FILE" ]; then
  echo "Clipboard skip: no HANDOVER file found"
else
  HANDOVER_NAME=$(basename "$HANDOVER_FILE")

  # Extract Open Items section (stop at next ## heading)
  OPEN=$(awk '/^## Open Items/{found=1; next} found && /^## /{exit} found{print}' "$HANDOVER_FILE" | head -25)
  # Extract Resume Checklist section (stop at next ## heading)
  RESUME=$(awk '/^## Resume Checklist/{found=1; next} found && /^## /{exit} found{print}' "$HANDOVER_FILE" | head -15)

  {
    printf "Continuing from: %s\n" "$HANDOVER_FILE"
    [ -n "$OPEN" ] && printf "\n## Open Items\n%s\n" "$OPEN"
    [ -n "$RESUME" ] && printf "\n## Resume Checklist\n%s\n" "$RESUME"
  } | pbcopy

  # User-facing terminal output MUST be the full absolute path so the IDE
  # renders it as a clickable link (per feedback_handover_print_full_path).
  echo "Handover written to: $HANDOVER_FILE"
  echo "Clipboard ready — paste into new session."
fi
```

Skipped silently on `--dry-run`. macOS only (`pbcopy` — no-op on other platforms).

---

## Usage

```
/lightsout              # checkpoint (default): Steps 0, 1-lite, W, M, 5, ⏚, D, 5b, 6, 7, 8 — target <3 min
/lightsout --full       # full pipeline: adds 1b, 1c, 2, 2b, 3, 4, W2, S, R — end-of-day only
/lightsout --dry-run    # show what would happen, no writes
/lightsout [date]       # retroactive for a specific date (implies --full)
```

### When to use which

- **After any session**: `/lightsout` (default) — ~5-7 tool calls, <3 min, captures handover + commits governance + releases lock
- **End of day**: `/lightsout --full` — 15-25 tool calls, publishes articles, sweeps debt, ingests prompts, audits shape, rolls call backlog
- **After major deliverable** (PR, panel review, batch completion): `/lightsout` then fresh session
- **Session-length-guard warning** (25 tool calls): `/lightsout` immediately, then fresh session

### What got moved out of default (2026-05-16)

The default mode was bleeding 6+ minutes on steps that don't belong on the per-session hot path:
- **W2 (prompt-usage ingest)** — already covered by hourly launchd job
- **S (session shape audit)** — diagnostic, not load-bearing
- **R (backlog roll-call)** — expensive multi-project grep, only matters at EOD
- **1c (paperclip ↔ backlog sync)** — runs as Paperclip cron elsewhere
- **2b (automation debt sweep)** — per-commit hooks already capture this

Lock TTL was also dropped from 30 min → 5 min so orphan locks self-heal in minutes, not half an hour. A `trap EXIT` was added to Step −2 so unconfirmed locks clean themselves up when /lightsout exits early. The 9-min wait that triggered this rewrite is no longer reachable in the default path.

### Cost Discipline

The default mode exists because context replay is the hidden cost multiplier — every turn resends the full conversation. A checkpoint in a 300K session costs 6x per turn vs a fresh 50K session. Rules:
- After every commit: evaluate `/lightsout` if context is heavy
- Never start a new logical task in a bloated session
- Cache-aware: sleep ≤270s in loops to stay inside 5-min prompt cache TTL
