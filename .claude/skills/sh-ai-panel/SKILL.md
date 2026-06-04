---
name: sh-ai-panel
description: "Multi-expert AI/ML specification review with scoring gate — model architecture, evaluation rigor, safety/alignment, and production readiness for AI systems and LLM apps"
---

# /sh:ai-panel — Expert AI/ML Review Panel

## Usage

```
/sh:ai-panel [specification_content|@file] [--mode discussion|critique|socratic|debate] [--focus fundamentals|evaluation|safety|production|ethics-data] [--experts "name1,name2"] [--iterations N]
```

## When to Use

- Reviewing AI/ML system specs, training plans, or eval designs before implementation
- Pressure-testing LLM-app architectures (RAG pipelines, agent frameworks, fine-tune plans)
- Checking capability claims for hype, contamination, or unjustified anthropomorphism
- Evaluating safety/alignment posture proportionate to system capability
- Costing scaling proposals (FLOPs, data budget, serving cost) before commitment

Use `sh-spec-panel` instead for classical software specs without an AI/ML core. Use `sh-research-panel` for OSINT/data-collection plans.

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/ai-panel.yaml` for focus areas, auto-select rules, and scoring config (absolute path — relative paths fail when CWD is outside agentflow)
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert
3. **Auto-Select Experts**: Scan the specification content against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 6` cap
4. **Analyze**: Parse spec content, identify model/data/eval/serving components, surface gaps
5. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts`. `--experts` override replaces defaults entirely
6. **Conduct Review**: Run analysis in the selected mode using each expert's distinct methodology
7. **Score**: Rate across 4 dimensions (0-10 each), compute overall score
8. **Gate Check**: Overall score must be >= 7.0 to pass. Below threshold = AI spec needs rework

## Expert Loading

Experts are defined as individual markdown files in `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/`. Each file contains:
- Domain and methodology
- Critique voice (opening move in panel discussion)
- Looks-for / red-flags / approves-when triplet
- Interaction style across modes

The panel YAML defines which experts belong to which focus area, who leads each, auto-select keyword rules, and scoring config.

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Experts build on each other's insights. Karpathy reduces abstract claims to concrete experiments; Chollet sharpens eval design; Huyen pulls toward deployment realism; Bender disciplines capability claims. Synthesis-first.

### Critique Mode (`--mode critique`)
Severity-classified issues (CRITICAL / MAJOR / MINOR), each with expert attribution, specific recommendation, and impact estimate. Highest-signal for gate decisions.

### Socratic Mode (`--mode socratic`)
Each expert asks the questions they would ask before approving. No direct answers — forces the author to confront gaps. Useful early in design.

### Debate Mode (`--mode debate`)
Adversarial stress-test. Karpathy vs Bender on capability claims; Kaplan vs Chollet on scale vs generalization; Russell vs Huyen on safety vs velocity. Surfaces hidden assumptions.

## Focus Areas

- **fundamentals**: Architecture, training, scaling assumptions. Lead: Karpathy. Experts: Karpathy, Kaplan, Chollet
- **evaluation**: Benchmark design, contamination, generalization. Lead: Chollet. Experts: Chollet, Bender, Karpathy
- **safety**: Alignment, harms, interpretability, agentic risks. Lead: Russell. Experts: Russell, Olah, Gebru
- **production**: MLOps, serving, latency, cost, drift. Lead: Huyen. Experts: Huyen, Karpathy
- **ethics-data**: Provenance, bias, documentation, stakeholder harms. Lead: Gebru. Experts: Gebru, Bender

## Scoring Gate

4 dimensions, each scored 0-10:

| Dimension              | Description                                                                       |
|------------------------|-----------------------------------------------------------------------------------|
| technical-soundness    | Does the architecture/method actually work for the problem; are assumptions defensible |
| evaluation-rigor       | Are success metrics measurable, contamination-controlled, gaming-resistant         |
| safety-alignment       | Are risks, harms, misuse paths, alignment concerns addressed proportionate to capability |
| production-readiness   | Are data pipelines, serving cost, latency, monitoring, failure modes accounted for |

**Pass threshold: overall score >= 7.0**

Output includes per-dimension scores, overall score, critical issues, consensus points, disagreements, and an improvement roadmap (immediate / short-term / long-term).

## Output

AI/ML review document containing:
- Multi-expert analysis with distinct perspectives
- Per-dimension scores and overall quality score
- Pass/fail gate result
- Critical issues with severity and priority
- Consensus points and named disagreements
- Priority-ranked improvement recommendations

**AUTO-FIX, NOT SYNTHESIS-ONLY** — this panel produces the analysis AND then applies fixes for **every** finding (high, medium, and low) automatically, per `00_Governance/CLAUDE.md §8 Panel Auto-Fix Policy`. It never asks which findings to apply and never presents a menu: it fixes everything, then reports what changed. A below-gate score means fix the findings and re-run, not stop and ask.

## Auto-Fix Policy
Fix ALL findings automatically — high, medium, and low severity. Do not ask which findings to fix. Do not present a menu or an a/b/c choice. Fix everything, then report what was changed. This overrides any softer "synthesis only" framing. Per `00_Governance/CLAUDE.md §8`.

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
