---
name: saas-redesign
description: Redesign this Vue 3 app's UI into a dark-first SaaS dashboard with a vertical left sidebar instead of the top nav, consistent spacing, and responsive behavior. Use when asked to redesign, modernize, or restyle the app shell or views into a SaaS look.
---

# SaaS Redesign (dark-first dashboard)

Turn the top-nav layout into a dark-first SaaS dashboard: dark sidebar, dark content surfaces, high-contrast data. Look and layout change; behavior does not.

## Layout
- App shell in `client/src/App.vue`: fixed left sidebar (240px) + scrolling main area. Remove the `top-nav` header.
- Sidebar, top to bottom: logo + subtitle, nav links (one per route in `client/src/main.js`), then `LanguageSwitcher` and `ProfileMenu` pinned at the bottom.
- `FilterBar` sits at the top of the main area as a sticky toolbar, not inside the sidebar.
- Active link: accent left border + lighter background, driven by `$route.path` as today.

## Theme (dark-first)
Define tokens once on `:root` in `App.vue` and use them everywhere; no raw hex in views.
- Surfaces: `--bg #0b1120`, `--sidebar #0f172a`, `--card #111827`, `--card-hover #1f2937`, `--border #1f2937`
- Text: `--text #e2e8f0`, `--text-muted #94a3b8`, `--text-dim #64748b`
- Accent: `--accent #3b82f6`; status keeps green/blue/yellow/red, brightened for dark (`#22c55e`, `#60a5fa`, `#facc15`, `#f87171`)
- Spacing scale `--s1..--s6` = 4/8/12/16/24/32px; radius `--r 10px`; shadow minimal, borders do the separation.
- Charts are custom SVG: switch axis, grid, and label colors to the tokens so they stay legible on dark.
- Check text contrast is at least 4.5:1 on every surface.

## Icons
- Inline SVG only, no new dependencies. Add `client/src/components/Icon.vue` taking a `name` prop with Lucide-style 24x24 stroke paths (`stroke="currentColor"`, `stroke-width="2"`, `fill="none"`).
- One icon per nav item: dashboard, inventory (box), orders (clipboard), spending (wallet), demand (trending-up), reports (file-chart).
- Icons are decorative next to labels: `aria-hidden="true"`.
- No emojis (project rule).

## Responsive
- ≥1024px: full sidebar with labels.
- 768–1023px: icons-only rail (72px); labels become `title` tooltips.
- <768px: sidebar hidden behind a hamburger button in a top bar; opens as an overlay drawer and closes on route change, Escape, or backdrop click.
- Card and KPI grids use `repeat(auto-fit, minmax(…, 1fr))`; wide tables scroll inside their card, never the page.
- No horizontal page scroll at 375px.

## Don't break
- Keep every `t('…')` label, router link, filter, and modal working.
- Delegate `.vue` creation and large edits to the `vue-expert` subagent (CLAUDE.md rule).

## Verify
1. App loads at http://localhost:3000 with no console errors.
2. Click every route; data and filters still update.
3. Check widths 1280, 900, and 375 (Playwright MCP, `browser_resize`) and screenshot each.
