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
> This is the **English-primary** documentation set — deliberately flipped from EduShield's Chinese-primary convention, since TokenShield is positioned as the international edition. If you update this file, update the Chinese counterpart `TokenShield_README.zh-TW.md` as well.

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

Two channels, same wire protocol as EduShield (`{ollamaUrl}/api/generate`, streaming NDJSON, `{ollamaUrl}/api/tags` for connection tests):

| Channel | Purpose | Persona-aware? |
|---------|---------|-----------------|
| 1 — Entity extraction | Finds names, vendors/organizations, addresses, project names, bank/payment accounts that static regex missed | No — entity types are universal |
| 2 — Risk assessment | Flags "extremely sensitive" narrative content | **Yes** — the risk category list sent to the model differs by `currentPersona` (self-harm/abuse/mental-health/immigration for `personal`; insider-info/M&A/layoffs/breach/litigation for `business`) |

> **Known accuracy limitation**: real-world testing shows Channel 1's name extraction is not reliable — small models like `qwen2.5:3b` frequently miss names, since they're a free-form, highly context-dependent entity type. The UI's passive hint (`layer1HintBanner`) points users at the Custom Dictionary for names instead of the deep scan for this reason; treat Channel 1's name output as best-effort, not authoritative.

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

**Scope decision**: unlike the region/persona-specific built-in presets, every custom/imported/auto-loaded Hard Block keyword and regex rule lands in one **always-on `custom` bucket** per dimension (`hardBlockPresets.custom`, `regexPresets.custom`) — never inside a specific persona/region's own array. This keeps collision handling simple (dedup only within `custom`, never silently rewriting a specific persona's or region's built-in list) and matches the fact `customDict` has always been a single flat, non-scoped array. The management tables in this panel list only the `custom` bucket; built-in region/persona rules remain visible (read-only) in the PII Rule Guide.

Built-in defaults are frozen as `HARD_BLOCK_PRESETS_DEFAULT` / `REGEX_PRESETS_DEFAULT` / `AI_PROMPTS_DEFAULT`. Live working state:
```javascript
let hardBlockPresets = { personal: [], business: [], custom: [] };
let regexPresets = { global: [], us: [], eu: [], uk: [], tw: [], jp: [], custom: [] };
let aiPrompts = { channel1: '', channel2Personal: '', channel2Business: '' };
```
Each entry carries a `source` field shown as a badge: `Built-in` / `Auto-loaded` / `Manually imported` / `Overridden`. Regex rules store `pattern`/`flags` as strings (not a live `RegExp`), reconstructed through the `tryCompileRegexRow()` try/catch guard on every use — a malformed rule is skipped and reported without aborting the rest of the scan. Note: the built-in `CREDIT_CARD` rule's Luhn `validate` function is preserved for built-in entries, but is necessarily lost if a CSV/config import happens to override it (CSV/JSON can't carry a function) — the import dialog surfaces a warning when this happens.

CSV templates: Hard Block Keywords is a single `Keyword` column; Regex Rules is 4 columns (`TypeTag,RuleName,Pattern,ExampleText`), where `Pattern` accepts a bare pattern (defaults to flag `g`) or a full `/pattern/flags` literal-style string.

Importing a CSV shows a **Merge with Existing / Replace All / Cancel** dialog whenever the `custom` bucket already has entries:
- **Merge**: keep existing `custom` entries, append new ones; a regex whose **pattern string** (flags excluded) matches an existing `custom` entry, or a keyword/value that matches case-insensitively, is fully overwritten (tagged `Overridden`).
- **Replace All**: wipe the `custom` bucket for that dimension and use only the imported content.
- **Cancel**: no changes.

**Same-folder auto-load**: on startup, `<script src="tokenshield.config.js">` is dynamically injected (same pattern as the Tailwind CDN/local-`style.css` degradation IIFE) — absent file is silently skipped, a broken file logs a console warning without crashing, and a valid file is merged into the `custom` buckets and prompts (tagged `Auto-loaded`):
```javascript
window.TOKENSHIELD_AUTO_CONFIG = {
  version: 1,
  roster: [ { type: "VENDOR", value: "...", reason: "..." } ],
  hardBlock: [ "custom hard-block term" ],
  regexRules: [ { type: "CUSTOM_CODE", pattern: "CODE-\\d{4}", flags: "g", name: "Custom code", example: "CODE-1234" } ],
  aiPrompts: { channel1: "...{{TEXT}}", channel2Personal: "...{{TEXT}}", channel2Business: "...{{TEXT}}" }
};
```
Both actions live inside a collapsible "Advanced Settings: Auto-load Config File" section at the bottom of the "Manage Custom Protection Rules" panel — hidden below desktop viewport widths (the same-folder workflow isn't practical on mobile), and showing an inline notice when the page is loaded via a URL instead of a local file, since the feature only takes effect for local `file://` usage. The "**Reload Config**" button there (with a confirmation prompt) resets all four dimensions to built-in defaults and re-runs the auto-load step — discarding this session's manual edits. The "**Export as Auto-load File**" button packages the current in-memory state (all four dimensions, all merge/override results, excluding `ai-session`-tagged roster entries) into the same format, downloaded as the fixed filename `tokenshield.config.js`.

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

Static markup opts in via `data-i18n` (sets `innerHTML`), `data-i18n-placeholder`, `data-i18n-title`, and `data-i18n-aria-label`. Dynamically-generated strings (toasts, `showConfirmModal()` messages, table renderers like `renderHardBlockMgmtTable()`/`renderRegexMgmtTable()`/`renderGuideTable()`) call `t()` directly instead of embedding literal English. `refreshDynamicText()` re-runs the pure-render functions so a panel that's already open updates immediately on a language switch, without needing the user to reopen it.

**Deliberately left untranslated** (treated as technical/reference content, same reasoning as why the "Regular Expression" column in the PII Rule Guide is never translated): the `REGEX_PRESETS_DEFAULT`/`HARD_BLOCK_PRESETS_DEFAULT` catalog (rule `name`/`example` fields, Hard Block keyword lists — translating the keyword lists would change actual detection behavior, not just display), and the `LOCAL_AI_PROMPTS`/`aiPrompts` prompt text (these are instructions sent to another AI model, not UI copy). `localStorage` persistence is a deliberate exception to TokenShield's zero-persistence-by-design rule (§1.2) — it's a display preference, not document content or scan state.

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
| "Restore Result" shows missing items | The AI edited or dropped a token | Check the red-highlighted chips on the left and fix manually |
| Copy button greyed out | A Hard Block term was detected | Click the red banner, review, and force-unlock if it's a false positive |

---

## 4. Advanced / Developer Notes

### 4.1 Adding a Hard Block term
No source editing needed for a personal customization — use "Manage Custom Protection Rules" → "Hard Block Keywords" tab (§2.8) to import a CSV. To change the built-in `personal`/`business` lists themselves, open `TokenShield.html`, find `HARD_BLOCK_PRESETS_DEFAULT`, and add a string to the appropriate array.

### 4.2 Adding a detection rule
Use "Manage Custom Protection Rules" → "Regex Rules" tab (§2.8) to import a 4-column CSV. To change a built-in region's rules directly, find `REGEX_PRESETS_DEFAULT`, and add `{ type: "TAG_NAME", regex: /your-regex/g, name: "Display Name", example: "Match Example" }` to the `global` array (always on) or a specific region array (`us`/`eu`/`uk`/`tw`/`jp`).

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
