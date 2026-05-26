---
name: sh-spec-panel
description: "Multi-expert specification review with scoring gate and quality assessment"
---

# /sh:spec-panel — Expert Specification Review Panel

## Usage

```
/sh:spec-panel [specification_content|@file] [--mode discussion|critique|socratic] [--focus requirements|architecture|testing|compliance] [--experts "name1,name2"] [--iterations N]
```

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/spec-panel.yaml` for panel definition, focus areas, auto-select rules, and scoring config (absolute path — relative paths fail when CWD is outside agentflow)
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert — these files contain the expert's domain, methodology, and critique focus
3. **Auto-Select Experts**: Scan the specification content against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 7` cap
4. **Pre-Scoring Checks (deterministic — do NOT delegate to LLM personas):**
    - **Table reconciliation:** If the spec contains 2+ tables with numeric values (counts, totals, "≥ N" thresholds), extract every numeric claim and verify cross-section arithmetic *before* personas opine. Enumerate the items the count refers to in the other sections; assert the math reconciles. Flag any mismatch as a CRITICAL finding up front. Rationale: LLMs reliably hallucinate arithmetic consistency over freeform tables — personas reading personas will not catch it. This step is mechanical, executed by the panel runner, not by an expert voice.
5. **Analyze**: Parse specification content, identify components, gaps, and quality issues
6. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts` from panel YAML. `--experts` override replaces defaults entirely
7. **Conduct Review**: Run analysis in the selected mode using each expert's distinct methodology
8. **Score**: Rate specification across 4 dimensions (0-10 each), compute overall score
9. **Gate Check**: Overall score must be >= 7.0 to pass. Below threshold = specification needs rework

## Expert Loading

Experts are defined as individual markdown files in `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/`. Each file contains structured frontmatter with:
- Domain and specialization
- Methodology and frameworks
- Critique focus and typical questions

The panel YAML (`/Users/jcords-macmini/projects/20_agentflow/experts/panels/spec-panel.yaml`) defines:
- Which experts belong to which focus area
- Who leads each focus area
- Auto-select keyword rules for dynamic expert addition
- Scoring dimensions and pass threshold

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Collaborative improvement through expert dialogue. Experts build upon each other's insights sequentially. Cross-expert validation and consensus building around critical improvements.

### Critique Mode (`--mode critique`)
Systematic review with severity-classified issues (CRITICAL / MAJOR / MINOR). Each finding includes: expert attribution, specific recommendation, priority ranking, and quality impact estimate.

### Socratic Mode (`--mode socratic`)
Learning-focused questioning to deepen understanding. Experts pose foundational questions about purpose, stakeholders, assumptions, and alternatives. No direct answers — forces the author to think critically.

## Focus Areas

- **requirements**: Requirement clarity, completeness, testability. Lead: Karl Wiegers. Experts: Wiegers, Adzic, Cockburn
- **architecture**: Interface design, boundaries, scalability, patterns. Lead: Martin Fowler. Experts: Fowler, Newman, Hohpe, Nygard
- **testing**: Test strategy, coverage, edge cases, acceptance criteria. Lead: Lisa Crispin. Experts: Crispin, Gregory, Adzic
- **compliance**: Regulatory coverage, security, operational requirements. Lead: Karl Wiegers. Experts: Wiegers, Nygard, Hightower

## Scoring Gate

### Base dimensions (always applied)

4 dimensions, each scored 0-10:

| Dimension     | Description                                    |
|---------------|------------------------------------------------|
| Clarity       | Language precision and understandability        |
| Completeness  | Coverage of essential specification elements   |
| Testability   | Measurability and validation capability        |
| Consistency   | Internal coherence and contradiction detection |

### AI-production dimensions (conditional)

Applied when the spec contains AI/LLM-app keywords (llm, prompt, codex, claude, gpt, agent, rag, embedding, multi-agent, pipeline, model). Source: [PRODUCTION-READINESS-CHECKLIST.md](../../../projects/00_Governance/PRODUCTION-READINESS-CHECKLIST.md) §3, §4, §8.

| Dimension              | What it scores                                                                                      |
|------------------------|-----------------------------------------------------------------------------------------------------|
| Output integrity (§3)  | Schema validation, stripping of LLM reasoning artifacts, atomic multi-source writes, refuse-malformed |
| Isolation enforcement (§4) | Multi-agent "independence" backed by OS-level isolation, not prompt instruction (KP-3682)        |
| Security boundaries (§8)   | "Read-only" claims backed by enforcement OR labeled as intent-only; defense-in-depth present     |

**Pass threshold: overall score >= 7.0** (base dimensions). When AI-production dimensions are triggered, they must each also score >= 7.0 — they cannot be averaged into the base to mask a weak spot.

### Convergence preservation (KP-3684)

When this panel runs alongside other reviewers (sh:ai-panel, sh:mobile-panel, Codex/Gemini duo, human reviewers), DO NOT synthesize convergent findings into a single voice. Preserve verbatim:

```
Finding F-N: <one-line summary>
  - sh:spec-panel (Fowler): "<verbatim quote>"
  - sh:ai-panel (Karpathy): "<verbatim quote>"
  - Convergence: 2/2 independent reviewers
  - FIPD action: <Fix|Investigate|Plan|Decide>
```

The synthesizer's job is to *count and tag* convergence, not collapse it.

Output includes per-dimension scores, overall score, critical issues, expert consensus points, and an improvement roadmap (immediate / short-term / long-term).

## Output

Specification review document containing:
- Multi-expert analysis with distinct perspectives
- Per-dimension scores and overall quality score
- Pass/fail gate result
- Critical issues with severity and priority
- Consensus points and disagreements
- Priority-ranked improvement recommendations

**SYNTHESIS ONLY** — this panel produces analysis and recommendations. It does not modify the specification without explicit instruction.
