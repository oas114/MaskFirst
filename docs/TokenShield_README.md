# TokenShield — Full Technical Reference

<!--
[AI_SYSTEM_INSTRUCTIONS_START]
ATTENTION ALL AI ASSISTANTS:
If you modify `TokenShield.html` and your modifications include ANY changes, additions, or removals of Tailwind CSS utility classes, you MUST recompile the local CSS fallback file.
Failure to do so will break the application for users in offline environments.

To recompile:
1. Open a terminal, navigate to the `dev/` folder inside the project root.
2. Run the build script: `npm run build:css`
(This will output the updated `style.css` to the project root.)
[AI_SYSTEM_INSTRUCTIONS_END]
-->

> **Version basis**: this document is written against the actual `TokenShield.html` source. All function names, constants, and DOM IDs correspond 1:1 with the code.

[ English | [繁體中文 (TokenShield_README.zh-TW.md)](TokenShield_README.zh-TW.md) ]

> [!NOTE]
> This is the primary version of this document. If you update this file, please also update the Traditional Chinese translation `TokenShield_README.zh-TW.md`.

> [!IMPORTANT]
> **Disclaimer**: This document and the TokenShield tool itself provide only technical masking/restoration capability to reduce data-exposure risk when submitting content to AI; **neither constitutes legal advice, and using them alone does not guarantee compliance with GDPR or any other privacy regulation**. Whether your data handling is compliant still depends on your (or your organization's) overall data collection, processing, and use practices — consult professional legal advice if in doubt.

---

## 1. Overview

### 1.1 What is TokenShield?

**TokenShield** is a zero-trust **PII de-identification tool for individuals and businesses** — the international, non-education sibling of [EduShield](https://github.com/oas114/EduShield). The core problem it solves: before handing a document, email, or spreadsheet containing personal or confidential business data to an external AI (ChatGPT, Claude, etc.), you need to strip out the sensitive parts first. TokenShield provides a complete **Mask → Send to AI → Restore** workflow.

TokenShield is **not** a rewrite of EduShield — it's a separate sibling file that reuses the same proven engine (regex matching, session vault, restore logic) but swaps out the content layer: English UI, region-switchable PII rulesets (US / EU / UK + a Global baseline), and a Personal/Business persona toggle instead of Taiwan-education-specific rules.

### 1.2 Architecture & Security Properties

| Property | Detail |
|----------|--------|
| **Runtime environment** | A single static HTML file. Open it directly in a browser — no Node.js, server, or install step required. |
| **Data lifecycle** | All processing (regex matching, token replacement, restore) happens in browser RAM. Everything (`sessionVault`, `customDict`) is destroyed on page close or reload — nothing is written to disk or uploaded. |
| **No credentials** | No API key or account needed. The local AI (Ollama) integration is optional and only ever connects to `http://localhost:11434` (loopback). |
| **Clears on load** | `window.addEventListener('load', ...)` clears every `textarea` and `input[type="text"]` (except `ollamaUrl`/`ollamaModel`) on load, preventing browser autofill from leaking a previous session's data. |
| **CSS framework** | Tailwind CSS, loaded via a 3-tier fallback: CDN first, then the local `style.css` (see [dev/tailwind.config.js](../dev/tailwind.config.js)'s `content: ["../*.html"]` glob), then a fail-safe guidance screen if both fail. |
| **No persistence, by design** | Region and Persona selections, like everything else in the app, are **not saved**. See §2.1 below for how to change your startup defaults. |

---

## 2. Core Modules & Technical Spec

### 2.1 Region & Persona Presets

This is TokenShield's key structural difference from EduShield. Instead of one flat, Taiwan-specific rule set, detection rules and Hard Block keywords are organized into **presets** the user switches between at runtime via two `<select>` dropdowns in the header toolbar (`#regionSelect`, `#personaSelect`).

```javascript
const DEFAULT_REGION = "us";      // "us" | "eu" | "uk" | "tw" | "jp"
let currentRegion = DEFAULT_REGION;

const DEFAULT_PERSONA = "personal"; // "personal" | "business"
let currentPersona = DEFAULT_PERSONA;
```

**Nothing is persisted** — reloading the page always resets to `DEFAULT_REGION`/`DEFAULT_PERSONA`. If you always work in one region or persona, open `TokenShield.html` in a text editor, find these two constants near the top of the `<script>` block, and change the value. That becomes your new startup default.

Only **one** region preset is active at a time, layered on top of the always-on `global` baseline — this deliberately avoids cross-country format collisions (e.g. a bare 9-digit number shouldn't simultaneously be tested as a US SSN and something unrelated). Persona works the same way: only one keyword set is active at a time.

`getActiveRegexRules()` returns `[...regexPresets.global, ...regexPresets.custom, ...regexPresets[currentRegion]]`; `getHardBlockKeywords()` returns `[...hardBlockPresets.custom, ...hardBlockPresets[currentPersona]]`. Both are recomputed on every scan, so switching the dropdowns takes effect immediately (they call `triggerPreview()`). The `custom` bucket (new — see §2.8) always merges in regardless of the selected region/persona; built-in defaults live in the frozen `REGEX_PRESETS_DEFAULT`/`HARD_BLOCK_PRESETS_DEFAULT`, and `regexPresets`/`hardBlockPresets` are the live working copies these getters actually read.

### 2.2 Detection Rule Library (`REGEX_PRESETS_DEFAULT`)

Each rule object has the shape `{ type, regex, name, example, validate? }`. Tokens follow the `{{TYPE_N}}` format, where `N` is a per-type running counter.

**Priority logic** (unchanged from EduShield): in `extractStaticEntities()`, entities are sorted by (1) `isCritical` first (Hard Block terms win), then (2) string length descending (to stop short matches from clipping longer ones), then de-duplicated against an `occupied` position array so overlapping matches don't double-count.

| Preset | Category | Tag | Regex Summary | Example |
|--------|----------|-----|----------------|---------|
| `global` (always on) | Email | `EMAIL_N` | `[a-zA-Z0-9._%+-]+@[...]` | user@example.com |
| `global` | IPv4 | `IPV4_N` | standard dotted-quad | 192.168.1.1 |
| `global` | IPv6 | `IPV6_N` | full 8-group form | 2001:0db8:...:7334 |
| `global` | Credit Card | `CREDIT_CARD_N` | major-issuer prefixes + **Luhn checksum** post-filter | 4111 1111 1111 1111 |
| `global` | Intl. Phone | `INTL_PHONE_N` | `+CC ...` generic | +1 415 555 2671 |
| `global` | Gregorian Date | `DATE_N` | `YYYY[-/.]MM[-/.]DD` | 2026-08-19 |
| `us` | SSN | `US_SSN_N` | `\d{3}-\d{2}-\d{4}` | 123-45-6789 |
| `us` | EIN | `US_EIN_N` | `\d{2}-\d{7}` | 12-3456789 |
| `us` | Phone | `US_PHONE_N` | NANP format | (415) 555-2671 |
| `us` | ZIP | `US_ZIP_N` | 5 or 5+4 digit | 94105-1234 |
| `eu` | IBAN | `IBAN_N` | 2-letter country + checksum + BBAN | DE89370400440532013000 |
| `eu` | Passport (approx.) | `EU_PASSPORT_N` | 2 letters + 7 digits | PA1234567 |
| `eu` | VAT (approx.) | `EU_VAT_N` | 2 letters + 8-12 digits | DE123456789 |
| `uk` | National Insurance | `UK_NI_N` | official NI format | AB123456C |
| `uk` | Postcode | `UK_POSTCODE_N` | official postcode format | SW1A 1AA |
| `uk` | Phone | `UK_PHONE_N` | `+44`/`0` + national number | +44 7911 123456 |
| `tw` | National ID | `TW_ID_N` | `[A-Z][12]\d{8}` + **checksum** (`twIdChecksum()`) | A123456789 |
| `tw` | Resident Certificate / Uniform ID | `TW_ARC_N` | `[A-Z][A-D89]\d{8}` | A800000014 |
| `tw` | Mobile Number | `TW_MOBILE_N` | `09\d{2}-?\d{3}-?\d{3}` | 0912-345-678 |
| `tw` | Landline / Extension | `TW_TEL_N` | area code + number (optional extension) | 02-23456789#123 |
| `tw` | Business Unified Number | `TW_UBN_N` | 8 digits + **checksum** (`twUbnChecksum()`, post-2021/12/22 mod-5 rule) | 04595257 |
| `tw` | Household/Mailing Address | `TW_ADDRESS_N` | county/city + district + road + number | 406 台中市北屯區崇德路三段100號 |
| `jp` | My Number (個人番号) | `JP_MYNUMBER_N` | 12 digits + **checksum** (`jpMyNumberChecksum()`) | 1234-5678-9018 |
| `jp` | Corporate Number (法人番号) | `JP_CORP_NUMBER_N` | 13 digits + **checksum** (`jpCorpNumberChecksum()`) | 8700110005901 |
| `jp` | Mobile Number | `JP_MOBILE_N` | `0[789]0-?\d{4}-?\d{4}` | 090-1234-5678 |
| `jp` | Landline Number | `JP_TEL_N` | area code + number | 03-1234-5678 |
| `jp` | Postal Code | `JP_POSTAL_N` | `\d{3}-?\d{4}` | 123-4567 |
| `jp` | Passport (approx.) | `JP_PASSPORT_N` | 2 letters + 7 digits | TH1234567 |

> [!NOTE]
> The `eu` passport/VAT patterns and the `jp` passport pattern are **approximations**, not exhaustive legal-grade validators. Contributions that tighten these, or add more countries as new presets, are welcome — see §5.

The Credit Card rule additionally runs a **Luhn checksum** (`luhnCheck()`) against every regex match before accepting it, to cut down false positives from arbitrary 13-19 digit numbers. The `tw`/`jp` national ID, business unified number, My Number, and corporate number rules do the same with their own official checksum algorithms (`twIdChecksum()`/`twUbnChecksum()`/`jpMyNumberChecksum()`/`jpCorpNumberChecksum()`, defined next to `luhnCheck()`), substantially cutting false positives on these otherwise-loose numeric formats.

### 2.3 Hard Block Interlock (`HARD_BLOCK_PRESETS_DEFAULT`)

Two keyword arrays, `personal` and `business`, each ~20 terms. If the currently-active persona's list matches the input text, TokenShield locks the UI:
- A **red warning banner** appears
- The **copy button is disabled**
- The user must explicitly **acknowledge and unlock** via the Unlock modal

Unlike EduShield's Chinese keyword matching (which needs no case handling), English keyword matching in `extractStaticEntities()`'s `addEnt()` is **case-insensitive** — it lowercases both the text and the keyword to find the match position, then re-slices the *original-cased* substring out of the source text so the token report shows the text as it actually appeared (not the lowercased keyword).

### 2.4 Custom Dictionary (`customDict`)

Same mechanism as EduShield, now with a `source` field added to every entry (`builtin`/`auto-loaded`/`manual`/`overridden`/`ai-session`, shown as a badge in the UI — see §2.8): CSV upload, online table editor (paste from Excel supported), or manual "mark as sensitive" selection, plus entities the local AI returns. The CSV template header is `Keyword,Category(optional)` — the upload parser detects a header row by checking (case-insensitively) whether the first cell contains the word "keyword", and now parses each row with `parseCsvLine()` (standard CSV double-quote escaping) instead of a naive `split(',')`. CSV-file imports are routed through the Merge/Replace/Cancel dialog (§2.8) rather than blindly overwriting the dictionary; the online table editor keeps its original overwrite-everything behavior.

### 2.5 Local AI Module (Ollama)

Two channels, same wire protocol as EduShield (`{ollamaUrl}/api/generate`, streaming NDJSON, `{ollamaUrl}/api/tags` for connection tests). The `format` field is a full JSON Schema (not the loose `"json"` string) forcing a flat array with a 5-value `type` enum (`PERSON`/`VENDOR`/`ADDRESS`/`PROJECT`/`BANK_ACCT`) — this stops the model from returning a type-grouped object that gets silently reduced to one category by the fallback parser. Channel 1 fixes `options: { temperature: 0, num_ctx: 8192 }`; Channel 2 fixes `options: { num_ctx: 8192 }` only (temperature intentionally left at the model's default — see below). `num_ctx` is raised from Ollama's 2048 default so longer pasted text doesn't get silently truncated.

| Channel | Purpose | Persona-aware? |
|---------|---------|-----------------|
| 1 — Entity extraction | Finds names, vendors/organizations, addresses, project names, bank/payment accounts that static regex missed | No — entity types are universal |
| 2 — Risk assessment | Flags "extremely sensitive" narrative content | **Yes** — the risk category list sent to the model differs by `currentPersona` (self-harm/abuse/mental-health/immigration for `personal`; insider-info/M&A/layoffs/breach/litigation for `business`) |

> **Known accuracy limitation**: real-world testing shows Channel 1's name extraction is not reliable — small models like `qwen2.5:3b` frequently miss names, since they're a free-form, highly context-dependent entity type. The UI's passive hint (`layer1HintBanner`) points users at the Custom Dictionary for names instead of the deep scan for this reason; treat Channel 1's name output as best-effort, not authoritative.

> **Channel 2 runs 3 sequential calls and unions the results**: any single run returning `critical: true` marks the text as risky (a union, not a majority vote). The 3 runs deliberately keep the model's default `temperature` — not Channel 1's `temperature: 0` — otherwise all 3 runs would return near-identical output and the repetition would be wasted. Testing with real narrative text found softly-worded, indirect phrasing had only a ~20% single-call hit rate; repeating the call raises the cumulative catch rate. A failed run (timeout, dropped connection) is skipped and the remaining runs continue — an error only surfaces if all 3 fail. The button text switches from a live character count (Channel 1) to the current attempt number, e.g. `Confirming semantic risk (2/3)` (Channel 2), since Channel 2's output is too short for a character count to be meaningful.

Same safeguards as EduShield: pre-flight connection check, manual cancel, runaway-output length guard (`maxAllowedLength`), 3-minute timeout confirmation, and `finally`-block UI recovery.

### 2.6 Local AI Prompt Library

A small addition not present in EduShield: the Settings modal includes a **Local AI Prompt Library** panel with two ready-to-copy system prompt templates (`LOCAL_AI_PROMPTS.personal` / `.business`, copied via `copyLocalPrompt(which)`). These are meant to be pasted ahead of your masked text into any local model's chat (Ollama, LM Studio, etc.) so the model's own behavior stays privacy-aware too — reinforcing that nothing, including your prompt phrasing, needs to leave the machine.

> [!NOTE]
> `LOCAL_AI_PROMPTS` is unrelated to and never read by the actual Channel 1/2 calls in `processAnonymizePhase2()` — it's purely a copy-to-clipboard convenience for external chat tools. The real scan prompts live in `aiPrompts`/`AI_PROMPTS_DEFAULT`, editable via the Custom Protection Manager — see §2.8.

### 2.7 Restore & Integrity Verification

Unchanged from EduShield. `sessionVault` maps `{{TAG_N}}` tokens to their original values; `processRestore()` strips a leading system-instruction prefix (now the ASCII marker `[SYSTEM INSTRUCTION: ...]` instead of the Chinese `【系統指令：...】` marker EduShield uses), then applies a triple-tolerance match: exact, whitespace-tolerant, and bracket-tolerant (`[TAG_N]`, `(TAG_N)`, `【TAG_N】`). Missing tokens are reported and highlighted in red across both the left-hand original view and the chip list.

**Persistent Mapping Vault**: `sessionVault` is rebuilt from scratch on every run, so token numbering isn't stable across batches. The "Mapping Vault" button on the Restore tab exports the accumulated original-value ↔ token mapping (`persistentVault`) to a file — **unencrypted CSV** (three columns, Excel-readable, with a risk notice shown before export since the file is plain text) or **encrypted JSON** (Web Crypto PBKDF2 + AES-GCM, password-protected, unrecoverable if the password is lost). Importing it later (merge or full replace) lets already-known values reuse their previous token across sessions. This is manual/one-off only — nothing is auto-persisted to browser storage, and coordinate-based table masking tokens (`{{TAB_C...}}`) are excluded since they're positional, not identity-based.

### 2.8 Custom Protection Manager (Four Dimensions)

Click "**Manage Custom Protection Rules**" in the toolbar to customize, import, and export across four dimensions without hand-editing the HTML source: Roster/Dictionary (§2.4), Hard Block Keywords, Regex Rules, and Local AI Prompts (the real scan prompts, not the copy-to-clipboard library in §2.6).

**Scope**: CSV/config-file *import* still only ever writes to one **always-on `custom` bucket** per dimension (`hardBlockPresets.custom`, `regexPresets.custom`) — never a specific persona/region's own array, keeping collision handling simple (dedup only within `custom`, never silently rewriting a specific persona's or region's built-in list from an import that may not even target it). The **inline add/edit UI in this panel is a separate path**: the tables show the currently *active* set (`custom` plus the selected Persona for Hard Block; `custom` plus `global` plus the selected Region for Regex), and any row can be edited or deleted directly — including built-in rows, not just `custom` ones — tagged with a small `PERSONAL`/`BUSINESS`/`GLOBAL`/`US`/etc. bucket badge. Switching Region or Persona live-refreshes the panel (and the PII Rule Guide) if either is open.

Built-in defaults are frozen as `HARD_BLOCK_PRESETS_DEFAULT` / `REGEX_PRESETS_DEFAULT` / `AI_PROMPTS_DEFAULT`. Live working state:
```javascript
let hardBlockPresets = { personal: [], business: [], custom: [] };
let regexPresets = { global: [], us: [], eu: [], uk: [], tw: [], jp: [], custom: [] };
let aiPrompts = { channel1: '', channel2Personal: '', channel2Business: '' };
```
Each entry carries a `source` field shown as a badge: `Built-in` / `Config file import` / `Manually imported` / `Overridden`. Regex rules store `pattern`/`flags` as strings (not a live `RegExp`), reconstructed through the `tryCompileRegexRow()` try/catch guard on every use — a malformed rule is skipped and reported without aborting the rest of the scan.

Built-in entries also carry a `defaultValue`/`defaultPattern` field — a stable link back to the corresponding `HARD_BLOCK_PRESETS_DEFAULT`/`REGEX_PRESETS_DEFAULT` entry that survives edits. This powers two safety-net affordances so "Reset to Defaults" (which wipes `custom` too) isn't the only way back: a **"Revert to Default"** button on any edited/still-present built-in row, and a **"removed from defaults"** list below each table for re-adding a deleted built-in row individually (`revertHardBlockEntryAt()`/`restoreRemovedHardBlockDefault()` and their regex equivalents). Editing a regex row preserves the built-in `validate` checksum function (Luhn, Taiwan ID, Japan My Number, etc.) — it's matched back in by rule `type` rather than pattern text, since `type` is a stable identifier even when the pattern/name/example were edited. All of this is working-copy state in memory: it resets to true defaults on reload unless carried over via config file export/import (below), which is why "Reset to Defaults" and "Export/Import Config File" are described together.

CSV templates: Hard Block Keywords is a single `Keyword` column; Regex Rules is 4 columns (`TypeTag,RuleName,Pattern,ExampleText`), where `Pattern` accepts a bare pattern (defaults to flag `g`) or a full `/pattern/flags` literal-style string.

Importing a CSV shows a **Merge with Existing / Replace All / Cancel** dialog whenever the `custom` bucket already has entries:
- **Merge**: keep existing `custom` entries, append new ones; a regex whose **pattern string** (flags excluded) matches an existing `custom` entry, or a keyword/value that matches case-insensitively, is fully overwritten (tagged `Overridden`).
- **Replace All**: wipe the `custom` bucket for that dimension and use only the imported content.
- **Cancel**: no changes.

**Manual config file import**: an earlier version auto-loaded `tokenshield.config.js` from the same folder on startup via a dynamically-injected `<script src="tokenshield.config.js">`. That approach let a tampered file execute arbitrary code without the user noticing, contradicting the "PII never leaves the browser" trust claim — it was replaced on 2026-08-25 with a manual import flow:
```javascript
window.TOKENSHIELD_AUTO_CONFIG = {
  version: 1,
  roster: [ { type: "VENDOR", value: "...", reason: "..." } ],
  hardBlock: [ "custom hard-block term" ],
  regexRules: [ { type: "CUSTOM_CODE", pattern: "CODE-\\d{4}", flags: "g", name: "Custom code", example: "CODE-1234" } ],
  // Present only for a Persona/Region bucket that's actually been edited/deleted from —
  // see hardBlockBucketIsDefault()/regexBucketIsDefault(). Omitted entirely when untouched,
  // so an export from someone who never edited a built-in rule doesn't carry the whole library.
  hardBlockOverrides: { personal: [ "..." ] },
  regexOverrides: { us: [ { type: "US_SSN", pattern: "...", flags: "g", name: "...", example: "..." } ] },
  aiPrompts: { channel1: "...{{TEXT}}", channel2Personal: "...{{TEXT}}", channel2Business: "...{{TEXT}}" }
};
```
The three buttons live inside a collapsible "Advanced Settings: Import / Export Config File" section at the bottom of the "Manage Custom Protection Rules" panel (hidden below desktop viewport widths).

**Import Config File** opens a file picker; the selected file is parsed by string-scanning and `JSON.parse()` only (never `eval`, never executed), then shows the same **Merge with Existing / Replace All** dialog CSV import uses (`openConfigImportDialog()`), with entry counts per dimension. **The choice matters more here than for CSV**: merge keys `custom`-bucket entries on their own content (a regex's pattern text, a keyword's own text), so editing exactly that field breaks merge's ability to recognize "same rule, updated" — it adds a duplicate instead. `hardBlockOverrides`/`regexOverrides` make this sharper still: since the exported list is a flat array with no identity beyond its content, a **Merge** import folds new/changed values in additively (nothing is ever removed), while only **Replace All** actually reapplies a deletion or cleanly supersedes an edited value — reapplying an edited or deleted built-in rule after a reload requires Replace All. One identity caveat: an edited built-in value that doesn't exactly match a true default text comes back tagged `Overridden`/`auto-loaded` without its own `defaultValue`/`defaultPattern` link (i.e. no more per-row "Revert to Default" for it specifically) — the edit itself is correctly reapplied, just not the "what it looked like before this edit" breadcrumb. Regex overrides are matched back to their built-in `validate` function by `type`. This is a manual, one-shot action: it does **not** re-apply on refresh or the next launch, so the user re-imports each time; a persistent on-load notice nudges them to do so, worded without claiming a file was "detected" since browser security blocks background-checking whether one exists.

**Export Config File** packages the current in-memory state (all four dimensions, all merge/override results, excluding `ai-session`-tagged roster entries) into the same format, downloaded as the fixed filename `tokenshield.config.js`, for sharing with a colleague or reuse on another machine — `hardBlockOverrides`/`regexOverrides` are added only for buckets that differ from true defaults. **Reset to Defaults** (with a confirmation prompt) resets all four dimensions to built-in defaults, discarding this session's manual edits or imported settings.

> [!NOTE]
> This mechanism only ever touches rule/prompt configuration — never the actual document content typed into the app. The existing zero-persistence promise (everything destroyed on page close/reload, see §1.2) is untouched for `sessionVault` and the input textareas.

### 2.9 Display Language (i18n)

A `#langSelect` dropdown in the header toolbar switches the interface between **English / 繁體中文 / 日本語**, deliberately decoupled from `currentRegion`/`currentPersona` (§2.1) — Region/Persona pick which *rule data* is active, this only picks which *language* the chrome renders in, so e.g. running the EU ruleset with a Traditional Chinese interface is a supported combination.

```javascript
const LANG_STORAGE_KEY = 'tokenshield_display_lang';
let currentLang = localStorage.getItem(LANG_STORAGE_KEY) || 'en'; // persisted — unlike Region/Persona
const I18N = { someKey: { en: '...', zh: '...', ja: '...' }, /* ~220 keys */ };
function t(key, vars) { /* looks up I18N[key][currentLang], falls back to en, then the raw key; supports {placeholder} interpolation */ }
function applyLanguage(lang) { /* sets currentLang, persists it, walks data-i18n*, calls refreshDynamicText() */ }
```

Static markup opts in via `data-i18n` (sets `innerHTML`), `data-i18n-placeholder`, `data-i18n-title`, and `data-i18n-aria-label`. Dynamically-generated strings (toasts, `showConfirmModal()` messages, table renderers like `renderHardBlockMgmtTable()`/`renderRegexMgmtTable()`) call `t()` directly instead of embedding literal English. `refreshDynamicText()` re-runs the pure-render functions so a panel that's already open updates immediately on a language switch, without needing the user to reopen it. The PII Rule Guide no longer renders a live rule table (`renderGuideTable()` was removed when the guide became a static concept explainer, see §2.8) — `refreshGuideModalIfOpen()` is kept as a documented no-op so callers elsewhere don't need to know that.

**Deliberately left untranslated** (treated as technical/reference content): the `REGEX_PRESETS_DEFAULT`/`HARD_BLOCK_PRESETS_DEFAULT` catalog (rule `name`/`example` fields, Hard Block keyword lists — translating the keyword lists would change actual detection behavior, not just display), and the `LOCAL_AI_PROMPTS`/`aiPrompts` prompt text (these are instructions sent to another AI model, not UI copy). `localStorage` persistence is a deliberate exception to TokenShield's zero-persistence-by-design rule (§1.2) — it's a display preference, not document content or scan state.

---

## 3. User Manual

### 3.1 Requirements

| Item | Requirement |
|------|-------------|
| Browser | Chrome / Edge recommended (needs ES2020+, ReadableStream, Clipboard API) |
| Startup / network | **Normal offline**: just open `TokenShield.html`. **Closed intranet (no internet at all)**: keep `TokenShield.html` and `style.css` in the same folder. |
| Local AI (optional) | Install [Ollama](https://ollama.com/), pull `qwen2.5:3b` (recommended default — runs on modest hardware), set `OLLAMA_ORIGINS=*`. |

### 3.2 Standard Workflow

> [!IMPORTANT]
> **Zero-trust reminder**: for real personal or confidential data, always download and run the offline single-file copy. The GitHub Pages hosted version is for feature evaluation only.

1. **(Optional) Set Region & Persona** — top toolbar dropdowns. Defaults come from `DEFAULT_REGION`/`DEFAULT_PERSONA` in the source.
2. **(Optional) Import or build a custom dictionary** — CSV upload or the online table editor.
3. **Paste your data** into "Original Data Input". Detected items highlight live (200ms debounce) and list as chips below.
4. **(Optional) Manual masking** — select text for a "mark as sensitive" menu; for tab-delimited tables, click a cell for cell/column/row masking options.
5. **Click "Execute De-identification"** — the right panel shows the token mapping and the masked output, ready to copy. A Hard Block match locks the copy button until you review and unlock it.
6. **(Optional) "Scan with Local AI"** — runs both Ollama channels if configured.
7. **Copy masked data**, send it to ChatGPT/Claude, then switch to the **Restore** tab.
8. **Paste the AI's reply**, click **Run Restore** — tokens are matched back to their original values. Click any restored value to cycle through Show / Partial mask / Full mask.

### 3.3 Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| A term you expected wasn't masked | It's on the whitelist, or doesn't match any active rule/dictionary entry | Select it and choose "Mark as sensitive" |
| Switching Region doesn't seem to change anything | You need to re-run the scan (edit the input or click Execute again) — switching only changes which rules apply going forward | Re-run de-identification after switching |
| "Restore Result" shows missing items | The AI edited or dropped a token | Click the red-highlighted chip on the left and paste the actual text from the AI reply to apply a fix |
| Copy button greyed out | A Hard Block term was detected | Click the red banner, review, and force-unlock if it's a false positive |

---

## 4. Advanced / Developer Notes

### 4.1 Adding a Hard Block term
No source editing needed at all — use "Manage Custom Protection Rules" → "Hard Block Keywords" tab (§2.8): "+ Add Row" for a `custom` entry, or click directly into any row shown for the active Persona to edit or delete a built-in one, CSV import for bulk changes. This only affects your own working copy (session state, or your exported config file) — to change what ships as the `personal`/`business` **default** for everyone using this file, edit `HARD_BLOCK_PRESETS_DEFAULT` in `TokenShield.html` source instead.

### 4.2 Adding a detection rule
Use "Manage Custom Protection Rules" → "Regex Rules" tab (§2.8): "+ Add Row" for a `custom` entry, or click directly into any row shown for the active Region to edit or delete a built-in one (edits are validated for syntax and ReDoS risk on save), CSV import for bulk changes. Same scope note as above — this only affects your working copy; to change the shipped default, add `{ type: "TAG_NAME", regex: /your-regex/g, name: "Display Name", example: "Match Example" }` to `REGEX_PRESETS_DEFAULT`'s `global` array (always on) or a specific region array (`us`/`eu`/`uk`/`tw`/`jp`) in source.

### 4.3 Adding a new region preset
Add a new key to `REGEX_PRESETS_DEFAULT` (e.g. `ca` for Canada) and to the live-state initializer in `buildDefaultRegexPresetsState()`, then add a matching `<option>` to `#regionSelect` in the HTML. Rules in a new region are layered on top of `global` exactly like the existing five. `tw`/`jp` were added through this exact extension point and can be used as a reference.

### 4.4 Rebuilding `style.css`
The Tailwind build tooling lives in this repo's own `dev/` folder — no external dependency on any other project. After changing Tailwind classes in `TokenShield.html`:
```powershell
cd dev
npm install   # first time only
npm run build:css
```
This runs `tailwindcss -i ./input.css -o ../style.css --minify`, per the script in [dev/package.json](../dev/package.json).

### 4.5 Folder structure

```text
TokenShield/  (repo root)
├── TokenShield.html                 <- TokenShield app (this document's subject)
├── docs/
│   ├── TokenShield_README.md        <- This document (English-primary)
│   └── TokenShield_README.zh-TW.md  <- Traditional Chinese translation
├── README.md / README.zh-TW.md      <- Project introduction
├── LICENSE                          <- MIT License
├── .gitignore / .nojekyll
├── style.css                        <- ✅ Compiled Tailwind fallback (checked in)
└── dev/                             <- Tailwind build tooling
    ├── input.css / tailwind.config.js / package.json
```

> [!NOTE]
> When distributing to end users, only `TokenShield.html` and `style.css` are needed. Everything under `dev/` is for development only.

---

## 5. Contributing

Region coverage is intentionally limited to US / EU / UK + Global at launch — not every country, to avoid cross-format collisions (see §2.1). Pull requests that add well-scoped new region presets (with regex + a couple of realistic examples), tighten the approximate EU passport/VAT patterns, or add new Hard Block terms for a specific persona are welcome.

---

## 6. About

* **GitHub**: [oas114/TokenShield](https://github.com/oas114/TokenShield)
* **Sibling project**: [oas114/EduShield](https://github.com/oas114/EduShield) — the Taiwan education-focused edition, in its own separate repository
* **Author**: OA (oas114)
* **Support**: [Ko-fi](https://ko-fi.com/oasgrow)
* **License**: [MIT](../LICENSE)
