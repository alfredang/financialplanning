# Horizon Wealth Planning

A one-page marketing site for a fictional financial planning firm, built with plain HTML, CSS and JavaScript in a single `index.html`. There are no frameworks, no build step and no dependencies.

**Live site:** https://alfredang.github.io/financialplanning/

![Horizon Wealth Planning hero: red-accented headline "Financial planning that starts with your retirement number" beside an interactive retirement calculator with a sun rising over a horizon line](docs/screenshot.png)

## What's on the page

- **Retirement calculator (primary lead magnet)** in the hero. Five sliders give an instant readiness score, shown as a sun rising over the logo's horizon line. The headline result is free; a milestone-by-milestone report unlocks with first name and email.
- **Retirement checklist (second lead magnet)**, partially gated: the first 5 of 15 checks are visible, the rest unlock with an email and can be printed or saved as a PDF.
- **Lunch talk invitation**: a popup 10 seconds into the visit (once per session, on every device) invites visitors to a free one-hour retirement planning lunch talk at the Bukit Timah office, with a name and email RSVP. It stops appearing once the talk is over.
- A **sticky call-to-action bar** on mobile.
- Services, a three-step "how it works", a testimonials carousel, an FAQ, and an enquiry form that can be pre-filled from the calculator result.
- **Light and dark themes**: follows the OS setting, with a toggle in the header that remembers your choice.

## SEO

- Keyword-focused title, meta description, canonical URL, Open Graph and Twitter card tags
- JSON-LD structured data: `FinancialService` (address, phone, hours, services), `WebSite` and `FAQPage`
- One `<h1>`, an `<h2>` per section, and crawlable text content for services, process, checklist and FAQ
- No third-party images: the largest element above the fold is text, which keeps LCP fast

## Security hardening

The site is static, so the attack surface is the browser. These defences are layered:

- **Strict Content Security Policy** (in a `<meta>` tag): `default-src 'none'`, script and style allowed **only by SHA-256 hash**, no inline event handlers, `object-src`, `base-uri`, `frame-src` and `form-action` all locked down, `upgrade-insecure-requests`.
- **Trusted Types** (`require-trusted-types-for 'script'`): DOM-XSS sinks like `innerHTML` throw, and the code builds DOM with `textContent` and `<template>` cloning only.
- **No third-party scripts or images**; only Google Fonts is external. Referrer policy is `strict-origin-when-cross-origin`.
- **Clickjacking**: a frame-buster hides the page if it's loaded inside a frame (GitHub Pages can't send `frame-ancestors`).
- **Forms**: `maxlength` on every input, control-character stripping, Unicode-aware name validation, allow-listed select values, honeypot fields, a minimum fill time and a per-form cooldown against bots. The enquiry form uses `method="post"` so personal data never lands in a URL.
- **No personal data in browser storage**: only the theme choice and a "checklist unlocked" flag.
- **CI supply chain**: GitHub Actions are pinned to full commit SHAs, and checkout doesn't persist credentials.

Client-side checks are not a security boundary. When you add a backend, re-validate everything server-side and rate-limit it. For real HTTP headers (HSTS, `frame-ancestors`, `X-Content-Type-Options`, `Permissions-Policy`), put the site behind a host or CDN that can set them.

## Tech stack

- HTML5, CSS3 (custom properties, mobile-first media queries, light/dark tokens) and vanilla JavaScript (one IIFE, no dependencies)
- Google Fonts: IBM Plex Sans Condensed (headings) and IBM Plex Sans (body)
- GitHub Actions and GitHub Pages for hosting

## Project structure

```
index.html                    # the whole site: markup, <style> and <script>
docs/screenshot.png           # README screenshot, also deployed as og-image.png
.github/workflows/pages.yml   # GitHub Pages deploy
.claude/skills/               # Claude Code skills used for the redesign
.mcp.json                     # Playwright MCP server config
CLAUDE.md                     # notes for Claude Code (including the CSP hash command)
```

## Placeholder content

The following are **fictional** and need replacing before real use:

- The firm name, address, phone number (`+65 6234 5678`) and email (`hello@horizonwealth.example`)
- The testimonials, the stats (15+ years, 1,200+ clients, S$500M) and the social links (all `href="#"`)

**There is no backend.** Every form goes through one function, `submitLead()`, which logs the lead as JSON (with any UTM campaign parameters) to the browser console. To actually capture leads, replace its body with a `fetch()` POST to your CRM or email tool and add that origin to `connect-src` in the CSP. The "we've emailed you a copy" messages assume you set up that delivery email.

## Running locally

Open `index.html` directly, or serve the folder:

```powershell
start index.html
# or
python -m http.server 8000   # then visit http://localhost:8000
```

You need internet access for Google Fonts. Use a local server rather than `file://` if you want the CSP to behave as it does in production.

## Deployment

Every push to `main` copies `index.html` (and `docs/screenshot.png` as `og-image.png`) into `_site/` and deploys only that to GitHub Pages via [.github/workflows/pages.yml](.github/workflows/pages.yml). You can also run the workflow manually from the Actions tab.

## Conventions

- **Everything stays in `index.html`**: CSS in one `<style>` tag, JS in one `<script>` tag. Don't add frameworks, build tools or extra source files.
- **Section order is the same across HTML, CSS and JS**, and each block is marked with a `===== N. NAME =====` banner comment.
- **After editing the `<style>` or `<script>` block, regenerate the CSP hashes** with the command in [CLAUDE.md](CLAUDE.md), or the browser will block them.
- **Never use `innerHTML`** or other HTML-string sinks; Trusted Types will throw. Use `textContent` or a `<template>`.
- **Use the design tokens** on `:root` (`--red`, `--ink`, `--bg`, `--surface`, `--space-1…8`, `--nav-h`) rather than hardcoded values. Dark values are defined twice (OS preference and toggle) and must match.
- **Styles are mobile-first.** The only breakpoints are `min-width: 768px` and `min-width: 1024px`, grouped near the end of the stylesheet.
- **Carousel:** `--per-view` in CSS and `getPerView()` in JS must use the same breakpoint. If you add a testimonial, update every slide's `aria-label="n of N"`.
- **Form fields:** each one needs a `.field` wrapper, a `<small id="{name}-error">`, a `maxlength` and an entry in the `validators` map. Leads go through `submitLead()`.
- **Reduced motion:** `prefers-reduced-motion` turns off the animations in CSS, and the JS skips the counters and autoplay.
- Add `.fade-in` (plus `.delay-1/2/3` if you want a delay) to any element to reveal it on scroll.

## Testing

There's no test suite. To syntax-check the JavaScript:

```bash
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > "$TEMP/hw.js" && node --check "$TEMP/hw.js"
```

Then check it by hand in a browser at mobile, tablet and desktop widths.
