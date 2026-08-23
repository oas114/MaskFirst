# TokenShield — Full Technical Reference

<!--
[AI_SYSTEM_INSTRUCTIONS_START]
ATTENTION ALL AI ASSISTANTS:
If you modify `TokenShield.html` and your modifications include ANY changes, additions, or removals of Tailwind CSS utility classes, you MUST recompile the local CSS fallback file.
Failure to do so will break the application for users in offline environments.

IMPORTANT: as of the parent-workspace-folder restructure, the Tailwind build tooling (`dev/`) is SHARED and lives ONE LEVEL UP from this repo, not inside it — this repo (`TokenShield/`) is a sibling of `EduShield/`, both under the same parent workspace folder.

To recompile:
1. Open a terminal, navigate to the shared `dev/` folder in the parent workspace folder (one level up from this repo root).
2. Run the build script: `npm run build:css`
(This compiles a master `style.css` in the parent folder AND automatically copies it into both `TokenShield/style.css` and `EduShield/style.css` — see the parent folder's `CLAUDE.md` for the full convention.)
[AI_SYSTEM_INSTRUCTIONS_END]
-->

> **Version basis**: this document is written against the actual `TokenShield.html` source. All function names, constants, and DOM IDs correspond 1:1 with the code.

[ English | [繁體中文 (TokenShield_README.zh-TW.md)](TokenShield_README.zh-TW.md) ]

> [!NOTE]
> This is the **English-primary** documentation set — deliberately flipped from EduShield's Chinese-primary convention, since TokenShield is positioned as the international edition. If you update this file, update the Chinese counterpart `TokenShield_README.zh-TW.md` as well.

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
| **CSS framework** | Tailwind CSS, loaded via a 3-tier fallback: CDN first, then the local `style.css` (compiled by a shared, parent-workspace-level Tailwind pipeline — see §4.4), then a fail-safe guidance screen if both fail. |
| **No persistence, by design** | Region and Persona selections, like everything else in the app, are **not saved**. See §2.1 below for how to change your startup defaults. |

---

## 2. Core Modules & Technical Spec

### 2.1 Region & Persona Presets

This is TokenShield's key structural difference from EduShield. Instead of one flat, Taiwan-specific rule set, detection rules and Hard Block keywords are organized into **presets** the user switches between at runtime via two `<select>` dropdowns in the header toolbar (`#regionSelect`, `#personaSelect`).

```javascript
const DEFAULT_REGION = "us";      // "us" | "eu" | "uk"
let currentRegion = DEFAULT_REGION;

const DEFAULT_PERSONA = "personal"; // "personal" | "business"
let currentPersona = DEFAULT_PERSONA;
```

**Nothing is persisted** — reloading the page always resets to `DEFAULT_REGION`/`DEFAULT_PERSONA`. If you always work in one region or persona, open `TokenShield.html` in a text editor, find these two constants near the top of the `<script>` block, and change the value. That becomes your new startup default.

Only **one** region preset is active at a time, layered on top of the always-on `global` baseline — this deliberately avoids cross-country format collisions (e.g. a bare 9-digit number shouldn't simultaneously be tested as a US SSN and something unrelated). Persona works the same way: only one keyword set is active at a time.

`getActiveRegexRules()` returns `[...REGEX_PRESETS.global, ...REGEX_PRESETS[currentRegion]]`; `getHardBlockKeywords()` returns `HARD_BLOCK_PRESETS[currentPersona]`. Both are recomputed on every scan, so switching the dropdowns takes effect immediately (they call `triggerPreview()`).

### 2.2 Detection Rule Library (`REGEX_PRESETS`)

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

> [!NOTE]
> The `eu` passport/VAT patterns are **approximations**, not exhaustive legal-grade validators (EU member states don't share one uniform format). Contributions that tighten these, or add more countries as new presets, are welcome — see §5.

The Credit Card rule additionally runs a **Luhn checksum** (`luhnCheck()`) against every regex match before accepting it, to cut down false positives from arbitrary 13-19 digit numbers.

### 2.3 Hard Block Interlock (`HARD_BLOCK_PRESETS`)

Two keyword arrays, `personal` and `business`, each ~20 terms. If the currently-active persona's list matches the input text, TokenShield locks the UI:
- A **red warning banner** appears
- The **copy button is disabled**
- The user must explicitly **acknowledge and unlock** via the Unlock modal

Unlike EduShield's Chinese keyword matching (which needs no case handling), English keyword matching in `extractStaticEntities()`'s `addEnt()` is **case-insensitive** — it lowercases both the text and the keyword to find the match position, then re-slices the *original-cased* substring out of the source text so the token report shows the text as it actually appeared (not the lowercased keyword).

### 2.4 Custom Dictionary (`customDict`)

Unchanged from EduShield's mechanism: CSV upload, online table editor (paste from Excel supported), or manual "mark as sensitive" selection, plus entities the local AI returns. The CSV template header is `Keyword,Category(optional)` — the upload parser detects a header row by checking (case-insensitively) whether the first cell contains the word "keyword".

### 2.5 Local AI Module (Ollama)

Two channels, same wire protocol as EduShield (`{ollamaUrl}/api/generate`, streaming NDJSON, `{ollamaUrl}/api/tags` for connection tests):

| Channel | Purpose | Persona-aware? |
|---------|---------|-----------------|
| 1 — Entity extraction | Finds names, vendors/organizations, addresses, project names, bank/payment accounts that static regex missed | No — entity types are universal |
| 2 — Risk assessment | Flags "extremely sensitive" narrative content | **Yes** — the risk category list sent to the model differs by `currentPersona` (self-harm/abuse/mental-health/immigration for `personal`; insider-info/M&A/layoffs/breach/litigation for `business`) |

Same safeguards as EduShield: pre-flight connection check, manual cancel, runaway-output length guard (`maxAllowedLength`), 3-minute timeout confirmation, and `finally`-block UI recovery.

### 2.6 Local AI Prompt Library

A small addition not present in EduShield: the Settings modal includes a **Local AI Prompt Library** panel with two ready-to-copy system prompt templates (`LOCAL_AI_PROMPTS.personal` / `.business`, copied via `copyLocalPrompt(which)`). These are meant to be pasted ahead of your masked text into any local model's chat (Ollama, LM Studio, etc.) so the model's own behavior stays privacy-aware too — reinforcing that nothing, including your prompt phrasing, needs to leave the machine.

### 2.7 Restore & Integrity Verification

Unchanged from EduShield. `sessionVault` maps `{{TAG_N}}` tokens to their original values; `processRestore()` strips a leading system-instruction prefix (now the ASCII marker `[SYSTEM INSTRUCTION: ...]` instead of the Chinese `【系統指令：...】` marker EduShield uses), then applies a triple-tolerance match: exact, whitespace-tolerant, and bracket-tolerant (`[TAG_N]`, `(TAG_N)`, `【TAG_N】`). Missing tokens are reported and highlighted in red across both the left-hand original view and the chip list.

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
Open `TokenShield.html`, find `HARD_BLOCK_PRESETS`, and add a string to the `personal` or `business` array.

### 4.2 Adding a detection rule
Find `REGEX_PRESETS`, and add `{ type: "TAG_NAME", regex: /your-regex/g, name: "Display Name", example: "Match Example" }` to the `global` array (always on) or a specific region array (`us`/`eu`/`uk`).

### 4.3 Adding a new region preset
Add a new key to `REGEX_PRESETS` (e.g. `ca` for Canada), then add a matching `<option>` to `#regionSelect` in the HTML. Rules in a new region are layered on top of `global` exactly like the existing three.

### 4.4 Rebuilding `style.css`
`TokenShield.html` and `EduShield.html` **share one Tailwind build pipeline**, but each repo keeps its own compiled `style.css`. The shared `dev/` folder lives at the **parent workspace folder level** (one level up from this repo, alongside the `EduShield/` sibling repo) — it is not part of either git repo. Its `tailwind.config.js` scans both apps: `content: ["../EduShield/*.html", "../TokenShield/*.html"]`. After changing Tailwind classes in either app:
```powershell
cd ../dev
npm run build:css
```
This runs `tailwindcss -i ./input.css -o ../style.css --minify && node copy-css.js` — it compiles a master `style.css` in the parent folder, then `copy-css.js` copies it into both `../TokenShield/style.css` and `../EduShield/style.css`. **This copy step is required** — a repo only ships the copy that physically lives inside it.

### 4.5 Folder structure (parent workspace layout)

```text
(parent workspace folder, not a git repo)/
├── dev/                              <- Shared Tailwind build tooling (not part of either repo)
│   ├── input.css / tailwind.config.js / package.json / copy-css.js
├── style.css                         <- Parent-level master compiled output
├── public/                           <- For oasgrow.com, deployed by a separate mechanism
│   ├── TokenShield/index.html        <- English interactive manual / landing page
│   └── EduShield/index.html          <- EduShield's interactive manual (Chinese)
├── TokenShield/                      <- ✅ This repo
│   ├── TokenShield.html              <- TokenShield app (this document's subject)
│   ├── TokenShield_README.md         <- This document (English-primary)
│   ├── TokenShield_README.zh-TW.md   <- Traditional Chinese translation
│   └── style.css                     <- Compiled Tailwind fallback (copied in from the parent build)
└── EduShield/                        <- Sibling repo (Taiwan education edition, independent GitHub repo)
```

> [!NOTE]
> When distributing to end users, only this repo's `TokenShield.html` and `style.css` are needed. `dev/` lives outside this repo and is for development only.

---

## 5. Contributing

Region coverage is intentionally limited to US / EU / UK + Global at launch — not every country, to avoid cross-format collisions (see §2.1). Pull requests that add well-scoped new region presets (with regex + a couple of realistic examples), tighten the approximate EU passport/VAT patterns, or add new Hard Block terms for a specific persona are welcome.

---

## 6. About

* **GitHub**: [oas114/TokenShield](https://github.com/oas114/TokenShield)
* **Sibling project**: [oas114/EduShield](https://github.com/oas114/EduShield) — the Taiwan education-focused edition, in its own separate repository
* **Author**: OA (oas114)
* **Support**: [Ko-fi](https://ko-fi.com/oasgrow)
* **License**: [MIT](./LICENSE)
