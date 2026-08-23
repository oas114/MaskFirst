# TokenShield Design Language

This document describes the visual design system used by `TokenShield.html`: the semantic color
tokens, the component-state conventions, and the rules for forms and destructive actions. It's a
reference for anyone (human or AI assistant) making UI changes to the tool, so the interface stays
internally consistent instead of drifting one raw Tailwind class at a time.

## 1. Design tokens

All colors are semantic Tailwind classes backed by CSS custom properties (defined once in an
inline `<style>` block in `TokenShield.html`'s `<head>`, and mirrored in `dev/tailwind.config.js`
so the same class names work whether the page is running off the Tailwind CDN or the compiled
`style.css` fallback — see the comment at the top of `dev/input.css` for why the variables live in
the HTML rather than in the CSS source).

Never write a raw Tailwind palette class (`bg-stone-100`, `text-emerald-700`, `border-red-300`,
etc.) or a literal hex code directly in a `class` or `style` attribute. Use the token instead. If a
shade you need doesn't have a token yet, add one to *both* the `<head>` variable block and
`dev/tailwind.config.js`'s `colors` object — they must stay in sync by hand.

| Token family | Purpose | Notes |
|---|---|---|
| `paper`, `paper-raised`, `paper-sunken`, `paper-hover` | Page and surface backgrounds | `paper` = page canvas, `paper-raised` = cards/modals (white), `paper-sunken` = card header/footer bars, `paper-hover` = hover backgrounds |
| `ink`, `ink-hover`, `ink-secondary`, `ink-muted`, `ink-faint` | Text and dark-surface hierarchy | Darkest → lightest. `ink-hover` doubles as the hover shade for `bg-ink` dark buttons/menus, not just a text tone |
| `line`, `line-strong`, `line-soft` | Borders | `line` = card/modal containers, `line-strong` = interactive elements (inputs, buttons, selects), `line-soft` = subtle inner dividers |
| `scrim` | Modal backdrop overlay, dark surfaces (right-click context menu) | Used with an opacity modifier, e.g. `bg-scrim/40` |
| `brand` (50–900) | Primary actions, active states, links, and — deliberately — the "success/info" role too | TokenShield's brand is **emerald**. Success/info were kept tied to brand color rather than split into a separate token; see the decision note below |
| `danger` (50–800) | Destructive actions, errors, critical findings | Merges the original red *and* rose usage into one family |
| `warning` (50–900) | Non-blocking warnings, caution states | |
| `accent-ai` (50–700, indigo) | The Layer-2 local-AI semantic-scan trigger button, and manual table-mask highlight chips | A brand-independent accent — same hue in both TokenShield and EduShield, since it marks "a secondary flagged action," not brand identity |
| `ai-finding` (50–700, purple) | Entity chips for things the local AI (not the static regex layer) detected | Also brand-independent and shared with EduShield — this is what visually tells a user "this highlight came from the AI pass," distinct from `accent-ai`'s button/action role |

**Decision note (success = brand):** unlike a typical design system that gives "success" its own
green independent of brand color, TokenShield's brand *is* emerald, so a separate success token
would just be a second shade of green with no practical distinction. Kept as one token
deliberately — revisit only if TokenShield's brand color ever changes to something success
wouldn't suit.

**RGB-triplet variables** (`--brand-500-rgb`, `--danger-500-rgb`, `--ai-finding-500-rgb`) exist
alongside the hex variables specifically so `rgba(var(--brand-500-rgb), 0.18)`-style translucent
backgrounds (used for text-highlight `<mark>` styling) can share the same source-of-truth color
without hardcoding a second copy of the value.

**Dark mode:** not implemented yet, but the CSS-variable architecture is set up for it — a dark
theme would only need to redefine the `<head>` variable block (behind a `prefers-color-scheme` or
`[data-theme]` selector), never touch a class name.

## 2. Component states

Every interactive element should cover, at minimum: `default`, `hover`, `focus-visible`,
`disabled`. Add `loading` only where there's a real asynchronous wait (currently just the Layer-2
AI scan button).

- Focus rings use `focus:ring-1 focus:ring-brand-500` (or `focus:ring-2 focus:ring-brand-100` for
  the larger text areas) — never remove `outline`/`ring` without substituting the other.
- Don't invent a new hover/active pattern per component; reuse the existing ramp-step relationship
  (e.g. a `bg-brand-700` button hovers to `bg-brand-800`; a `bg-ink` dark surface hovers to
  `bg-ink-hover`).

## 3. Forms, validation, and destructive actions

- **No native `alert()` / `confirm()`.** Use the two helpers defined near `openModal`/`closeModal`
  in the main `<script>` block:
  - `showToast(message, variant)` — non-blocking notification, auto-dismisses after 4s. `variant`
    is one of `info | success | danger | warning`. Use this for anything that was previously a
    fire-and-forget `alert()`.
  - `showConfirmModal(message, opts)` — returns a `Promise<boolean>`, styled like the app's other
    modals. Pass `{ danger: true }` for a destructive confirmation (colors the OK button red) and
    custom `okLabel`/`cancelLabel` text instead of generic "OK"/"Cancel" wherever a more specific
    verb reads better (e.g. `okLabel: 'Reload'`, not "OK").
- **Confirm before it's destroyed, not before it's created.** Deleting a rule, clearing a table, or
  reloading defaults over manually-edited state warrants `showConfirmModal`. Adding a new rule or
  dictionary entry does not — it's cheap to undo (just delete it), so a confirmation step would
  only add friction for the common case.
- **Batch validation follows the CSV-import pattern.** When importing a CSV of regex rules, each
  row is compiled with `tryCompileRegexRow()` and failures are listed inline (row number, the
  offending pattern, and `err.message`) in the import modal — never silently dropped. If a new
  bulk-entry flow is added anywhere else, follow this same shape rather than inventing a new one.
  (Note: regex rules can currently only be added via CSV import, not a single-row text input, so
  there's no separate single-field regex validation UI to keep in sync with this.)

## 4. Keeping this in sync with EduShield

EduShield's `docs/DESIGN_LANGUAGE.md` (Traditional Chinese) documents the same system with one
intentional difference: its `brand` ramp is blue instead of emerald. Everything else — token
names, the state rules, the forms/destructive-action rules — should stay identical between the two
documents. If you change one side, change the other.
