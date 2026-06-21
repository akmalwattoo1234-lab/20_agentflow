---
name: sh-ux-design
description: "Build clickable wireframe prototypes in code to validate flow and structure before hi-fi design"
---

# /sh:ux-design — Wireframe Prototyping in Code

## Usage

```
/sh:ux-design [flow_source] [--fidelity lo|mid] [--nav click-through|functional] [--concept-b "description"]
```

**Announce at start:** "I'm using the sh:ux-design skill to build a clickable wireframe prototype."

<HARD-GATE>
Do NOT apply production styling, custom icons, real images, animations, or design polish. This is a WIREFRAME. The entire point is to validate structure and flow before investing in hi-fi. If the output looks polished, you've gone too far. Read `wireframe-rules.md` before generating any markup.
</HARD-GATE>

## Why Wireframes in Code

Static wireframes in Figma require your team to imagine clicking through. Code wireframes let them actually test the flow. And because they're in the codebase, graduating to hi-fi is incremental — you build on top, not start over.

## Checklist

You MUST complete these steps in order:

1. **Ingest flow source** — pull user flow from Figma, Miro, or PRD
2. **Read strategic context** — check for `/sh:business-panel` output (JTBD, ICP, risk flags)
3. **Enter planning mode** — clarify fidelity, nav model, scope, stack
4. **Generate wireframe** — in isolated worktree, target stack, applying wireframe constraints
5. **Smoke test** — Playwright click-through every step
6. **Iterate** — refine based on walkthrough findings
7. **Concept branch** (optional) — alternative interaction pattern
8. **Compare & decide** (if multiple concepts) — structured comparison, pick winner
9. **Graduation gate** — checklist before handoff to `/frontend-design`

## Phase 1: Ingest

Pull the user flow that defines what screens and transitions to wireframe.

**Source priority:**
1. Figma/FigJam URL → Figma MCP `get_figjam` or `get_design_context`
2. Miro board URL → Miro MCP `context_get` or `board_list_items`
3. PRD markdown → extract flow steps from user stories

**Output:** List of screens with transitions (e.g., "Create Account → [continue] → Select Role → [skip|connect] → Connect Calendar")

## Phase 2: Strategic Context

Check if a `/sh:business-panel` analysis exists for this feature. Look in `docs/` for recent business-panel output.

**If found, extract constraints:**
- **JTBD** → wireframe must demonstrate user can complete this job
- **ICP** → complexity ceiling (non-technical user? keep steps ≤ 5)
- **Risk flags** → mandatory fallback paths (e.g., "if OAuth fails, user isn't stuck")
- **Competitive benchmark** → step count ceiling ("competitors need 8 steps, we need ≤ 5")

**If not found:** Proceed without. Suggest running `/sh:business-panel` if the feature is strategic.

## Phase 3: Plan

Enter planning mode. Ask clarifying questions — **one at a time:**

1. **Fidelity:** Lo-fi grayscale or mid-fi with brand colors?
2. **Navigation:** Click-through only or functional forms/inputs?
3. **Style:** Classic wireframe (hand-drawn feel) or minimal (clean boxes)?
4. **Scope:** All flow steps or a subset? Which steps are highest risk?
5. **Stack:** What framework does the target project use? (React, Jinja2, plain HTML, etc.)

**Output:** Wireframe plan saved to `docs/plans/wireframe-<feature>.md` using `/sh:plan` format.

## Phase 4: Generate

### Setup
- `/sh:worktree` — create isolated branch `wireframe/<feature>`
- Read `wireframe-rules.md` — apply ALL visual constraints

### Build
Generate clickable prototype in the **target project's stack** (not generic HTML). Each flow step becomes a navigable screen.

**MCP tools available:**
- Figma MCP `get_variable_defs` — pull design tokens if mid-fi fidelity
- Context7 MCP `query-docs` — component library docs for the target stack

### Verify
After generation, run Playwright smoke test:
- Every screen renders without errors
- Every primary action navigates to the next step
- No dead ends (every path reaches completion or has explicit skip/back)

**Report:** "Wireframe ready — N screens, N transitions, all clickable. View at [local URL]."

## Phase 5: Iterate

User walks through the wireframe. Common feedback patterns:

- "Too many steps" → merge screens, reduce clicks-to-completion
- "This step needs more detail" → add inline widget (availability picker, calendar, etc.)
- "The order feels wrong" → resequence without restructuring
- "I don't like step N" → revise single screen

**After each iteration:**
1. Apply changes
2. Re-run Playwright smoke test (cheap — catches dead ends immediately)
3. `/sh:review` — self-check wireframe still matches original user flow

## Phase 6: Concept Branching (Optional)

When user wants an alternative approach: "What if this was chat-based instead?"

1. `/sh:worktree` — new branch `wireframe/<feature>-concept-b`
2. Same wireframe constraints, different interaction pattern
3. Both concepts get their own route (`/wireframe-a`, `/wireframe-b`) OR separate worktrees
4. Playwright screenshots of both for comparison

**Max concepts:** 3. More than 3 = scope problem, not a wireframing problem.

## Phase 7: Compare & Decide

When multiple concepts exist, generate comparison. Read `concept-compare.md` for the template.

**Comparison dimensions:**
- Step count / clicks-to-completion
- Form count and input complexity
- Drop-off risk per step
- JTBD fit (does the job get done?)
- ICP match (complexity appropriate for target user?)

**Optional — strategic validation:**
```
/sh:business-panel docs/wireframe-comparison.md --focus growth --mode debate
```
Panel debates which concept better serves strategy. Christensen (JTBD fit), Godin (adoption friction), Taleb (what breaks), Meadows (system leverage).

**Decision:** User picks winner. Merge winning branch. Delete losing branches. Clean up worktrees via `superpowers:finishing-a-development-branch`.

## Phase 8: Graduation Gate

Before handing off to `/frontend-design`, ALL must pass. Read `handoff.md` for details.

```
□ All steps clickable, no dead ends (Playwright evidence)
□ Single concept chosen, variant branches cleaned up
□ Step count final (structural freeze — no adding/removing screens after this)
□ Interaction model locked (click-through vs functional vs chat-based)
□ Component mapping drafted (wireframe placeholder → target component)
□ JTBD validated (if business-panel brief exists)
```

**Use `/sh:verify`** — evidence before claiming ready.

**Graduation output:**
> "Wireframe approved and frozen. N screens, N transitions. Component mapping in `docs/wireframe-<feature>-handoff.md`. Ready for `/frontend-design`."

**The terminal state is invoking `/frontend-design`.** The wireframe files stay in place. Hi-fi builds ON TOP — replacing styles and placeholders with real components while preserving flow structure.

## Integration

| Skill | Relationship |
|-------|-------------|
| `/sh:business-panel` | Upstream — strategic constraints feed into Phase 2 |
| `/sh:brainstorm` | Upstream — produces PRD and user flow |
| `/sh:spec-panel` | Upstream — validates spec quality before wireframing |
| `/sh:plan` | Called in Phase 3 — wireframe plan uses same doc format |
| `/sh:worktree` | Called in Phase 4+6 — isolation per concept |
| `/sh:review` | Called in Phase 5 — self-review wireframe against flow |
| `/sh:verify` | Called in Phase 8 — evidence-based graduation check |
| `superpowers:finishing-a-development-branch` | Called after concept decision — cleanup losing branches |
| `/frontend-design` | Downstream — builds hi-fi on wireframe foundation |

## Key Principles

- **Ugly on purpose** — if it looks polished, you skipped the wireframe stage
- **Validate structure, not aesthetics** — catch flow problems before they're expensive
- **One primary action per screen** — if a screen has two CTAs, it needs splitting
- **Dead ends are bugs** — every path must reach completion or have explicit escape
- **Cheap iteration** — Playwright tests every time, not just at the end
- **Build on top, not over** — graduation preserves structure, replaces surface
