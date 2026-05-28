---
name: sh-test-panel
description: "Multi-expert test-strategy review — coverage, fixture quality, AC testability, regression discipline"
---

# /sh:test-panel — Expert Test Strategy Review Panel

## Usage

```
/sh:test-panel [content|@file] [--mode discussion|critique|socratic|debate] [--focus coverage|fixtures|acceptance|regression] [--experts "name1,name2"] [--iterations N]
```

## Behavioral Flow

1. **Load Panel Config**: Read `/Users/jcords-macmini/projects/20_agentflow/experts/panels/test-panel.yaml` for panel definition, focus areas, auto-select rules, and scoring config (absolute path — relative paths fail when CWD is outside agentflow).
2. **Load Experts**: Read expert files from `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/` for each selected expert.
3. **Auto-Select Experts**: Scan the content (spec / test plan / diff) against panel YAML `auto-select` keywords — add matching experts up to `max-experts: 6` cap.
4. **Pre-Scoring Checks (deterministic — do NOT delegate to LLM personas):**
    - **AC-to-test mapping:** If the content lists numbered ACs (AC-01, AC-02...), enumerate them and check that each AC is referenced by at least one named test (filename + test function). Unmapped ACs become CRITICAL findings before deliberation. Rationale: an AC without a falsifying test is a process AC at best and a hallucinated guarantee at worst.
    - **Mock/real-boundary inventory:** Scan the test plan for `mock`, `monkeypatch`, `stub`, `fake`. For each, note whether the boundary being mocked is an external dependency (justified) or an internal collaborator (potential test-isolation smell). Flag inappropriate internal mocks as MAJOR.
5. **Analyze**: Parse content, identify gaps in coverage, fixture rot, regression-prevention gaps, and AC testability issues.
6. **Assemble Panel**: Select experts based on `--focus` area or use `default-experts` from panel YAML. `--experts` override replaces defaults entirely.
7. **Conduct Review**: Run analysis in the selected mode using each expert's distinct methodology.
8. **Score**: Rate the test strategy across 4 dimensions (0-10 each), compute overall score.
9. **Gate Check**: Overall score must be >= 7.0 to pass.

## Expert Loading

Experts are defined as individual markdown files in `/Users/jcords-macmini/projects/20_agentflow/experts/individuals/`. Each file contains structured frontmatter with:
- Domain and specialization
- Methodology and frameworks
- Critique focus and typical questions

The panel YAML (`/Users/jcords-macmini/projects/20_agentflow/experts/panels/test-panel.yaml`) defines:
- Which experts belong to which focus area
- Who leads each focus area
- Auto-select keyword rules for dynamic expert addition
- Scoring dimensions and pass threshold

## Analysis Modes

### Discussion Mode (`--mode discussion`)
Collaborative test-strategy refinement. Experts build on each other's observations — coverage gaps surface fixture-design questions; AC testability gaps surface regression-discipline gaps.

### Critique Mode (`--mode critique`)
Systematic review with severity-classified findings (CRITICAL / MAJOR / MINOR). Each finding includes expert attribution, the testing assumption it challenges, the failure mode left unguarded, and a concrete recommendation.

### Socratic Mode (`--mode socratic`)
Foundational questioning — "what does this test prove?", "what would a green test miss?", "what does a failure here look like in CI?". No direct answers; forces the author to defend the test design.

### Debate Mode (`--mode debate`)
Two-camp adversarial format around a contested testing trade-off (e.g. mock-heavy vs integration-heavy, unit-vs-e2e ratio). Three rounds, then a synthesized trade-off statement.

## Focus Areas

- **coverage**: Test scope across unit/integration/e2e, gap analysis, hot-path identification, mutation thinking. Lead: Lisa Crispin. Experts: Crispin, Gregory, Adzic.
- **fixtures**: Fixture quality, labeled-data design, deterministic seeds, drift detection, fixture rot. Lead: Gojko Adzic. Experts: Adzic, Crispin.
- **acceptance**: AC testability, spec-by-example, falsifiability by single failing assertion. Lead: Gojko Adzic. Experts: Adzic, Wiegers, Crispin.
- **regression**: Regression discipline, flaky-test prevention, CI/CD gating, broken-main recovery. Lead: Janet Gregory. Experts: Gregory, Crispin, Nygard.

## Scoring Gate

4 dimensions, each scored 0-10:

| Dimension                | Description                                                                                                  |
|--------------------------|--------------------------------------------------------------------------------------------------------------|
| AC testability           | Can each AC be falsified by a single failing pytest assertion; are process ACs separated from machine-enforceable ACs |
| Coverage completeness    | Is the test plan complete across unit/integration/regression; are the right boundaries mocked vs real        |
| Fixture quality          | Are labeled fixtures present where correctness ACs require them; are fixture drift and rot addressed         |
| Regression discipline    | Does the plan prevent re-introducing prior bugs; are CI gates and failure-recovery paths specified           |

**Pass threshold: overall score >= 7.0**

## Output

Test strategy review document containing:
- Multi-expert analysis with distinct perspectives
- Per-dimension scores and overall quality score
- Pass/fail gate result
- Critical issues with severity and testing assumption challenged
- AC-to-test mapping table (from pre-scoring check)
- Consensus points and disagreements
- Priority-ranked improvements

**SYNTHESIS ONLY** — this panel produces analysis and recommendations. It does not author or modify test code without explicit instruction. Per `00_Governance/CLAUDE.md §8 Panel Auto-Fix Policy`, downstream auto-fix consumers MAY apply the recommendations across all severities; this skill itself surfaces them.
