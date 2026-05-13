# VoiceAgent Expert Mode — User Stories

## Context

CloudTalk's VoiceAgent setup is currently a click-by-click form wizard. It works for non-technical PLG users but underserves three other personas — TC consultants, partner integrators, and sales-assisted technical customers — who need faster iteration, more exposed parameters, and portability across accounts. Several knobs that were exposed in the previous UI (model selection, voice latency tradeoffs, pacing) were removed and now require an internal request to change. Customers are losing deals to Retell and VAPI partly because those platforms expose more control. Today's workaround for technical users is to copy-paste prompts to and from GPT, which is slow and doesn't cover the hidden parameters.

The strategic decision after discovery: ship a **read-only JSON inspector with an import flow** as Phase 1 (Option 1), expand the form with the missing parameters in parallel, and defer **editable JSON / AI-mediated config editing** to Phase 2 pending validation. The design principle stays *UI primary, expert mode secondary*: easy for non-technical, doable for technical, hackable for TC, impossible for everyone else.

## Personas

- **Non-technical user (PLG)** — the primary persona. Sets up an agent through the wizard, never opens the inspector, never sees JSON. Should not be confused or distracted by the new surfaces.
- **Technical customer (sales-assisted)** — knows what they want and expects parity with Retell/VAPI on exposed knobs. Should be able to self-serve without filing internal requests.
- **TC consultant / partner** — sets up agents on behalf of multiple customer accounts. Workflow is copy-paste-tweak across accounts. Wants config portability and confidence that changes can be reviewed before committing.

## Phase plan

- **Phase 1 (this doc)** — read-only JSON inspector, import flow with diff preview, form expansion to surface missing knobs, schema versioning, plan-tier gating on expensive parameters. Targets the future PR statements directly.
- **Phase 2 (deferred, listed at the end)** — editable JSON, AI Composer, Save-as-template, version history with one-click revert. Sequenced after validation.

---

## Epic 1: Read-only configuration transparency

### Story 1.1 — View the agent's full configuration as JSON

> *As a TC consultant, I want to view an agent's full configuration as a read-only JSON document, so that I can understand exactly what's configured, debug behavior issues without filing a support ticket, and share the config with colleagues or customers.*

**Acceptance criteria**

- An "Assistant Configuration" button appears in the agent header card (secondary button style, with a `</>` icon).
- Clicking it slides a 520px side panel in from the right; the page content shifts left rather than being overlaid.
- Panel header shows the title "Assistant Configuration" and a close (X) button.
- Panel sub-header shows "JSON Format" with the line count and a `read-only` indicator.
- JSON is rendered with syntax highlighting, line numbers, and the schema version pill (`voiceagent@1.4`).
- The JSON view has no cursor, no inline edit affordances, no edit/save buttons.
- Toggling the same button or pressing the close button slides the panel out; the button reflects open state visually.

### Story 1.2 — Copy configuration with one click

> *As a TC consultant, I want to copy the entire configuration to my clipboard with one click, so that I can paste it into Claude or GPT for editing, save it as a Notion snippet, or attach it to a support ticket without manual selection.*

**Acceptance criteria**

- A "Copy" button appears in the panel toolbar.
- Click copies the full JSON exactly as displayed (preserving formatting and indentation).
- A short toast or inline confirmation acknowledges the copy.

### Story 1.3 — Persona protection for non-technical users

> *As a non-technical PLG user, I want the configuration view to feel optional and low-prominence, so that I'm not confused or distracted by JSON when I'm just trying to set up a simple agent.*

**Acceptance criteria**

- The "Assistant Configuration" button is rendered as a secondary action, never as the primary CTA on the page.
- The side panel never auto-opens for users who haven't opened it before.
- Closing the panel returns the user to the standard form view with no lingering UI.
- The form view is a complete editing surface for everything a non-technical user needs (no required fields are JSON-only).

---

## Epic 2: Cross-account configuration porting

### Story 2.1 — Open the import flow

> *As a TC consultant, I want a clear entry point to import a configuration from another account, so that I don't have to manually re-click through every section to apply a known-good setup.*

**Acceptance criteria**

- An "Import config" button appears in the panel toolbar next to Copy.
- Click opens a centered modal titled "Import configuration" with subtitle "Paste a JSON config from another account or from Claude. Changes will be reviewed before applying."
- Modal has a paste textarea, a footer with Cancel / Validate buttons, and a sample-paste affordance for first-time users.
- Cancel or close (X) dismisses the modal without changing any state.

### Story 2.2 — Validate pasted JSON before applying

> *As a TC consultant, I want pasted configurations validated before I apply them, so that I don't accidentally break a working agent with a typo, a malformed JSON, or a config from an incompatible schema version.*

**Acceptance criteria**

- The Validate button is disabled when the paste area is empty.
- On click, the system parses the JSON and runs schema validation:
  - Invalid JSON → footer shows a red error with the parse position; modal stays in paste step.
  - Schema mismatch (missing required fields, unknown enum values, type errors) → footer shows a specific error explaining the failure; modal stays in paste step.
  - Schema version mismatch (e.g., pasting a v1.3 config when current schema is v1.4) → footer shows a version error with a link to the migration doc.
  - Valid → modal advances to the diff step with a green confirmation strip.
- Validation runs entirely client-side for immediate feedback; deeper backend validation (model availability, voice availability for plan tier) runs on Apply.

### Story 2.3 — Review changes before applying

> *As a TC consultant, I want to see exactly what will change before I apply an imported config, so that I can confirm only the intended changes are applied and avoid surprises.*

**Acceptance criteria**

- Diff step shows all field-level changes against the current saved version.
- Each change rendered as added (green +) or removed (red −), with the field path in dotted notation (e.g., `voice.speed`, `behavior.tone`).
- Diff count and version reference shown at the top (e.g., "5 changes vs current saved version (v12 · 2 days ago)").
- A Back button returns to the paste step without losing the pasted content.
- Apply button is the primary action; Cancel is hidden once the diff is shown (Back replaces it).

### Story 2.4 — Apply imported configuration

> *As a TC consultant, I want to apply a validated, reviewed configuration with one click, so that I can move on to the next account quickly and meet the goal of closing setup tickets 3x faster.*

**Acceptance criteria**

- Apply button commits the new configuration as the next version (e.g., v12 → v13).
- Form fields update to reflect the new values.
- Side panel JSON re-renders to reflect the new state.
- Modal closes; a success toast confirms the change ("Configuration applied · v13").
- Updated form fields show a temporary "Updated" pill that fades on next page load.
- Backend deeper-validation errors (e.g., "this model isn't on your plan") block the Apply with an inline error, leaving the user in the diff step.

### Story 2.5 — Use template variables for portability

> *As a TC consultant, I want to author configurations with template variables like `${customer.transferNumber}` or `${company.name}`, so that one config works across multiple customer accounts without find-and-replace per account.*

**Acceptance criteria**

- Template variables (`${path.to.value}`) are visually highlighted in both the JSON inspector and the diff view (distinct color, e.g., amber).
- Validation accepts template variables in any string field.
- Template variables resolve at agent runtime, not at save time, against a per-account variable store.
- Documentation lists available variable namespaces (`customer.*`, `company.*`, `booking.*`).

---

## Epic 3: Form expansion to expose hidden knobs

This epic addresses the "we removed configurability customers had before" problem. It runs in parallel with Epic 1–2 and is required to deliver the technical-customer PR statement (matching Retell on control).

### Story 3.1 — Select LLM model for the agent

> *As a technical customer, I want to choose which LLM the agent uses, so that I can make informed cost/quality tradeoffs and stop filing internal requests for model changes.*

**Acceptance criteria**

- A model selector is exposed in the form (Behavior or Advanced section).
- Choices are gated by plan tier: Free → Haiku only; Pro → Haiku + Sonnet; Enterprise → Haiku + Sonnet + Opus.
- Each choice shows estimated per-minute inference cost and a one-line tradeoff description (e.g., "Faster, lower cost, may produce shorter answers").
- Changing model shows an inline notice about cost impact before save.
- The selection is reflected in the JSON inspector immediately on save.

### Story 3.2 — Adjust voice pacing and interruption behavior

> *As a technical customer, I want to control voice speed, interruption sensitivity, and endpointing latency, so that I can tune the agent for my callers (e.g., slower for elderly demographics, more responsive for fast-moving sales calls).*

**Acceptance criteria**

- Voice section exposes: speed slider (0.7–1.4, default 1.0), interruptible toggle, endpointing-ms slider with sensible bounds.
- Each control has a tooltip with a tradeoff explanation.
- Out-of-range values are blocked at the form level with inline messages.
- Aggressive values (e.g., speed > 1.3) trigger a soft warning in the JSON inspector when viewed.

### Story 3.3 — Configure first-turn and end-of-call behavior

> *As a technical customer, I want to control whether the agent speaks first, the exact greeting, the voicemail message, and the end-call message, so that I can match my brand voice and call flow expectations.*

**Acceptance criteria**

- First-message-mode dropdown (Assistant speaks first / Caller speaks first / Wait for caller).
- First-message text field (rendered with template-variable highlighting).
- Voicemail-message text field.
- End-call-message text field.

### Story 3.4 — Configure transcriber

> *As a technical customer, I want to select transcriber model, language, and confidence threshold, so that I can match the agent to caller demographics and accuracy needs.*

**Acceptance criteria**

- Transcriber section in form (Advanced or its own section).
- Provider, model, language selectors with sensible defaults per region.
- Confidence threshold slider with default 0.4.
- Fallback plan toggle.

---

## Epic 4: Schema as a public contract

### Story 4.1 — Versioned, public schema

> *As a backend engineer, I want a versioned and public schema for VoiceAgent configurations, so that customers' saved configs continue to work as we evolve the product, and so the schema becomes a stable contract instead of an implicit one.*

**Acceptance criteria**

- Schema version is included in every saved config (e.g., `"$schema": "voiceagent@1.4"`).
- Existing v1.x configs never break silently after a backend deploy.
- Adding required fields requires a version bump (1.4 → 2.0).
- Renaming or removing fields requires a migration plan documented before merge.
- A public changelog is published; the inspector panel links to it from the schema docs link.

### Story 4.2 — Migrate older configs on save

> *As a customer with an older saved configuration, I want my agent to keep working when CloudTalk evolves the schema, so that I don't have to re-edit anything when minor schema updates ship.*

**Acceptance criteria**

- On save, configs are migrated from any v1.x to the latest minor version transparently.
- Migration is logged in the version history with a "schema migration" note.
- The customer is not prompted unless a major-version migration is required.

### Story 4.3 — Reject incompatible imports clearly

> *As a TC consultant, I want clear, actionable errors when I try to import a config from a different schema version, so that I understand what went wrong and what to do next.*

**Acceptance criteria**

- Imports validated against the current schema version.
- If incompatible: error message explains the version mismatch and links to the migration doc; modal stays in paste step.
- If on a compatible older minor version: import proceeds with an automatic migration and a notice ("Migrated from v1.3 to v1.4 — 2 fields renamed").

---

## Epic 5: Safety and trust

### Story 5.1 — See current saved version

> *As a TC consultant, I want to know what version the agent is on and when it was last saved, so that I can avoid testing changes that haven't actually been committed and so that I can speak precisely with customers about what they're using.*

**Acceptance criteria**

- Agent header shows current version label (e.g., "v12") and last-saved timestamp.
- After import: version label updates to the new version.

### Story 5.2 — Block aggressive configuration values

> *As a CloudTalk product owner, I want safety guards on parameters like voice speed, response length, and interruption sensitivity, so that customers don't accidentally configure agents that sound rushed, robotic, or rude and damage CloudTalk's brand.*

**Acceptance criteria**

- Schema enforces min/max for speed, latency, and turn duration.
- Form sliders enforce same bounds visually.
- Out-of-range values rejected at the validate step in import.
- Edge values (within bounds but suboptimal) trigger soft warnings in the inspector.

### Story 5.3 — Strip sensitive data from shared configs

> *As a TC consultant sharing a config across customer accounts, I want template variables for customer-specific fields, so that I don't accidentally leak one customer's data (transfer numbers, customer names, auth tokens) into another's account.*

**Acceptance criteria**

- Schema flags fields likely to contain account-specific data.
- (Phase 2 follow-up) "Save as template" automatically converts these fields to `${}` variables.
- Import warns if literal values appear in fields that should be variables (e.g., a phone number where a `${customer.transferNumber}` is expected).

### Story 5.4 — Audit trail for configuration changes

> *As a CloudTalk admin or support engineer, I want every config change logged with full context, so that I can support customers when something breaks and trace what happened.*

**Acceptance criteria**

- Every save creates an immutable audit entry: who, when, summary of changes, full pre/post config.
- Admins can view the audit log for any agent.
- Audit log includes whether the change came from the form, the import flow, or (Phase 2) a Composer suggestion.

---

## Epic 6: Plan tier and pricing (Viability)

### Story 6.1 — Gate model selection by plan

> *As a CloudTalk product owner, I want to gate expensive model selection by paid tier, so that we don't blow COGS on free-tier customers picking Opus on high-volume workflows.*

**Acceptance criteria**

- Free tier: Haiku only (auto-selected, not exposed as a choice).
- Pro tier: Haiku + Sonnet exposed.
- Enterprise tier: full model set exposed.
- UI clearly indicates which models are gated and why; offers an upgrade path.
- API enforces gates regardless of UI (so an imported config picking a gated model on an ungated plan is rejected at validation).

### Story 6.2 — Transparent inference cost

> *As a Pro or Enterprise customer, I want to see how my model and voice choices affect per-minute cost, so that I can make informed decisions and don't get surprised on the next invoice.*

**Acceptance criteria**

- Per-minute cost estimate visible at the model selector and voice provider selector.
- Estimated cost summary shown on the agent header (e.g., "~$0.14/min").
- Monthly invoice itemizes inference cost separately from base subscription.

---

## Epic 7: Discovery and validation tasks

These are not user stories per se, but commitments tied to the Phase 1 launch.

- Run desirability research with 5 TCs before locking schema. Show real configs, ask them to read, modify, and apply scenarios. Kill condition: if 3 of 5 can't get to a working config in under 15 minutes, pause Phase 2 and revisit.
- Run a feasibility spike on codependent schema validation (voice availability depending on language and provider; speed range depending on voice). Kill condition: if Monaco-style live-constrained editing is infeasible for Phase 2, descope or replace with form-only.
- Decide pricing structure on model selection before launch (tier-gated vs. pass-through priced). Block on this — it's the most likely commercial blocker.
- Validate cross-account paste workflow with at least 3 active TCs. Confirm whether their actual workflow is "paste once with placeholders" or "paste 10 times with hand-edits." If the latter, accelerate template variables (Story 2.5) to launch parity.

---

## Phase 2 — deferred user stories

Listed for transparency. Not committed to scope until Phase 1 validation completes.

### Story 8.1 — Edit configuration as JSON inline

> *As a TC consultant, I want to edit the JSON configuration directly in the side panel with live schema validation, so that I can make surgical changes without leaving the product to use Claude or a text editor.*

(Depends on: Phase 1 schema stability, COGS decision, desirability validation. Adds: live syntax validation, autocomplete on enums, inline diff before save, test-before-save loop.)

### Story 8.2 — Save configuration as a reusable template

> *As a TC consultant, I want to save a working configuration as a named template, so that I can apply it to new customer accounts in seconds instead of pasting JSON each time.*

(The wedge that makes "10 accounts in minutes" literally true rather than aspirational. Should auto-convert account-specific values to template variables on save.)

### Story 8.3 — Edit configuration with Claude (Composer)

> *As a TC consultant, I want to describe configuration changes in natural language and have the system apply them as a reviewable patch, so that I can iterate even faster than copy-paste.*

(Depends on: stable schema, prompt engineering investment, latency tolerance. Closes the Claude-loop entirely inside the product.)

### Story 8.4 — Test configuration before saving

> *As a TC consultant, I want to make a real test call against a pending configuration without saving it, so that I can validate the change before customers are exposed to it.*

(May ship in Phase 1 if Apply is decoupled from Save. As specified in Phase 1, Apply commits immediately. Decision pending.)

### Story 8.5 — Roll back to a previous version

> *As a TC consultant, I want to revert to a previous configuration version, so that I can recover from a bad change without re-typing everything.*

(Requires the audit/version history from Story 5.4 to land first.)

---

## Out of scope (this round)

- **Per-section JSON/Form toggle (Retell pattern)** — explicitly not chosen. Adds bidirectional sync complexity for marginal benefit over the global inspector.
- **Embedded Monaco editor in the panel** — only relevant if Phase 2 adds editability.
- **Custom skill authoring via JSON** — the Skills section stays form-driven; the JSON view shows skill state but skills aren't authored as raw JSON in Phase 1.
- **Multi-language schema docs** — single English doc for launch.

---

## Open decisions

- Does **Apply** in the import flow commit immediately, or does it stage the change and require an explicit Save? (Affects Story 2.4 and Story 8.4.) Recommendation: stage-then-save once test-before-save is built; commit immediately for v1.
- Does the inspector show the **compiled prompt** as a second tab in Phase 1, or only after editable Expert mode ships? Recommendation: Phase 1 ships read-only inspector + import only; compiled prompt view ships with Phase 2 editable mode (where it earns its weight). If TC research shows debugging is the dominant inspector use case, accelerate.
- **Pricing model for inference** — tier-gated vs. pass-through. Blocks launch.
