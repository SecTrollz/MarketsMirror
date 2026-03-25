# MarketsMirror UI Re-Approach Plan

## 1) Current-State Audit (Why the UI feels brittle)

The current app is implemented as a single monolithic `index.html` containing:

- Full layout/visual styling.
- All runtime state and orchestration logic.
- Multiple analysis engines and report-generation flows.
- Renderers for chat, charts, diagnostics, insights, and reveal screens.

This creates a **high-coupling failure mode**: small style or scripting edits can break unrelated flows.

### Primary failure hotspots

1. **Single-file complexity risk**
   - CSS, markup, and JavaScript all live in one file.
   - Long-term maintainability and onboarding cost is high.

2. **Mixed responsibilities in runtime flow**
   - Session progression, NLP analysis, UI rendering, and integration logic are intertwined.
   - Hard to test or debug in isolation.

3. **Layout fragility across breakpoints**
   - Desktop + mobile behavior depends on many layered media queries and dynamic height handling.
   - Visual regressions are likely when changing panel structure.

4. **Feature-surface breadth without module boundaries**
   - Chat, psychographic scoring, sentiment charting, model controls, educational report generation, and reveal state all share global context.

5. **Regression risk from direct DOM coupling**
   - Many features bind directly to concrete element IDs.
   - Renaming/reorganizing markup can silently break behavior.

---

## 2) Re-Architecture Goals

1. **Preserve full functionality** (no feature drops).
2. **Stabilize the UI shell** so layout and interaction bugs are easier to isolate.
3. **Create explicit module boundaries** between:
   - Domain logic (analysis/scoring)
   - Session orchestration (turn handling, phase transitions)
   - UI rendering (chat, charts, panels)
4. **Enable incremental QA** with deterministic checks.
5. **Support future extensibility** (new insight modes, additional visualizations, additional model backends).

---

## 3) Target Organization (Feature-complete, modular)

## 3.1 File/Folder structure

```text
/src
  /core
    app-state.js              # canonical state container
    session-controller.js     # begin/send/continue/restart orchestration
    event-bus.js              # lightweight pub/sub

  /analysis
    sentiment.js              # VADER + scoring utilities
    lexicon-engine.js         # lexicon matching / evidence mapping
    bayes-profiler.js         # accumulators and uncertainty
    multimodal-engine.js      # ONNX/HF integration wrappers
    lingua-insight.js         # educational report pipeline

  /render
    render-chat.js            # chat timeline/input/status renderers
    render-charts.js          # radar/sentiment chart adapters
    render-frameworks.js      # 28-axis framework panel
    render-insights.js        # 35-point customer grid
    render-reveal.js          # completion/reveal screen
    render-status.js          # header + diagnostics status

  /ui
    dom-map.js                # centralized ID/class references
    actions.js                # UI action binding
    responsive.js             # viewport/safe-area management

  /styles
    tokens.css
    base.css
    layout.css
    components.css
    responsive.css

  /config
    frameworks.js
    question-bank.js
    defaults.js

  main.js
index.html
```

## 3.2 Core design principles

- **Single source of truth state**: no hidden state in render modules.
- **Unidirectional flow**: user action -> controller -> state update -> render.
- **Render as pure functions** when possible (state in, DOM out).
- **Adapter boundaries** for external dependencies (Chart.js, ONNX, HF).
- **Defensive null-safe DOM access** and explicit lifecycle hooks.

---

## 4) Functional Mapping (Current features -> Target modules)

| Current capability | Target module(s) | Notes |
|---|---|---|
| Intro/start flow | `session-controller`, `render-chat`, `actions` | Keep existing begin-session UX with cleaner state transitions. |
| Turn-based chat | `render-chat`, `session-controller` | Maintain message roles, thinking state, turn counter. |
| 9 frameworks / 28 axes | `config/frameworks`, `analysis/bayes-profiler`, `render-frameworks` | Preserve scoring semantics and evidence display. |
| Radar + sentiment charts | `render-charts` | Isolate Chart.js lifecycle and resizing handling. |
| VADER and lexical features | `analysis/sentiment`, `analysis/lexicon-engine` | Keep current behavior, improve testability. |
| Multimodal status + ONNX/HF | `analysis/multimodal-engine`, `render-status` | Keep optional deep NLP path and in-memory token handling. |
| Customer 35-point insight map | `analysis/lingua-insight`, `render-insights` | Preserve export/download JSON actions. |
| Reveal/final summary | `render-reveal`, `session-controller` | Preserve continue/new-session behavior. |
| Responsive mobile behavior | `ui/responsive`, `/styles/responsive.css` | Consolidate viewport/safe-area logic and breakpoint rules. |

---

## 5) Implementation Plan (Phased, low-risk)

### Phase 0 — Safety Net (No UX changes)

- Add baseline smoke checks:
  - App boots without console errors.
  - Session can run 0 -> 17 turns.
  - Reveal screen appears and actions work.
  - Charts render and update.
  - JSON copy/download buttons work.
- Capture desktop + mobile screenshots for baseline diffs.

### Phase 1 — Structural split (No behavior changes)

- Extract CSS into `/styles/*` keeping same selectors.
- Extract JavaScript into `/src/main.js` and grouped modules.
- Introduce `dom-map.js` to centralize selectors.
- Keep function signatures stable to reduce breakage.

### Phase 2 — State and orchestration hardening

- Introduce `app-state` container with immutable update helpers.
- Move all turn/phase transitions into `session-controller`.
- Emit typed events (`TURN_SUBMITTED`, `PROFILE_UPDATED`, `SESSION_COMPLETED`).

### Phase 3 — Renderer isolation

- Move each visual area into dedicated renderer modules.
- Ensure renderers never mutate analysis state directly.
- Add resilient re-render logic for hidden/visible sections.

### Phase 4 — Analysis isolation

- Extract sentiment, lexicon, Bayes, and insight generation into `/analysis`.
- Add deterministic fixture tests for representative responses.
- Add adapter wrappers for ONNX/HF so fallback behavior is explicit.

### Phase 5 — UX quality pass

- Accessibility pass:
  - semantic regions, aria-live for message stream updates,
  - keyboard navigation and focus states,
  - contrast checks for key status elements.
- Responsive pass:
  - tune panel heights, chart constraints, and input ergonomics.

### Phase 6 — Stabilization and release

- Run full manual regression matrix (desktop/mobile).
- Compare output parity (scores/insights/reveal) against baseline fixtures.
- Ship with changelog and rollback tag.

---

## 6) Regression Checklist (must pass before finalizing)

1. Start session from intro.
2. Submit >= 5 turns and verify:
   - chat history renders,
   - turn counter increments,
   - status metrics update.
3. Verify framework panel bars and evidence rendering.
4. Verify radar + sentiment charts update without layout overflow.
5. Toggle LinguaInsight modes and confirm report/meta updates.
6. Copy JSON + download JSON from insights section.
7. Reach reveal state and verify continue/restart actions.
8. Test mobile viewport (<= 820px) for:
   - readable text,
   - input usability,
   - no panel clipping.

---

## 7) Definition of Done for “Complete UI Finalization”

The UI re-approach is complete when:

- All current features remain functional.
- Codebase is modularized into clear ownership boundaries.
- Critical user journeys pass regression checks on desktop and mobile.
- No blocker console errors during full session flow.
- New contributors can locate logic by concern in under 5 minutes.

---

## 8) Immediate next step (recommended)

Execute **Phase 0 + Phase 1** first, in one PR, with strict “no behavior change” scope. This gives the biggest stability gain while minimizing risk to profiling functionality.


## 9) March 2026 Execution Update

- Runtime JavaScript has been moved back to `src/main.js` and is loaded from `index.html` via a single deferred script tag.
- `src/main.js` is now kept in parity with the previously inline scripts to reduce bootstrap breakage risk.
- ONNX initialization includes visible progress feedback (spinner + progress bar + label + percentage).
- `styles/app.css` now includes matching `.mm-loader*` styles so extracted-CSS migrations remain consistent.
