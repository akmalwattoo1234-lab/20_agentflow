---
name: sh-security-panel
description: "Multi-expert security review — threat modeling, secrets handling, LLM/prompt injection, data classification, incident response"
---

# /sh:security-panel — Expert Security Review Panel

## Usage

```
/sh:security-panel [content|@file] [--mode discussion|critique|socratic|debate] [--focus threat-model|secrets|llm-security|data-handling|resilience] [--experts "name1,name2"] [--iterations N]
```

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/security-panel.yaml` for panel definition, focus areas, auto-select rules, and scoring config (absolute path — relative paths fail when CWD is outside agentflow).
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert.
3. **Auto-Select Experts**: Scan the content against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 6` cap.
4. **Pre-Scoring Checks (deterministic — do NOT delegate to LLM personas):**
    - **Secret-leak scan:** Shape-detect literal secrets in the diff/spec — high-entropy strings near `key=`, `token=`, `password=`, `Authorization:`, `AKIA*` (AWS), `ghp_*` (GitHub), `sk-*` (OpenAI), `xoxb-*` (Slack), inlined `.env` content. Any match is a CRITICAL finding flagged before any expert speaks. Rationale: a leaked secret is a fact, not an opinion — no need for expert deliberation to confirm.
    - **Trust-boundary inventory:** Enumerate every external data ingress (HTTP body, file upload, scraped page, LLM output, env var sourced from third-party, plugin input). For each, note whether the design specifies a validation/classification step before the data crosses into business logic. Missing boundary checks become MAJOR findings before deliberation.
5. **Analyze**: Parse content (spec / diff / threat model / runbook) and surface attack-surface, trust-zone, and secrets-handling gaps.
6. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts` from panel YAML. `--experts` override replaces defaults entirely.
7. **Conduct Review**: Run analysis in the selected mode using each expert's distinct methodology.
8. **Score**: Rate the design across 4 dimensions (0-10 each), compute overall score.
9. **Gate Check**: Overall score must be >= 7.0 to pass.

## Expert Loading

Experts are defined as individual markdown files in `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/`. Each file contains structured frontmatter with:
- Domain and specialization
- Methodology and frameworks
- Critique focus and typical questions

The panel YAML (`/Users/jcords-macmini/projects/20_agentflow/experts/panels/security-panel.yaml`) defines:
- Which experts belong to which focus area
- Who leads each focus area
- Auto-select keyword rules for dynamic expert addition
- Scoring dimensions and pass threshold

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Collaborative threat-model refinement. Experts build on each other's observations — Valsorda's trust-zone mapping surfaces Willison's prompt-injection concerns, which surface Nygard's blast-radius questions.

### Critique Mode (`--mode critique`)
Systematic review with severity-classified findings (CRITICAL / MAJOR / MINOR). Each finding includes expert attribution, the threat it exploits, the asset at risk, and a concrete mitigation. Critical findings name the adversary explicitly — "an attacker controlling the scraped page can..." rather than "this is unsafe".

### Socratic Mode (`--mode socratic`)
Foundational questioning — "who is the adversary?", "what is the asset?", "where is the trust boundary?", "what does compromise look like, and how would we detect it?". No direct answers; forces the author to defend the threat model.

### Debate Mode (`--mode debate`)
Two-camp adversarial format around a contested security trade-off (e.g. usability vs hardening, hard-enforced vs documented isolation). Three rounds, then a synthesized trade-off statement.

## Focus Areas

- **threat-model**: Adversary modeling, attack surface enumeration, trust zones, asset classification. Lead: Filippo Valsorda. Experts: Valsorda, Nygard, Willison.
- **secrets**: Credential storage, rotation, transit, blast-radius limitation, OAuth/JWT lifecycle. Lead: Filippo Valsorda. Experts: Valsorda, Hightower, Majors.
- **llm-security**: Prompt injection, indirect injection via untrusted data, output sanitization, tool-use authorization, agent trust boundaries. Lead: Simon Willison. Experts: Willison, Valsorda, Nygard.
- **data-handling**: PII classification, GDPR boundaries, retention, deletion paths, subject-access requests, taxonomy at ingest. Lead: Filippo Valsorda. Experts: Valsorda, Majors.
- **resilience**: Incident response, blast-radius, detection latency, post-mortem discipline, fault isolation, supply-chain dependencies. Lead: Michael Nygard. Experts: Nygard, Majors, Hightower.

## Scoring Gate

4 dimensions, each scored 0-10:

| Dimension                  | Description                                                                                                  |
|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Threat-model coverage      | Are adversaries named, assets classified, attack surface enumerated; or is 'security' deferred to a future phase |
| Secrets discipline         | Are secrets stored, transported, rotated, and scoped correctly; is the blast radius of a leaked credential bounded |
| Trust-boundary enforcement | Are trust zones explicit and validated at ingress; or is 'we control both sides' used as justification for skipping isolation |
| Failure and detection      | Are detection paths and incident-response procedures specified; can a compromise be observed and contained, not just postulated |

**Pass threshold: overall score >= 7.0**

## Output

Security review document containing:
- Pre-scoring check results (secret-leak scan, trust-boundary inventory) surfaced FIRST
- Multi-expert analysis with distinct perspectives
- Per-dimension scores and overall quality score
- Pass/fail gate result
- Critical issues with severity, adversary named, and asset at risk
- Consensus points and disagreements
- Priority-ranked mitigations

**AUTO-FIX, NOT SYNTHESIS-ONLY** — this panel produces the analysis AND then applies fixes for **every** finding (high, medium, and low) automatically, per `00_Governance/CLAUDE.md §8 Panel Auto-Fix Policy`. It never asks which findings to apply and never presents a menu: it fixes everything, then reports what changed. A below-gate score means fix the findings and re-run, not stop and ask.

## Auto-Fix Policy
Fix ALL findings automatically — high, medium, and low severity. Do not ask which findings to fix. Do not present a menu or an a/b/c choice. Fix everything, then report what was changed. This overrides any softer "synthesis only" framing. Per `00_Governance/CLAUDE.md §8`. (Real-world *execution* beyond editing the reviewed artifact — e.g. force-push for history rewrite — stays gated per the note below.)

**Note on secret findings:** If the pre-scoring check finds a literal secret in the diff, the recommended remediation is rotation FIRST, then commit history rewrite (per `~/.claude/CLAUDE.md` — force-push is still gated and requires explicit authorization).

## Output Contract (machine-readable verdict)

The quality gate (`quality_gate.run_stage3_panel`) consumes this panel via
`claude -p` and reads the **score from stdout, not the exit code**. Therefore
the **final line** of your output MUST be exactly one of:

```
PANEL-VERDICT: <overall_score>
PANEL-VERDICT: FAIL: <machine_reason>
```

- `<overall_score>` is the numeric overall score (0–10, one decimal — e.g. `8.3`).
  The gate applies the pass threshold (default 7.0); do NOT pre-apply it — just
  report the score you computed.
- Use the `FAIL: <machine_reason>` form only when no score could be produced
  (structural failure) — snake_case naming the first blocker (e.g.
  `no_content`, `panel_config_missing`, `experts_unavailable`).
- Emit the line literally, on its own line, as the last meaningful output.
  Omitting it makes the gate fail-closed with `panel_no_verdict` (inconclusive).
