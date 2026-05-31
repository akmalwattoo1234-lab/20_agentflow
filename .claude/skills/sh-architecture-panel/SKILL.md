---
name: sh-architecture-panel
description: "Multi-expert architecture review — boundaries, integration patterns, failure modes, evolvability"
---

# /sh:architecture-panel — Expert Architecture Review Panel

## Usage

```
/sh:architecture-panel [content|@file] [--mode discussion|critique|socratic|debate] [--focus boundaries|integration|reliability|evolution] [--experts "name1,name2"] [--iterations N]
```

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/architecture-panel.yaml` for panel definition, focus areas, auto-select rules, and scoring config (absolute path — relative paths fail when CWD is outside agentflow).
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert.
3. **Auto-Select Experts**: Scan the content against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 6` cap.
4. **Analyze**: Parse architecture content (spec / diff / module diagram), identify boundaries, contracts, integration seams, failure-handling gaps.
5. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts` from panel YAML. `--experts` override replaces defaults entirely.
6. **Conduct Review**: Run analysis in the selected mode using each expert's distinct methodology.
7. **Score**: Rate the design across 4 dimensions (0-10 each), compute overall score.
8. **Gate Check**: Overall score must be >= 7.0 to pass. Below threshold = design needs rework.

## Expert Loading

Experts are defined as individual markdown files in `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/`. Each file contains structured frontmatter with:
- Domain and specialization
- Methodology and frameworks
- Critique focus and typical questions

The panel YAML (`/Users/jcords-macmini/projects/20_agentflow/experts/panels/architecture-panel.yaml`) defines:
- Which experts belong to which focus area
- Who leads each focus area
- Auto-select keyword rules for dynamic expert addition
- Scoring dimensions and pass threshold

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Collaborative architecture refinement. Experts build on each other's observations sequentially — boundary critique flows into integration concerns flow into failure-mode review.

### Critique Mode (`--mode critique`)
Systematic review with severity-classified findings (CRITICAL / MAJOR / MINOR). Each finding includes expert attribution, the architectural assumption it challenges, a concrete recommendation, and the cost of leaving it unfixed.

### Socratic Mode (`--mode socratic`)
Foundational questioning — "what's the seam?", "where does this fail?", "how does this change look in two years?". No direct answers; forces the author to defend the design.

### Debate Mode (`--mode debate`)
Two-camp adversarial format. Experts split into pro-design and skeptic positions, exchange three rounds, and the panel runner synthesizes the disagreement into explicit trade-off statements.

## Focus Areas

- **boundaries**: Module/service boundaries, contracts, coupling, cohesion, dependency direction. Lead: Sam Newman. Experts: Newman, Fowler, Hohpe.
- **integration**: Inter-service patterns, message exchange, eventual consistency, idempotency. Lead: Gregor Hohpe. Experts: Hohpe, Newman, Nygard.
- **reliability**: Failure modes, circuit breakers, bulkheads, timeouts, degradation paths. Lead: Michael Nygard. Experts: Nygard, Newman, Hohpe.
- **evolution**: Change cost, versioning strategy, refactor paths, abstraction maturity. Lead: Martin Fowler. Experts: Fowler, Newman.

## Scoring Gate

4 dimensions, each scored 0-10:

| Dimension                  | Description                                                                                              |
|----------------------------|----------------------------------------------------------------------------------------------------------|
| Boundary clarity           | Are module/service boundaries explicit, with one-way dependency arrows and unambiguous contracts at each seam |
| Integration correctness    | Are integration patterns appropriate; idempotency, ordering, and delivery guarantees handled explicitly |
| Failure-mode coverage      | Are failure modes named and addressed; timeouts, retries, circuit-breakers, fallbacks specified where needed |
| Evolvability               | Can the design absorb foreseeable change without rewrites; versioning and migration paths exist          |

**Pass threshold: overall score >= 7.0**

## Output

Architecture review document containing:
- Multi-expert analysis with distinct perspectives
- Per-dimension scores and overall quality score
- Pass/fail gate result
- Critical issues with severity and architectural assumption challenged
- Consensus points and disagreements (with explicit trade-off statements in debate mode)
- Priority-ranked improvement recommendations

**SYNTHESIS ONLY** — this panel produces analysis and recommendations. It does not modify the code or design without explicit instruction. Per `00_Governance/CLAUDE.md §8 Panel Auto-Fix Policy`, downstream auto-fix consumers MAY apply the recommendations across all severities; this skill itself surfaces them.

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
