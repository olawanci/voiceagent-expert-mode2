# VoiceAgent Advanced Mode

Design prototype + documentation for CloudTalk's VoiceAgent Advanced Mode feature — a JSON-editable view of agent configuration alongside the existing Form-based wizard.

## What's in this folder

### `prototype/`

Interactive HTML prototypes you can open in any modern browser.

- **`voiceagent-tessa-edit.html`** — the current prototype. Form / Advanced toggle, editable JSON editor, validation + repair panel, save flow with diff modal, fallback application, paste-detection notice, first-time onboarding banner, Duplicate / Test / Delete actions. Open this one first.
- `voiceagent-tessa-toggle.html` — earlier iteration: Form / Advanced toggle with read-only JSON view (no editing).
- `voiceagent-tessa-import.html` — earlier iteration: read-only side panel + import flow.
- `voiceagent-tessa.html` — earliest iteration: Form / Expert toggle with editable Expert mode (tabs for Config / Compiled prompt / Diff).

All earlier prototypes are kept for reference and design evolution context.

### `docs/`

- **`voiceagent-advanced-handoff.md`** — design handoff with rationale for each major decision (why toggle vs side panel, why textarea vs Monaco, why explicit save vs auto-save, etc.). Read this for the "why."
- **`voiceagent-acceptance-criteria.md`** — testable acceptance criteria organized by epic, with MUST/SHOULD/MAY language. Use this as the engineering contract.
- **`voiceagent-expert-user-stories.md`** — user stories with acceptance criteria, organized by persona (PLG, technical customer, TC consultant) and phase. Read this for the "who and what."
- **`config-breaking-use-cases.md`** — ~60 use cases for how users can invalidate or break the JSON config, with severity and recommended system response. Use this as the validation test plan and risk register.

### `assets/`

- **`voiceagent-json-syntax.svg`** — exportable SVG of the syntax-highlighted JSON editor view, ready to drop into Figma.

## How to use

1. **For design review** — open `prototype/voiceagent-tessa-edit.html` in a browser. Toggle between Form and Advanced. Try the test scenarios in the handoff doc.
2. **For engineering scoping** — read `docs/voiceagent-advanced-handoff.md` (architecture decisions) and `docs/voiceagent-acceptance-criteria.md` (testable AC). The "Open questions" section at the end of the handoff lists items needing engineering input before implementation begins.
3. **For QA** — `docs/voiceagent-acceptance-criteria.md` includes 12 regression scenarios that must pass before release. `docs/config-breaking-use-cases.md` covers validation edge cases.
4. **For Figma** — drop `assets/voiceagent-json-syntax.svg` onto a canvas. Each line is a separate text element (line numbers + code spans), editable post-import.

## Personas served

- **Group 1 — Non-technical PLG users.** Form view only. Never see JSON unless they toggle.
- **Group 2 — Technical customers (sales-assisted, partners).** Paste configs from Claude. Edit JSON when the form doesn't expose the parameter they need.

TC consultants (Group 3) get value but are not the primary target.

## Design principles (summary)

1. Form is primary, Advanced is opt-in.
2. Transparency over magic — fallbacks visible before save, diffs reviewable before commit.
3. Recoverable by default — every state has a clear way back.
4. Schema is the contract — versioned, documented, stable.
5. Match Form view's UX patterns where possible — toggle, modals, etc.

See `docs/voiceagent-advanced-handoff.md` for the full rationale per decision.

## Out of scope

Items deliberately deferred to later phases:
- Monaco editor (Phase 3+)
- AI-mediated config repair / Composer (Phase 3+)
- Schema-driven autocomplete on enums (requires Monaco or equivalent)
- Hover docs on schema fields (requires Monaco or equivalent)
- Version history view with one-click revert (Phase 2.5)
- Concurrent edit conflict resolution UI (Phase 2.5)
- Compiled prompt preview tab (Phase 3+)
- Save as template (Phase 3+)

## Versioning

Current schema referenced in the prototype: `voiceagent@1.4`.
