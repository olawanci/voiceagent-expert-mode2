# VoiceAgent Advanced Mode — Acceptance Criteria

Concise ACs grouped by epic. "MUST" indicates a release blocker. Prototype `voiceagent-tessa-edit.html` is the visual reference.

---

## Toggle (Form / Advanced)

- Centered pill toggle above all content with two buttons: `Form` (list icon) and `</> Advanced`.
- Form is the default on first load. Switching preserves in-progress state in both views.
- Toggle MUST be discoverable but never auto-switched by role.

## Form view

- All existing AI Receptionist form sections render unchanged.
- When pending JSON edits exist: banner explains the state, form inputs are disabled, Discard / Review & save actions available.

## Advanced view (editable JSON)

- Renders the current config as a syntax-highlighted JSON editor with a line-numbered gutter.
- Header includes `{ }` icon, "Configuration" title, line count, and schema version pill.
- Template variables (`${path.to.var}`) visually distinct from literal strings.
- Editor accepts keyboard input. Tab inserts two spaces. Cursor position preserved across re-renders.
- Cmd/Ctrl+S opens the save modal when there are pending saveable changes.
- `Copy`, `Import`, `Revert`, and `Save` buttons in the editor card header. Revert appears when pending changes exist; Save shows a pending count badge.

## Validation

- Every edit (debounced ≤ 200ms) runs JSON parse + schema validation.
- Status strip: green ✓ when valid, amber when warnings only, red when errors. Updates within 300ms of last keystroke.
- Errors block save; warnings allow save with fallback application.
- Covers: syntax failures, missing required sections, type mismatches, out-of-range values, invalid enums, soft-required emptiness.
- Validation MUST NOT cause typing lag or modify user input.

## Repair panel

- Appears below status strip when 1+ issues exist. Lists each issue: severity icon, line number, message, quick-fix buttons.
- Scrolls internally beyond 5 rows; editor does not shrink.
- Click row (not button) → editor scrolls to line and cursor focuses there.
- Quick-fix buttons mutate the JSON in place and trigger re-validation. Only `Revert to saved` is offered for syntax errors.
- Available fixes: clamp speed/maxTurn to bounds, restore missing section, add default language, set default greeting, revert all.

## Diff highlighting

- Lines that differ from last saved show: gutter marker + line background tint + 3px accent bar.
- Modified = blue, added = green, error = red. Errors take precedence over diff colors.
- Updates within 200ms of input; scrolls in sync with editor.

## Copy

- `Copy` copies the full JSON to the clipboard via the Clipboard API.
- Toast confirms: "Configuration copied to clipboard."
- Gracefully no-ops if the Clipboard API is unavailable.

## Import flow

- `Import` opens a modal with paste → validate → diff → apply.
- Validate produces specific errors for malformed JSON, schema mismatches, and missing required sections. The flow does not advance until valid.
- Diff shows field-level changes (`path: old → new`); template variables stay highlighted.
- Apply commits as the next saved version; version increments; toast confirms.
- If pending Advanced edits exist on open: yellow notice warns they'll be discarded.
- Imports MUST NOT modify the config until Apply is clicked.

## Save flow

- Save button disabled when no pending changes or when validation errors exist.
- Save opens modal: validation strip (green), optional fallback notice (blue), diff metadata, field-level diff list.
- Confirm → applies fallbacks → commits new version → updates editor → fires toast.
- Revert restores last saved baseline without confirmation. Cancel in modal closes without modifying.
- Save MUST NOT auto-fire or bypass the diff modal.

## Fallback application

- Soft-required fields (empty `identity.languages`, empty `behavior.greeting`, missing `behavior.model`, etc.) auto-apply documented defaults on save.
- Save modal shows a fallback notice listing each applied default before commit.
- Toast indicates when fallbacks were applied.
- Fallbacks MUST be visible in the modal — never silently injected.
- Hard-required fields (e.g., `identity.name`) have no fallback; missing them is an error.

## State transitions

- Form → Advanced: switches immediately.
- Advanced → Form with no pending changes: switches immediately.
- Advanced → Form with pending changes: confirmation modal with three actions:
  - `Keep editing` — closes modal, stays in Advanced
  - `Discard and switch` (red) — reverts edits, switches to Form
  - `Apply defaults and switch` (primary) — only shown when warnings exist and no errors; applies fallbacks, commits, switches

## First-time notice

- On first entry to Advanced view (per user per browser): banner appears above editor card.
- Title: "Welcome to Advanced mode." Body explains capabilities (edit, import) and the fallback safety net.
- Dismissible via X. Dismissal persists via `localStorage`; degrades to session-only if localStorage unavailable.
- Banner MUST NOT block interaction.

## Schema as contract

- Every saved config includes a `$schema` field with the current version.
- Imports with same minor version: accepted as-is.
- Imports with older minor version: auto-migrate on Apply with notice in toast.
- Imports with older major or future versions: rejected with link to docs.
- Within-major schema changes MUST be additive only.

---

## Cross-cutting

### Responsiveness

- Works from 360px to wide desktop. Sidebar hides below 768px. Modals become bottom sheets below 768px.
- All interactive elements have touch targets ≥ 32px.
- No horizontal page scroll at any supported width.

### Accessibility

- Full keyboard navigation with visible focus states on all interactive elements.
- View toggle uses `role="tablist"`; tabs use `aria-selected`.
- Modals trap focus, return focus to trigger on close, dismissible via Esc.
- Decorative SVGs are `aria-hidden`; icon-only buttons have `aria-label`.
- Validation messages reach screen readers via live regions.
- Color is never the sole indicator of state.

### Performance

- Advanced view first paint < 100ms after toggle activation.
- Validation, highlighting, and diff render < 300ms at configs ≤ 200 lines.
- No typing lag at configs ≤ 500 lines.
- Total bundle ≤ 200KB gzipped.

### Browser support

- Latest Chrome, Safari, Firefox, Edge desktop.
- iOS Safari 15+, Android Chrome on 11+.
- Graceful degradation in older browsers (no crash; basic Form still works).

### Audit & security

- Every save logged server-side with user, timestamp, version pre/post, diff summary.
- Server-side schema validation runs on every save and import (client validation is UX aid, not security boundary).
- Webhook URLs in tool configs restricted to allowlisted hosts; private/internal IPs rejected.
- Pasted content sanitized for invisible and control characters before validation.
- Toast messages MUST NOT contain user PII or config values.

---

## Regression scenarios

End-to-end journeys that MUST pass before release.

1. First user opens an agent → toggles to Advanced → sees first-time notice → dismisses → notice doesn't return.
2. User clicks Copy → paste elsewhere matches displayed JSON.
3. User imports a valid config from another agent → diff reviewed → applied → version increments.
4. User imports malformed JSON → specific error shown → flow does not advance.
5. User imports older minor version → auto-migrates on Apply with notice.
6. User edits in Advanced → Save → reviews diff → confirms → version increments.
7. User breaks speed range → red status + repair panel → clicks quick-fix → green → save.
8. User deletes `identity` AND breaks speed → both errors listed → resolves each → saves.
9. User empties `identity.languages` → warning → save modal shows fallback notice → save applies `en-US`.
10. User edits in Advanced → clicks Form → modal with three options → Apply defaults and switch → lands in Form with config saved.
11. User clicks Revert → editor returns to last saved baseline.
12. User makes valid edit → Cmd/Ctrl+S → save modal opens.

---

## Out of scope

- Monaco editor integration
- AI-mediated config repair (Composer)
- Schema-driven autocomplete on enum values
- Hover docs on schema properties
- Multi-cursor edits, code folding, find/replace
- Version history view with one-click revert
- Concurrent edit conflict resolution UI
- Compiled prompt preview tab
- Per-section JSON/Form toggle
- Save as template

---

Cross-references: `voiceagent-advanced-handoff.md` (design rationale), `voiceagent-expert-user-stories.md` (user stories), `config-breaking-use-cases.md` (validation catalog), `voiceagent-tessa-edit.html` (prototype).
