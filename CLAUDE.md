# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A one-page lead-generation site for "Horizon Wealth Planning", a (fictional) independent financial planner in Singapore. The whole site is a single `index.html`, with CSS in one `<style>` tag and JS in one `<script>` tag (in `<head>`). This is a hard constraint from the brief: use only vanilla HTML, CSS and JS, with no frameworks, build tools, package.json or extra files. Keep new code inside `index.html`.

## Commands

There is no build, lint or test tooling.

- Open in a browser: `python -m http.server 8000` then visit http://localhost:8000 (or `start index.html`)
- Syntax-check the JS: `sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > "$TEMP/hw.js" && node --check "$TEMP/hw.js"`
- **Regenerate the CSP hashes (required after ANY edit to the `<style>` or `<script>` block):**

```bash
node - <<'EOF'
const fs = require("fs"), crypto = require("crypto");
let html = fs.readFileSync("index.html", "utf8");
const hash = s => "'sha256-" + crypto.createHash("sha256").update(s.replace(/\r\n?/g, "\n"), "utf8").digest("base64") + "'";
const js = html.match(/<script>([\s\S]*?)<\/script>/)[1];
const css = html.match(/<style>([\s\S]*?)<\/style>/)[1];
html = html.replace(/script-src '[^']+'/, "script-src " + hash(js)).replace(/style-src '[^']+'/, "style-src " + hash(css));
fs.writeFileSync("index.html", html);
console.log("CSP hashes updated");
EOF
```

If you skip this, the browser blocks the whole script or stylesheet and the page breaks silently. Both this and the syntax check find the blocks by the literal strings `<script>` and `<style>`, so never write those literal tags anywhere else in the file (including comments). The JSON-LD block is `<script type="application/ld+json">`, which the regex does not match and CSP does not govern.

The page needs internet access only for Google Fonts (IBM Plex Sans Condensed for headings, IBM Plex Sans for body). There are no third-party images or scripts.

## Architecture

The file follows the same order in all three layers, and each major block is marked with a `/* ===== N. NAME ===== */` banner comment (`<!-- ===== -->` in the HTML). Keep that convention when adding sections. Page order: header, hero + retirement calculator, stats, services, process, checklist, testimonials, FAQ, contact, footer (with privacy notice), lunch talk dialog, sticky mobile CTA, back to top, chat widget.

**CSS**
- Design tokens are custom properties on `:root`: `--red`, `--red-strong`, `--red-soft`, `--on-red`, `--ink`, `--muted`, `--bg`, `--surface`, `--surface-2`, `--line`, `--line-strong`, the spacing scale `--space-1…8`, `--radius-sm/--radius/--radius-lg`, and `--nav-h`. Use the tokens and don't hardcode colors.
- **Dark mode:** the dark token values are defined twice and must stay identical: once in `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) }` (follows the OS) and once in `:root[data-theme="dark"]` (manual toggle). In dark mode `--red` is a lighter pink-red and `--on-red` is dark, so always pair `background: var(--red)` with `color: var(--on-red)`.
- Styles are mobile-first. The only breakpoints are `min-width: 768px` and `min-width: 1024px`, grouped near the end of the `<style>` tag. The desktop nav appears at 1024px (hamburger below that).
- A `prefers-reduced-motion` block turns off transitions and animations. The JS also checks `reduceMotion` and skips the counters and autoplay when it's set.
- Never add inline `style="..."` attributes: the CSP blocks them. Setting `el.style.x` from JS is fine.

**JS**
One IIFE in `<head>`. Section 0 runs immediately (frame-busting, applying the stored theme before first paint); everything else runs in `init()` on `DOMContentLoaded`: nav + theme toggle, scroll handler, fade-in, counters, retirement calculator, checklist gate, carousel, enquiry form, newsletter, lunch talk dialog, footer year.
- **Security rules (enforced by CSP `require-trusted-types-for 'script'`):** never use `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval` or string `setTimeout`. They throw. Build DOM with `createElement`/`textContent`/`replaceChildren`, or clone a `<template>` (see `enquiry-success-tpl`).
- **Lead capture:** every form goes through `submitLead(kind, fields)`. It is the single integration point: it currently logs JSON to the console after a 1.2s delay. To wire a real backend, replace its body with a `fetch()` POST and add the endpoint's origin to `connect-src` in the CSP. UTM parameters are captured (allow-listed) into `attribution` and sent with every lead.
- **Anti-spam (client-side only, the backend must re-validate):** each form has a `.hp-field` honeypot input (`name="website"`); `looksLikeBot()` also rejects submits within 2.5s of page load; `coolingDown(key)` limits one submit per 30s per form. Bots get a fake success.
- **Input hygiene:** `clean(value, max)` strips control characters and caps length; `NAME_RE` (Unicode letters) validates names; select/radio values are checked against the `ALLOWED` lists. Every input has a `maxlength`.
- **Retirement calculator:** range inputs `#calc-age/-retire/-savings/-monthly/-income`, each with an `<output id="{id}-out">`. `project()` works in today's dollars using the constants `GROWTH`, `INFLATION`, `RETIRED_GROWTH`, `LIFE_TO`; if you change them, update the assumptions text under the calculator and the FAQ answer (visible text and JSON-LD). The score moves the sun in the `#gauge` SVG. The headline result is free; the milestone table (`#calc-report`) unlocks after the name + email gate, which also unlocks the checklist.
- **Checklist:** the first 5 items are visible; the rest sit in `#checklist-locked` (`inert`, `aria-hidden`, blurred) until `unlockChecklist()` runs. The unlock flag `hw-checklist-unlocked` is kept in localStorage (no personal data is ever stored). "Print or save as PDF" adds `body.printing-checklist` so the print styles show only the sheet.
- **Lunch talk dialog (`#talk-dialog`):** a modal invite shown on every device after `TALK_DELAY_MS` (10s), once per session (`hw-talk-shown` in sessionStorage). It waits if the tab is hidden or a form field has focus, and stops appearing after `TALK_ENDS`. RSVPs go through `submitLead("lunch-talk-rsvp", …)`. For a new event, update the date/venue text in the dialog, `TALK_ENDS` and the `event` id.
- **Chat widget (`.chat-widget`):** a robot launcher that opens `#chat-panel`, a non-modal panel that types out `CHAT_LINES` and then shows `wa.me/6512345678` links with pre-filled text. The number appears in every `wa.me` href, so change them all together. The "hi" bubble shows once per session (`hw-chat-seen` in sessionStorage). The widget must stay after `.sticky-cta` in the DOM, because `.sticky-cta.show ~ .chat-widget` lifts it above the mobile bar. Colours come from `--wa`/`--on-wa` (defined in all three token blocks).
- **Fade-in:** add `.fade-in` (plus `.delay-1/2/3`). Use sparingly.
- **Counters:** `.stat-number[data-target][data-prefix][data-suffix]` with a sibling `.sr-only` final value; the animated span is `aria-hidden`.
- **Carousel:** `--per-view` on `.carousel` (1, or 3 at 1024px+) must match `getPerView()`. To add a testimonial, copy an `<li class="carousel-slide">` and update every slide's `aria-label="n of N"`.
- **Enquiry form:** the `validators` map (field name → error string, `""` = valid). Each field needs a `.field` wrapper and `<small id="{name}-error">`. `contactMethod` is special-cased in `getFieldWrapper`.

## SEO

- Title, meta description, canonical, Open Graph/Twitter tags and JSON-LD (`FinancialService`, `WebSite`, `FAQPage`) are in `<head>`. The canonical and `og:*` URLs point at `https://alfredang.github.io/financialplanning/`, so update them if the domain changes.
- The FAQ answers exist twice: the visible `<details>` and the `FAQPage` JSON-LD. Keep them word-for-word in sync.
- One `<h1>`; each section has an `<h2>`. `og-image.png` is copied from `docs/screenshot.png` by the Pages workflow.

## Security notes

GitHub Pages cannot set HTTP headers, so security headers are approximated in `<meta>` (CSP, referrer policy). CSP `frame-ancestors` doesn't work in a meta tag, so a JS frame-buster in section 0 hides the page if framed. For real headers (HSTS, `frame-ancestors`, `X-Content-Type-Options`, `Permissions-Policy`) put the site behind a host or CDN that supports them (Cloudflare, Netlify).

## Placeholder content

The firm, address, phone number, email (`hello@horizonwealth.example`), stats, testimonials and social links (`href="#"`) are all fictional placeholders.
