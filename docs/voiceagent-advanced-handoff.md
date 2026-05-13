# VoiceAgent Advanced mode — design handoff

This document captures the design decisions for the VoiceAgent setup's Advanced mode (Phase 1: read-only inspector + import; Phase 2: editable JSON). Read this before implementing. The prototype at `voiceagent-tessa-edit.html` is the visual reference; this doc explains *why* each piece is shaped the way it is.

---

## 1. Context

CloudTalk's VoiceAgent setup is a form-driven wizard. Some users — specifically technical customers and partners — want more control than the form exposes. Today's workaround is copy-paste between the form and ChatGPT/Claude, which is slow and doesn't cover parameters that aren't surfaced in the form at all.

Advanced mode is the answer: a JSON-based view that lives alongside the form, exposing the full configuration document. Phase 1 makes it read-only with a paste-to-import flow; Phase 2 makes it editable in place.

## 2. Personas

The product serves two personas. A third — TC consultants — gets value but is not optimized for.

**Group 1 — Non-technical PLG users.** Should never see JSON unless they deliberately toggle to it. The Advanced view should feel like something they *could* explore if curious but never *have* to. Their full experience is the form.

**Group 2 — Technical customers (sales-assisted, partners).** Know what they want, can paste from Claude. Want to make complex changes the form doesn't expose (model selection, voice pacing, etc.) or apply a known-good config from another agent.

**Group 3 — TC consultants.** Set up agents on behalf of multiple customers. Cross-account porting workflow. Not the primary target — anything that serves them serves Group 2 too.

## 3. Design principles

1. **Form is primary, Advanced is opt-in.** Form is the default landing experience. Advanced is one click away but never imposed.
2. **Transparency over magic.** Users see what's happening. Fallbacks are visible before save, diffs are reviewable before commit.
3. **Recoverable by default.** Every state has a clear way back. Revert is always available; no destructive irrecoverable actions.
4. **Schema is the contract.** The JSON shape is versioned, documented, and treated as a stable API customers can depend on.
5. **Match the form's UX patterns.** The floating save bar in Advanced mirrors the Form view's pattern. The toggle is symmetric. Consistency over local optimization.

---

## 4. Architecture decisions

This section documents the key design decisions and why each won over alternatives.

### 4.1 Toggle pattern, not side panel

**Decision.** Form and Advanced are two distinct views of the same agent, selected via a centered pill toggle (`Form` / `</> Advanced`) sitting above all content. Switching between them is a mode change.

**Alternatives considered.**

- *Side panel pushed onto Form view* — like VAPI's "Assistant Configuration" panel. Lets the user see Form and JSON simultaneously.
- *Modal overlay* — opens JSON as a focused dialog over the Form. Used for one-shot inspection.
- *Persistent dev mode* — like Figma's Dev Mode. Toggle into a different working mode with its own chrome.

**Why toggle won.** The compare-form-and-JSON-at-once workflow is valuable only for TC consultants, who aren't our primary target. Group 1 never needs it; Group 2 reads JSON in Claude, not in our UI. Without the compare requirement, a panel adds chrome without delivering value. The toggle pattern is also forward-compatible: when Phase 2 makes JSON editable, the toggle stays unchanged — the `</>` view just gains a cursor and validation.

**Trade-off accepted.** TCs lose the side-by-side comparison they'd benefit from. Mitigated by the fact that Copy/Import handle their core cross-account workflow.

### 4.2 Centered toggle, above all cards

**Decision.** The toggle row sits above the agent header card, centered horizontally.

**Alternatives considered.** Embedded in the agent header card (right side, next to Test/Delete) — less prominent, feels like an action rather than primary navigation.

**Why centered-above won.** The toggle is the first decision the user makes about how to view the agent. Putting it above the agent identity card reinforces "mode selector, not an action." Centered placement reads as primary nav rather than a tool tucked into a corner. The pattern matches Claude's artifact toggle.

### 4.3 Phase 1 ships read-only inspector + import; Phase 2 adds editable JSON

**Decision.** Phase 1: Advanced view is a read-only inspector with Copy and Import buttons. All editing happens through (a) the form, or (b) the import-modal paste flow.

**Alternatives considered.** Ship editable JSON immediately (Retell pattern).

**Why phased won.** Several risks aren't validated yet — desirability of JSON editing, schema-as-contract commitments, COGS impact of exposing model selection. Read-only inspector solves ~80% of the technical-user need (see the config, port it across accounts) with ~20% of the risk. Editable JSON ships in Phase 2 after the schema stabilizes and the pricing decisions are made.

**Trade-off accepted.** Group 2 still needs a round-trip through Claude or a text editor to make in-place edits. Acceptable because they're already doing this workflow externally today.

### 4.4 Textarea + syntax overlay, not Monaco

**Decision.** The JSON editor uses a transparent `<textarea>` over a syntax-highlighted `<pre>` element, with a custom line-number gutter alongside. No Monaco, no CodeMirror.

**Alternatives considered.** Monaco editor with full JSON schema language service (autocomplete, hover docs, multi-error reporting).

**Why textarea won (for this prototype and likely Phase 2 launch).** The user explicitly preferred the simpler textarea over Monaco — Monaco's chrome and density felt intimidating for non-engineers. Monaco gives multi-error reporting and autocomplete; the textarea gives approachability and a smaller bundle. For Phase 2 launch, ship textarea; revisit Monaco when feature parity with Retell becomes the gating factor.

**Trade-off accepted.** Only one syntax error visible at a time (JSON parser limitation). No autocomplete on enum values. No hover docs. Recovery is straightforward (revert), and the simpler editor reads as friendlier.

### 4.5 Line-level diff highlighting with multiple cues

**Decision.** Modified lines get a soft blue background tint + blue accent bar on the left + blue marker in the gutter. Added lines get green equivalents. Error lines get red, taking precedence over diff colors.

**Alternatives considered.** Gutter markers only (less visible), full git-style diff with old/new side-by-side (too heavy).

**Why this won.** Three reinforcing cues (gutter dot, line background, accent bar) mean changes catch the user's eye regardless of where they're looking in the editor. Error precedence ensures users see the most actionable thing first. Background tint covers the full line so the cue is visible even when reading the line content.

### 4.6 Repair panel with quick-fix buttons (not inline VS Code lightbulbs)

**Decision.** Errors and warnings surface in a panel below the editor's status strip, each with click-to-fix buttons. The panel scrolls if there are many issues.

**Alternatives considered.** Inline lightbulb at the affected line (VS Code style), modal review of all issues.

**Why panel won.** The textarea doesn't support per-line affordances cleanly. The panel is more discoverable for non-experts who don't know the lightbulb convention. Click-to-jump-to-line ties the panel back to the editor location. With 3+ errors, a scrolling list is more usable than three lightbulbs scattered across lines.

### 4.7 Floating save bar, not top-right buttons

**Decision.** Save and Discard live in a bottom-centered floating bar (dark, with amber section list), matching the pattern already used in the Form view. No Save/Discard buttons in the editor card header.

**Alternatives considered.** Save/Discard in the editor toolbar at top-right (the prototype's earlier state).

**Why floating bar won.** Consistency with the Form view is the primary reason. The bar is always visible when there are pending changes regardless of which view you're in or how far you've scrolled. The amber section list (`Unsaved changes in Identity, Voice…`) tells the user *what* changed without needing to scroll the editor or open the diff modal.

### 4.8 Explicit save with diff review modal, not auto-save

**Decision.** Save is an explicit action. Clicking Save changes opens a modal with the field-level diff for review; the user clicks Save changes again in the modal to commit.

**Alternatives considered.** Google Docs-style auto-save, save-and-publish split (save = draft, publish = live).

**Why explicit save won.** Voice agents serve real customers in real calls. Every save can affect live behavior. Auto-save makes it too easy to ship a bad change. The diff modal is the safety gate.

**Trade-off accepted.** More clicks. Acceptable because save is a deliberate action, not a frequent operation.

### 4.9 Save modal shows semantic field-level diff, not text diff

**Decision.** The save modal lists changes as `voice.speed: 1.0 → 1.15`, not as raw text line diffs.

**Why semantic won.** A user reviewing a save shouldn't have to mentally parse a JSON text diff to understand what's changing. Semantic diffs abstract away whitespace and ordering, presenting the actual *value* changes. More meaningful for non-experts.

**Trade-off accepted.** Formatting-only changes (indentation, key order) don't appear in the diff — the modal says "no semantic changes" even if the user reformatted the document. This is correct behavior; the saved canonical form re-stringifies anyway.

### 4.10 Save applies soft-required fallbacks before committing

**Decision.** If a soft-required field (e.g., `identity.languages`) is empty or missing, the system applies a documented default on save. The save modal shows what fallbacks will be applied in a blue info strip above the diff.

**Alternatives considered.** Backend silently injects fallbacks (no transparency), or block save until user explicitly fills in the field (no flexibility).

**Why client-side-with-transparency won.** Transparency: the user sees `identity.languages → ["en-US"]` in the modal before confirming. Predictability: the same defaults apply whether saved via Advanced or via Form. User agency: they can override the default via a quick-fix in the repair panel, or accept it implicitly by saving.

### 4.11 Import is a separate commit flow, not "load into editor"

**Decision.** Clicking Import opens a modal with paste → validate → diff → apply. Apply commits directly as the new saved version (bumps version, fires toast). It does not load into the editor as pending edits.

**Alternatives considered.** Import pastes into the editor; user reviews in-context and clicks Save normally.

**Why direct-commit won.** Import is a discrete action with its own review surface (the diff in the modal). Going through the editor and then through the save modal would mean two diff reviews, which is overkill. The import flow is its own commit path.

**Subtle behavior.** If the user has pending Advanced-mode edits when they click Import, the modal shows a yellow notice: "You have N unsaved edits that will be discarded if you apply an import." The import doesn't auto-merge pending state — it replaces.

### 4.12 Confirmation modal when switching JSON → Form with pending changes

**Decision.** If the user clicks Form while in Advanced view with pending changes, a confirmation modal appears with options:

- **Keep editing** — stays in Advanced.
- **Apply defaults and switch** *(only shown when warnings exist and no errors)* — applies fallbacks, saves the config with edits, switches to Form.
- **Discard and switch** — reverts to saved state, switches to Form. Styled red to indicate destructive.

**Alternatives considered.** Silent switch (loses changes), or disable Form toggle until user saves/discards.

**Why modal won.** Explicit user agency. Three options cover the realistic paths:
- "I changed my mind, stay" (Keep editing)
- "I want to commit what I've got plus defaults, then move on" (Apply defaults and switch)
- "I'm just abandoning these edits" (Discard and switch)

The Apply-defaults path only appears when applicable, so the modal stays simple in the common case.

---

## 5. Component behavior

### 5.1 Toggle row

- Rendered as a 2-button pill (`Form` and `</> Advanced`).
- Centered above the agent header card, in its own row.
- Active state has white background, inactive is transparent.
- Form button has a list-rows icon to its left; Advanced has the `</>` braces icon.

### 5.2 Form view

- Layout matches existing AI Receptionist form (Identity, Skills, Scenarios, etc.).
- When pending JSON changes exist: a pending banner appears at the top of the form, and all form inputs are disabled. This is the bidirectional-sync guard — prevents the user from editing form fields while there are unreconciled JSON edits.
- Banner text explains: "You have N unsaved changes in the Advanced view. Save or discard them before editing here." (Buttons are not needed — the floating save bar at the bottom handles them.)

### 5.3 Advanced view

The Advanced view is a single card containing the JSON editor. Components, top to bottom:

**Header strip.** Left: `{ }` icon + "Configuration" title + read-only/lines count pill (e.g., `47 lines`). Right: Copy and Import buttons. Phase 2: editor mode indicator and additional actions.

**Validation status strip.** Single-line summary of validation state:
- Green: ✓ No issues
- Amber: ⚠ N warnings — see suggestions below
- Red: ✕ N errors — see suggestions below

**Repair panel** (only visible when issues exist). Below the status strip. Lists each issue as a row:
- Severity icon (red dot for error, amber for warning)
- Line number (clickable to jump to that line in the editor)
- Issue message (with inline `<code>` formatting for field names)
- Quick-fix buttons aligned right (primary blue for recommended fix)

When 2+ issues exist, the panel shows a summary row at the top: "2 errors and 1 warning · Click any row to jump to that line." The list scrolls if more than ~5 issues.

**Editor body.** Three columns:
1. Line gutter (left) — line numbers with diff markers (blue for modified, green for added, red for error lines).
2. Editor pane — transparent textarea over a syntax-highlighted `<pre>`. Scrolls horizontally and vertically; gutter scrolls in sync.

Line backgrounds tint per diff state (blue/green/red) with a 3px accent bar on the left of each tinted line.

### 5.4 Floating save bar

- Appears at bottom-center when pending changes exist (any view).
- Dark background (`#111322`), 12-14px corner radius, drop shadow.
- Left: "Unsaved changes in `[Section, Section, ...]`" — section names in amber, derived from semantic diff against last saved config.
- Right: `Discard` (white) and `Save changes` (brand blue) buttons.
- Discard reverts the editor to last saved without confirmation.
- Save changes opens the save review modal.
- When validation errors exist, Save changes is disabled (low opacity).

### 5.5 Save review modal

Centered modal. Contents:
- Header: "Save configuration changes" + subtitle "Review the changes before committing. Saving creates a new version that affects live calls."
- Validation strip (green): "Valid — final server-side checks will run on save."
- Optional fallback notice (blue strip): "↩ N defaults will be applied on save: `path1` → `value1` · `path2` → `value2`"
- Diff metadata: "N field changes vs current saved version (v12 · 2 days ago)"
- Diff list: red removed lines + green added lines, monospace, dotted-path field names.
- Footer: status text + Cancel / Save changes buttons.

On Save changes: applies fallbacks → updates baseline → increments version → updates version label → fires toast → closes modal.

### 5.6 Import modal

Modal. Two steps (paste, then diff).

**Paste step.**
- Header: "Import configuration" + subtitle.
- Optional pending-notice (yellow): "You have N unsaved edits that will be discarded if you apply an import." Only when pending JSON state exists.
- Syntax-highlighted paste area (textarea over highlight pre — same pattern as the main editor).
- Status line below textarea: "Empty — paste a configuration JSON to continue" / "X lines · Y chars · ready to validate" / red "Invalid JSON — [error]" / "Schema check failed · missing: [list]"
- Optional "Paste sample →" link to demo the flow.
- Footer: Cancel / Validate buttons.

**Diff step.**
- Validation strip (green): "Valid — review the changes before applying."
- Diff list (same shape as save modal).
- Footer: Back / Apply buttons.

On Apply: commits as new saved version, bumps version, fires toast, closes modal.

### 5.7 Discard confirmation modal

Smaller centered modal. Shown when user clicks Form toggle while Advanced has pending changes.

- Header: "Discard unsaved changes?" + dynamic message based on context.
- Footer buttons:
  - **Keep editing** (secondary) — closes modal, no action.
  - **Apply defaults and switch** (primary blue, conditional) — applies fallbacks, commits, switches to Form. Only shown when there are warnings and no errors.
  - **Discard and switch** (red, destructive) — reverts editor to saved, switches to Form.

---

## 6. Validation rules

Validation runs synchronously on every editor input. Pipeline:

1. **Parse JSON.** Caught failures yield syntax errors with the offending line.
2. **Schema check.** If parsed, run schema validation:
   - Required sections (identity, voice, behavior) — error if missing.
   - Field types — error if mismatched.
   - Enum values — error if outside allowed list.
   - Number ranges — error if out of bounds.
   - Soft-required fields — warning if empty/missing, with fallback to apply on save.
3. **Cross-field consistency (Phase 2).** Voice ID for provider, language for voice, etc.
4. **Plan / permission checks (server-side, on save).** Model availability, region restrictions, quotas.

See `config-breaking-use-cases.md` for the full catalog of validation cases.

### 6.1 Severity tiers

- **Error** — blocks save. Repair panel offers quick-fix buttons where applicable.
- **Warning** — allows save with fallback or notice. Soft-required emptiness is the canonical case.
- **Info** — passive guidance, doesn't surface in the panel.

### 6.2 Quick-fix actions

The repair panel can offer per-error quick-fixes. Currently implemented:

- `setSpeed(value)` — clamp voice.speed to a specified value.
- `setMaxTurn(value)` — clamp behavior.maxTurnSec.
- `insertSection(section)` — restore a missing required section from the last saved config.
- `addLanguage(value)` — append a language to identity.languages.
- `setGreeting(value)` — set a default greeting.
- `revertAll` — full revert to last saved.

Each quick-fix is a button in the repair panel with a label like "Set to 1.5" or "Restore from saved." Primary fixes are styled in brand blue.

---

## 7. State management

### 7.1 Pending state model

The editor maintains:
- `baselineConfig` — last saved JSON object.
- `baselineText` — `JSON.stringify(baselineConfig, null, 2)`.
- `editorInput.value` — current editor text (may differ from baselineText).
- `lastValidation` — `{ errors, warnings, parsed }` from the most recent validate() run.

Pending changes = any non-empty diff between `editorInput.value` and `baselineText`.

### 7.2 Save commits

On Save: `baselineConfig` is replaced by `applyFallbacks(lastValidation.parsed)`. `baselineText` is regenerated. The editor is re-rendered. Version increments.

### 7.3 Form/Advanced state sharing

Currently the prototype doesn't bind form inputs to the underlying JSON state. **In production**, the form and Advanced views must share a single source of truth — either:

- **JSON-is-canonical.** Form is a renderer over the parsed config. Form save = JSON Merge Patch on top of saved config. Advanced save = full replacement.
- **Section-by-section sync.** Form maintains per-section state; Advanced view reads/writes JSON. Switching between views requires reconciliation.

Recommendation: JSON-is-canonical with the form rendering only the fields it knows how to render. Form save sends a JSON Merge Patch (RFC 7396) of changed fields. Unrendered fields (e.g., `clientMessages`) are preserved.

---

## 8. Edge cases & state transitions

### 8.1 Mode switches with pending state

| From | To | Pending state | Action |
|------|----|----|--------|
| Form | Advanced | None | Switch directly. |
| Form | Advanced | Form edits | Should be impossible (form inputs disabled when JSON has pending) but if encountered: stay, banner explains. |
| Advanced | Form | None | Switch directly. |
| Advanced | Form | Pending JSON, no errors, no warnings | Switch with confirmation modal (Keep editing / Discard and switch). |
| Advanced | Form | Pending JSON + warnings only | Switch with confirmation modal (Keep editing / Apply defaults and switch / Discard and switch). |
| Advanced | Form | Pending JSON + errors | Switch with confirmation modal (Keep editing / Discard and switch). Apply-defaults is hidden. |

### 8.2 Save with errors

Save button is disabled when `lastValidation.errors.length > 0`. Save modal cannot be opened. The repair panel is the recovery surface.

### 8.3 Save with warnings only

Allowed. Save modal opens. The fallback notice strip indicates which defaults will be applied. User confirms.

### 8.4 Save with no pending changes

Save bar is hidden. Save button is unreachable via UI. Cmd+S is a no-op.

### 8.5 Import with pending Advanced state

Import modal opens with the yellow warning notice. User can proceed; on Apply, the pending Advanced edits are discarded.

### 8.6 Cmd/Ctrl+S keyboard shortcut

Only active when the editor textarea has focus. Calls `canSave()` (pending + no errors) → opens save modal if true.

### 8.7 Concurrent edit conflicts (Phase 2)

Currently not handled in the prototype. Production must implement optimistic locking: client sends `If-Match: v12` header on save; server returns 409 if HEAD is v13. Client surfaces conflict UI with the other user's changes.

---

## 9. Phase 1 vs Phase 2 scope

### Phase 1 (ship)

- Form view (existing)
- Advanced view: read-only JSON inspector with syntax highlighting
- Copy button (clipboard write)
- Import flow (paste → validate → diff → apply)
- Schema versioning (`voiceagent@1.4`) — required for any import flow
- Form expansion: add model selection, voice speed, transcriber controls, hangup detection to the form. *Critical*: without this, Group 2 has to use Advanced for every change.

### Phase 2 (defer)

- Editable JSON in Advanced view (with validation, repair panel, save flow as documented)
- Quick-fix actions in the repair panel
- Save modal with field-level diff
- Floating save bar
- Confirmation modal for JSON → Form with pending changes
- Soft-required fallback application on save
- Template variables (`${customer.x}`)
- "Save as template" (the cross-account porting wedge)
- Test-before-save loop

### Phase 3+ (further future)

- Schema-driven autocomplete (likely via Monaco)
- AI-mediated config editing (Composer pattern)
- Concurrent edit conflict handling
- Version history with one-click revert
- Compiled prompt preview tab

---

## 10. Open questions for engineering review

1. **JSON Merge Patch on Form save.** Confirm that the form save endpoint accepts partial patches and preserves unrendered fields. Required for the "form doesn't know about all fields" architecture.

2. **Schema delivery.** Where does the schema live? Bundled with the client? Fetched at runtime? Versioned via URL (`/schemas/voiceagent/1.4.json`)?

3. **Fallback resolution: client or server?** The prototype applies fallbacks client-side and shows them in the save modal. Production option: server applies on save, client just sends user input. Trade-off: transparency vs single-source-of-truth.

4. **Cost projection for model selection.** Implementing the "this config will cost $X/month" warning requires usage-aware lookups. Where does that live?

5. **Test call loop.** Phase 1 doesn't include test-before-save. When this ships, where does the test infrastructure live? Spin up an ephemeral agent? Use a sandboxed account?

6. **Concurrent edits.** When does optimistic locking become a requirement? Only matters if multiple users edit the same agent — for most accounts, this may never occur.

7. **Form-Advanced state sync precedence.** When form view becomes interactive with JSON state binding, define explicit precedence: user form edits > user JSON edits > fallback defaults > schema defaults.

---

## 11. Reference

- Prototype file: `voiceagent-tessa-edit.html`
- Use case catalog: `config-breaking-use-cases.md`
- User stories: `voiceagent-expert-user-stories.md`
- Earlier prototypes (for comparison):
  - `voiceagent-tessa.html` — original Form/Expert toggle with editable JSON
  - `voiceagent-tessa-import.html` — read-only side panel + import flow
  - `voiceagent-tessa-toggle.html` — read-only toggle pattern (no editing)

The current prototype (`voiceagent-tessa-edit.html`) represents the design we're shipping in Phase 2. Phase 1 ships a subset: everything except the editable JSON capability. The toggle, the inspector, the import flow, and the Form expansion all ship in Phase 1.
