---
name: sh-business-panel
description: "Multi-expert business analysis with advisory recommendations (no scoring gate)"
---

# /sh:business-panel — Business Panel Analysis

## Usage

```
/sh:business-panel [document_path_or_content] [--mode discussion|debate|socratic] [--focus competitive|growth|risk|communication] [--experts "name1,name2"] [--synthesis-only]
```

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/business-panel.yaml` for panel definition, focus areas, and auto-select rules (absolute path — relative paths fail when CWD is outside agentflow)
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert — these contain domain, methodology, and critique focus
3. **Auto-Select Experts**: Scan content against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 6` cap
4. **Analyze**: Parse business content, identify strategic themes and domains
5. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts`. `--experts` override replaces defaults entirely
6. **Conduct Analysis**: Run analysis in the selected mode using each expert's distinct framework
7. **Synthesize**: Generate consolidated findings with consensus, disagreements, and prioritized recommendations

**No scoring gate** — this is an advisory panel. It produces strategic analysis and recommendations only.

## Expert Panel (9 experts from core-business pack)

| Expert                         | Domain                                      |
|--------------------------------|---------------------------------------------|
| Clayton Christensen            | Disruption Theory, Jobs-to-be-Done          |
| Michael Porter                 | Competitive Strategy, Five Forces            |
| Peter Drucker                  | Management Philosophy, MBO                   |
| Seth Godin                     | Marketing Innovation, Tribe Building         |
| W. Chan Kim & Renee Mauborgne | Blue Ocean Strategy                          |
| Jim Collins                    | Organizational Excellence, Good to Great     |
| Nassim Nicholas Taleb          | Risk Management, Antifragility               |
| Donella Meadows                | Systems Thinking, Leverage Points            |
| Jean-luc Doumont               | Communication Systems, Structured Clarity    |

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Collaborative analysis where experts build upon each other's insights through their frameworks. Default mode. Sequential commentary, cross-expert validation, consensus building.

### Debate Mode (`--mode debate`)
Adversarial analysis for stress-testing ideas. Experts challenge each other's positions, surface disagreements, and argue alternatives. Use for controversial topics or high-stakes decisions.

### Socratic Mode (`--mode socratic`)
Question-driven exploration for deep strategic thinking. Experts pose probing questions rather than giving answers. Forces deeper examination of assumptions and alternatives.

## Focus Areas

- **competitive**: Competitive positioning, market forces, strategy. Lead: Michael Porter. Experts: Porter, Christensen, Kim & Mauborgne
- **growth**: Marketing, tribe building, scaling. Lead: Seth Godin. Experts: Godin, Collins, Drucker
- **risk**: Risk management, antifragility, systems dynamics. Lead: Nassim Taleb. Experts: Taleb, Meadows
- **communication**: Structured clarity, presentation, stakeholder messaging. Lead: Jean-luc Doumont. Experts: Doumont, Godin

## Output

Business analysis document containing:
- Expert perspectives from selected panelists
- Consensus points across experts
- Disagreements with reasoning from each side
- Priority-ranked strategic recommendations
- Actionable next steps

**AUTO-FIX, NOT SYNTHESIS-ONLY** — when this panel reviews a written artifact (a spec, doc, or plan), it applies fixes for **every** finding automatically, per `00_Governance/CLAUDE.md §8 Panel Auto-Fix Policy` — it never asks which to apply and never presents a menu. The only boundary it preserves: it does not *execute real-world business decisions* (make outward moves, spend, or code changes beyond the reviewed artifact) without approval — that is a reversibility guard, not finding-fix friction.

## Auto-Fix Policy
Fix ALL findings on the reviewed artifact automatically — high, medium, and low severity. Do not ask which findings to fix. Do not present a menu or an a/b/c choice. Fix everything, then report what was changed. Per `00_Governance/CLAUDE.md §8`.
