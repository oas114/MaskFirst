[ English | [繁體中文 (README.zh-TW.md)](README.zh-TW.md) ]

# 🛡️ TokenShield — Zero-Trust PII De-identification for AI Workflows

> *"Mask sensitive data before sending it to AI. Restore it in one click. 100% in-browser — no data ever leaves your device."*

> **Disclaimer**: This tool provides technical masking/restoration capability to reduce data-exposure risk; **it does not constitute legal advice, and using it alone does not guarantee compliance with GDPR or any other privacy regulation**. Whether your data handling is compliant still depends on your (or your organization's) overall data collection, processing, and use practices.

**Author:** OA (oas114) | **[Support this project (Ko-fi)](https://ko-fi.com/oasgrow)**

---

✨ **[Try the Interactive Manual](https://oasgrow.com/TokenShield/)** ✨
*We highly recommend exploring the interactive manual first to experience how TokenShield works.*

---

## Why TokenShield?

When you paste an employee performance review, an HR incident report, or a client email into ChatGPT to polish the wording, do you actually know what happens to that text afterward?

Most people don't — and the answer depends on which AI, and which plan, you're using.

**TokenShield** removes the guesswork. Mask the sensitive parts before you send anything — the original data never leaves your device. **Mask → Send → Restore**, entirely inside your browser.

TokenShield is the international, individual/business-focused sibling of [EduShield](https://github.com/oas114/EduShield), which targets Taiwan's education sector specifically. Both share the same zero-trust engine; TokenShield swaps in an English UI, switchable regional rule presets, and a Personal/Business persona toggle.

---

## Technical Architecture

### Zero-Trust, Pure Frontend Design

TokenShield is a **single static HTML file**. There is no backend, no database, no API server. Everything — regex matching, token replacement, restoration, and AI interaction — runs in your **local browser RAM**.

| Property | Detail |
|----------|--------|
| **Data Persistence** | Zero. All data — including your Region/Persona selection — is destroyed on page close or refresh. No disk writes, no cloud uploads. |
| **Credentials** | None required. No API keys, no accounts. |
| **Network** | Fully optional. Works completely offline. Local AI (Ollama) connects only to `http://localhost:11434` (loopback). |
| **Startup Safety** | All input fields are cleared on load to prevent browser autofill from leaking previous session data. |

### How Do I Know This Is Actually Safe? Ask an AI.

You don't need to read code, and you shouldn't just take our word for it. TokenShield is a single HTML file with no background network calls and no dependencies to install — copy the entire source of `TokenShield.html` and paste it into any AI you already use (ChatGPT, Claude, etc.), then ask it directly: "Does this page send any user input to a server?" Let the AI verify it for you, rather than trusting the developer's claims alone. The in-app "PII Rule Guide" carries the same reminder.

### Region & Persona Presets

Unlike a one-size-fits-all rule set, TokenShield lets you switch:
- **Region** (top toolbar): 🇺🇸 US / 🇪🇺 EU (GDPR) / 🇬🇧 UK / 🇹🇼 Taiwan / 🇯🇵 Japan — each layered on top of an always-on **Global** baseline (email, IP addresses, credit card numbers). Only one region is active at a time, to avoid cross-country format collisions.
- **Persona** (top toolbar): **Personal** or **Business** — swaps the Hard Block keyword set between personal-privacy terms and corporate-confidentiality terms.

Nothing is saved between sessions — see [TokenShield_README.md](./docs/TokenShield_README.md) for how to change your permanent default by editing two constants in the source.

### Display Language

A separate **language selector** (top toolbar) switches the interface between **English / 繁體中文 / 日本語**, independent from the Region ruleset above — Region decides which regex rule set is active, language only decides what the interface text looks like, so e.g. a Taiwan-based business processing EU data can still run the EU ruleset while reading the UI in Traditional Chinese. Your choice is remembered across sessions via `localStorage`, and defaults to English.

### Hybrid Defense Engine

**Layer 1 — Static Regex Fast-Match**: built-in `REGEX_PRESETS` cover national IDs, phone numbers, IBANs, credit cards, and more — see the in-app "PII Rule Guide" for the full, currently-active list.

**Layer 2 — Local LLM Semantic Scan** *(Optional)*: integrates with [Ollama](https://ollama.com/) running on your own machine. No data ever leaves your device. Entity extraction and risk-assessment prompts are persona-aware.

> **Practical limits of name detection**: small local models are not yet consistently reliable at recognizing person names — a free-form, highly context-dependent entity type — so name masking should not rely on this layer alone. If your documents contain fixed, recurring names, add them directly to the **Custom Dictionary** (manageable from the toolbar) to guarantee they're masked every time.

### Session Vault & Restore Mechanism

Every masked item is stored in an in-memory `sessionVault` as a unique token (`{{TYPE_N}}`). After the external AI processes the masked text, TokenShield restores original data using a **triple-tolerance matching algorithm** (exact, whitespace-tolerant, bracket-tolerant).

### Hard Block Interlock

The active persona's `HARD_BLOCK_PRESETS` list contains extremely sensitive terms. If any are detected:
- A **red warning banner** appears at the top of the UI
- The **copy button is locked** — preventing accidental data exfiltration
- The user must explicitly **acknowledge and unlock** before proceeding

---

## Deployment Modes

| Mode | Use Case | Files Needed |
|------|----------|-------------|
| **Online Sandbox** | Quick feature evaluation — *never use with real data* | None (browser-based) |
| **Offline Single-File** *(Recommended)* | Everyday use with real documents | `TokenShield.html` only |
| **Air-Gapped / No Internet** | Locked-down machines, closed intranets | `TokenShield.html` + `style.css` (in the same folder) |

> [!WARNING]
> Any online-hosted version is for **evaluation only**. For any real personal data or confidential records, always use the **offline single-file mode**.

---

## Core Workflow

```
1. Paste & Mask
   Paste your document → System auto-detects & replaces PII with tokens (e.g., {{PERSON_1}})

2. Send to AI
   Click "Copy Masked Data" → Paste into ChatGPT / Claude → AI processes safely

3. Restore
   Paste AI reply back → Click "Run Restore" → All tokens replaced with originals
```

---

## Localization & Contributing

TokenShield ships with US / EU / UK / Taiwan / Japan region presets — not every country, by design (see [TokenShield_README.md](./docs/TokenShield_README.md) for why). We welcome Pull Requests to:
- Add a new region preset (regex + a couple of realistic examples)
- Tighten the approximate EU passport/VAT patterns
- Contribute new Hard Block terms for the Personal or Business persona
- Translate the UI or documentation into additional languages

---

## Documentation

| Document | Description |
|----------|--------------|
| 📖 [TokenShield_README.md](./docs/TokenShield_README.md) | Full technical reference — modules, APIs, data structures, developer guide (English, primary) |
| 📖 [TokenShield_README.zh-TW.md](./docs/TokenShield_README.zh-TW.md) | Full technical reference (Traditional Chinese) |

---

## Roadmap

TokenShield is under active development. Planned directions include (in priority order):

1. **Persistent Mask Mapping Storage**: A way to save the mask ↔ original mapping so a session can be resumed later.
2. **File Upload Masking**: Direct masking support for uploaded Excel, Word, and PDF files, not just pasted text.
3. **Enterprise-grade Features**: Audit logs and related governance features for the Business persona.

---

## License & Author

- **License**: [MIT License](./LICENSE)
- **Author**: OA (oas114)
- **Support**: [Buy me a coffee on Ko-fi](https://ko-fi.com/oasgrow)
- **Contact**: oasgrow [at] gmail.com — open to partnerships, institutional inquiries, and feature feedback
- **Sibling project**: [EduShield](https://github.com/oas114/EduShield) — the Taiwan education-focused edition
