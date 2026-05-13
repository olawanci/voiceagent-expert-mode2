# Config-breaking use cases

A catalog of the ways a user can invalidate or damage a VoiceAgent JSON config in the Advanced view, with the recommended system response for each. Use this as a test plan, a risk register, and a guide for what validation rules to ship.

Severity legend:

- **Error** — blocks save; the agent cannot run in this state
- **Warning** — allows save with a fallback or notice; the agent runs but suboptimally
- **Info** — informational only; the user should know but isn't blocked

---

## 1. Syntax errors (JSON doesn't parse)

These prevent the schema validator from even running. Recovery is almost always "revert" or surgical text fix.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 1.1 | Missing closing brace `}` | Error | Show parser error with line number. Only fix offered: Revert to saved. |
| 1.2 | Trailing comma after last property | Error | Same as above; in a richer editor, offer an auto-fix that removes the comma. |
| 1.3 | Missing comma between properties | Error | Same as above. |
| 1.4 | Unquoted property name or string value | Error | Same as above; suggest quoting in error message. |
| 1.5 | Smart quotes from rich-text paste (`"…"`) | Error | Strip and replace smart quotes with straight quotes on paste. Recoverable without revert. |
| 1.6 | Unterminated string | Error | Surface the line where parser got confused; revert is the safe path. |
| 1.7 | Comment (`//` or `/* */`) inserted | Error | JSON doesn't allow comments; strip on paste, or surface error. |
| 1.8 | Invalid escape sequence (`"\q"`) | Error | Surface error at line; revert. |
| 1.9 | Mixed line endings (CRLF/LF) | None visible | Normalize on paste; doesn't break parsing but can confuse diff. |

---

## 2. Schema violations (JSON parses, structure is wrong)

The parser is happy but the schema rejects.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 2.1 | Deleted required top-level section (`identity`, `voice`, `behavior`) | Error | Quick-fix: "Restore from saved" inserts the section from the last saved config. |
| 2.2 | Deleted required nested property (`identity.name`) | Error | Quick-fix: restore from saved, or "Set placeholder" with the agent's previous name. |
| 2.3 | Wrong type — string where number expected (`"speed": "fast"`) | Error | If coercible (`"1.2"` → `1.2`), offer "Parse as number"; otherwise revert. |
| 2.4 | Wrong type — object where array expected | Error | Revert; no safe surgical fix. |
| 2.5 | Invalid enum value (`tone: "vibrant"`) | Error | "Did you mean: warm, empathetic, professional?" with click-to-fix per option. |
| 2.6 | Out-of-range number (`speed: 1.8` when max is 1.5) | Error | Clamp to bound with one-click fix ("Set to 1.5") + suggest a more conservative alternative ("Set to 1.2 — recommended"). |
| 2.7 | Out-of-range with unit confusion (`maxTurnSec: 1800`) | Error | Clamp with explanation: "maxTurnSec is in seconds, max is 120. Did you mean milliseconds?" |
| 2.8 | Empty string in required field (`name: ""`) | Error | "Name cannot be empty" with revert option. |
| 2.9 | Null in non-nullable field (`voice: null`) | Error | Revert from saved. |
| 2.10 | Unknown property at known path (`identity.warmth: "high"`) | Warning | Field is not in schema. Strip on save, or preserve as expert-only with a notice. |
| 2.11 | Duplicate property keys in same object | Error | JSON spec is ambiguous; most parsers keep the last value. Warn loudly. |

---

## 3. Empty soft-required fields (fallback applies)

The field is required for the agent to function, but the system has a safe default.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 3.1 | Empty `identity.languages` array | Warning | Default `en-US` applied on save. Quick-fix: "Add en-US" / "Add en-GB". |
| 3.2 | `languages` key removed entirely | Warning | Same as above. |
| 3.3 | Empty `behavior.greeting` string | Warning | Default greeting applied on save. Quick-fix: "Set default greeting". |
| 3.4 | Missing `behavior.model` | Warning | Default model (account tier maximum) applied on save. |
| 3.5 | Missing `voice.provider` | Warning | Default provider (account-configured) applied on save. |
| 3.6 | Missing `behavior.maxTurnSec` | Warning | Default 30s applied on save. |
| 3.7 | Missing `voice.speed` | Warning | Default 1.0 applied on save. |

---

## 4. Cross-field consistency violations

The values are individually valid but their combination isn't.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 4.1 | `voiceId` doesn't exist for selected `provider` (`elevenlabs` + `alloy` which is OpenAI-only) | Error | List valid voices for the current provider; surface in error message. |
| 4.2 | Language without a matching voice (`languages: ["pl-PL"]` but no Polish voice configured) | Warning | "No Polish voice — agent will fall back to default. Add a voice for `pl-PL`?" |
| 4.3 | Model not available on plan tier (`opus` on Free) | Error on save | "Pro plan required for `claude-opus-4-6`. Use `claude-sonnet-4-6` or upgrade." |
| 4.4 | Skill references nonexistent skill name | Error | List available skills; the reference is dangling. |
| 4.5 | Scenario references a skill that's been deleted | Error | Same as above. |
| 4.6 | Template variable references undefined account-level variable (`${customer.unknownField}`) | Warning | Warn with a link to defined variables; the agent will see the literal string at runtime. |
| 4.7 | Two skills with the same `name` | Error | Names are identifiers; collision creates ambiguity. |
| 4.8 | Transcriber language doesn't match identity languages | Warning | Surface mismatch; agent may fail to transcribe correctly. |

---

## 5. Schema version mismatch

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 5.1 | Older minor version pasted (`voiceagent@1.3`) | Info | Auto-migrate to current; notice in toast: "Migrated from v1.3." |
| 5.2 | Older major version (`voiceagent@0.9`) | Error | Reject with link to migration guide. |
| 5.3 | Future version (`voiceagent@2.0`) | Error | "This config is from a newer schema than this account supports." |
| 5.4 | Missing `$schema` field | Error | "Schema version required. Add `\"$schema\": \"voiceagent@1.4\"`?" Quick-fix to add it. |
| 5.5 | Malformed `$schema` value (`voiceagent1.4`) | Error | Suggest correct format with quick-fix. |

---

## 6. Reference / dependency errors

The config references things that exist elsewhere — these references can break.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 6.1 | Knowledge source ID points to deleted document (`knowledge.sources[0].id: "doc_xyz"` no longer exists) | Warning | "Source `doc_xyz` was deleted. The agent will skip it." Offer to remove. |
| 6.2 | Template variable references unknown namespace (`${unknown.foo}`) | Warning | Warn; agent will see literal string. |
| 6.3 | Transfer destination is a phone number that has been disconnected | Warning | Catch at save time via account validation. |
| 6.4 | Webhook URL points to an internal/private network address | Error | Block: SSRF risk. |
| 6.5 | Webhook URL uses `http://` instead of `https://` | Warning | "Plaintext webhook — consider HTTPS." Or block on stricter policy. |

---

## 7. Business / semantic warnings (config is valid but suboptimal)

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 7.1 | Aggressive pacing (`speed: 1.4`) | Warning | "May sound rushed for some accents. Consider 1.2." |
| 7.2 | Very slow pacing (`speed: 0.6`) | Warning | "May feel sluggish. Consider 0.9." |
| 7.3 | Expensive model on high-volume agent (Opus on >1000 calls/day) | Warning | Cost projection: "At current call volume, this model will cost ~$X/month. Consider Sonnet or Haiku." |
| 7.4 | Very long `maxTurnSec` (>90s) | Warning | "Long turns may feel unnatural. Consider 30–45s." |
| 7.5 | Tone-greeting mismatch (`tone: "professional"` + casual `greeting`) | Info | Hard to detect mechanically; lint rule could surface. |
| 7.6 | Empty `skills` and `scenarios` (agent only greets) | Warning | "Agent has no skills or scenarios. Callers will only hear the greeting." |
| 7.7 | `interruptible: false` with long greeting | Warning | "Callers can't interrupt and the greeting is long. They may hang up." |
| 7.8 | No fallback scenario for unknown intents | Warning | "Define a fallback scenario for unmatched callers." |

---

## 8. Sensitive data / security flags

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 8.1 | Hardcoded phone number where `${customer.x}` expected (`"to": "+1-415-555-0100"`) | Warning | "This won't port across accounts. Extract as `${customer.transferNumber}`?" |
| 8.2 | API key visible in webhook URL or header (`?apikey=sk-xyz`) | Error | Block save; strip the credential from the diff before display. |
| 8.3 | PII in greeting (`"Hi John, this is..."` — specific customer name) | Warning | "Greeting includes a specific name. This won't be reusable across customers." |
| 8.4 | Email address in scenario reply (literal `support@company.com`) | Info | "Consider `${company.supportEmail}` for portability." |
| 8.5 | Personally-identifying transcriber settings that imply caller demographics | Info | Privacy review prompt. |

---

## 9. Accidental destruction

Whole-config-level catastrophic edits.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 9.1 | Select-all + delete (editor is empty) | Error | Empty isn't valid JSON. Single fix: Revert to saved. |
| 9.2 | Pasted wrong content (an email, an image, paragraph of text) | Error | JSON parse fails → revert. |
| 9.3 | Pasted partial JSON (first half of a long config) | Error | Syntax error from unclosed braces; revert. |
| 9.4 | Paste contains BOM or zero-width characters | Error | Strip invisible chars on paste before validating. |
| 9.5 | User typed over a key by mistake (`"identity"` → `"identty"`) | Error | Unknown property at top level; suggest "Did you mean `identity`?" |

---

## 10. Plan / permission errors

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 10.1 | Model requires paid tier (`opus` on Free) | Error on save | Block save; offer upgrade path or fallback model. |
| 10.2 | Provider requires add-on (`elevenlabs` not enabled) | Error on save | Block; suggest available providers. |
| 10.3 | Region restriction (voice not available in user's region) | Error on save | Block; suggest equivalent voice in user's region. |
| 10.4 | Feature flag off (advanced transcriber options) | Error on save | Block; offer to remove the advanced fields. |
| 10.5 | Quota exceeded (config size > 50KB, too many skills, etc.) | Error on save | Block; suggest trimming. |

---

## 11. Concurrent edit conflicts

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 11.1 | Server has newer version (someone else saved meanwhile) | Error on save | Conflict UI: show both versions, let user merge or override. |
| 11.2 | Editor's cached state is stale (browser tab left open for 24h) | Error on save | Detect via version mismatch; offer to reload before editing. |
| 11.3 | User has multiple tabs editing the same agent | Error on save | Same as above; broadcast tabs (BroadcastChannel) can warn earlier. |

---

## 12. Logical / behavioral contradictions

Hard to detect mechanically but worth lint rules for.

| # | Use case | Severity | Recommended response |
|---|----------|----------|----------------------|
| 12.1 | `maxTurnSec: 5` with verbose greeting that takes 10s | Info / lint | "Greeting may be cut off; the max turn is 5s." |
| 12.2 | Greeting asks "Can you tell me your name?" but no `take_message` or similar skill | Info / lint | "Greeting prompts the caller, but there's no skill to handle the response." |
| 12.3 | `tone: "professional"` but greeting starts with "Hey!" | Info / lint | Tone-content mismatch; subjective. |
| 12.4 | Scenarios contradict each other (two scenarios with the same trigger) | Warning | "Multiple scenarios match the same trigger. Only the first will fire." |
| 12.5 | Skill defined but never referenced by any scenario | Info | "Skill `transfer_to_billing` is defined but never used." |

---

## Sequencing the response

When multiple issues exist simultaneously (which is common — a user pasting a broken config might hit 5+ at once), the repair panel should prioritize:

1. **Syntax errors first** — nothing else matters until the JSON parses.
2. **Required-section errors next** — `identity` missing blocks all other field-level checks.
3. **Field-level errors** — type, enum, range. These can be fixed in parallel.
4. **Cross-field errors** — only meaningful after individual fields are valid.
5. **Soft warnings** — surfaced but not blocking.
6. **Lint / business warnings** — passive guidance.

Save is gated on errors only. Warnings allow save with fallback application or explicit accept.

## Recovery paths the user always has

Independent of which use case, the user can always:

- **Revert to saved** — atomic undo back to the last committed version.
- **Discard and switch to Form** — same as revert, then go to Form view.
- **Import a different config** — paste a known-good config to start over.
- **Apply defaults and continue** — for fallback-applicable warnings, accept the system defaults.
- **(Future)** Ask Claude to repair — for combinations the user can't diagnose alone.

These exist for every use case in this catalog, with varying suitability — but the user is never stuck.
