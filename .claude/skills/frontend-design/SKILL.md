---
name: frontend-design
description: >-
  Redesign this app's Vue 3 UI into a modern SaaS-style interface: convert a
  top navigation bar into a vertical left sidebar, apply clean modern card
  layouts, consistent spacing, and professional visual polish. Use when the
  user asks to "redesign the UI", "modernize the frontend", "give this a SaaS
  look", "add a sidebar", "replace the top nav with a sidebar", "make this
  look more professional/polished", "clean up the layout", "improve the
  design system", or similar visual/layout requests about client/. This is a
  VISUAL/LAYOUT design skill only — it audits the app, assembles design
  tokens, a sidebar spec, and a layout plan, then delegates all actual .vue
  file creation/editing to the vue-expert subagent (per this repo's mandatory
  rule that vue-expert owns all .vue changes). Do NOT use this skill for new
  features, API integration, business logic, or data changes — use
  vue-expert directly for those; this skill does not write code itself.
---

# Frontend Design — SaaS Sidebar Redesign

Turns a Vue 3 app with a top nav bar into a modern SaaS-style shell: vertical
left sidebar navigation, real design tokens, polished cards, and basic
responsive behavior. This skill plans; `vue-expert` implements.

## Scope & Non-Goals

**In scope:** layout structure (top nav → sidebar), design tokens
(color/spacing/radius/shadow/font), card and shared-component polish,
sidebar responsiveness (mobile drawer).

**Out of scope:** new features, new business logic, API/data changes. The
only routing change this skill ever makes is *nav-surfacing* an already-
existing, unregistered view (e.g. a `views/*.vue` file with no route) if the
user asks for it — never scaffolding a brand-new view or endpoint.

**Never edit `.vue` files directly.** This repo's `CLAUDE.md` mandates that
any `.vue` creation/modification goes through the `vue-expert` subagent. This
skill's job ends at producing a concrete spec; implementation is always
delegated (see "Delegate, don't implement" below).

## Step 1 — Audit the app

Before proposing anything, gather facts. Don't assume — grep and read:

1. **Root layout component**: usually `client/src/App.vue` (or
   `src/layouts/*.vue`). This is almost always where the top nav and the
   single global `<style>` block live.
2. **Router config**: `client/src/main.js` or `src/router/index.js`. List
   every registered route.
3. **Orphaned views**: diff `client/src/views/*.vue` against the registered
   routes. Flag any view file with no route/nav entry — ask the user whether
   to wire it in as part of this pass, don't silently add or silently ignore it.
4. **Existing design tokens**: grep for `:root` and `var(--` across
   `client/src`. If tokens already exist, extend them — never introduce a
   second, competing token set.
5. **Global stylesheet location**: is there one unscoped `<style>` block
   (commonly in the root layout component) or a separate `.css` file imported
   in `main.js`? Match whatever already exists; don't invent a second global
   stylesheet.
6. **Hardcoded palette/spacing/radius**: grep the global stylesheet for hex
   colors, `rem`/`px` spacing, and `border-radius` values. Tally what repeats —
   these become your design tokens. Never invent an unrelated brand palette;
   the existing colors are the brand.
7. **Icon strategy**: grep for `<svg` usage and check `package.json` for an
   icon package (`lucide`, `heroicons`, `@fortawesome/*`, etc.) or a UI
   framework (Tailwind, Vuetify, PrimeVue, etc.).
8. **Existing responsive strategy**: grep for `@media` in the global
   stylesheet and any nav/filter components. If none exist, design mobile
   behavior from scratch rather than "fixing" something that isn't there.
9. **Sticky/fixed offsets tied to the old header height**: grep for
   `position: sticky` / `position: fixed` with a `top:`/`left:` value near
   components that render directly below or beside the current top nav (e.g.
   a filter bar). These values are almost always hardcoded to the old header's
   height and will break once the header disappears — they must be
   recalculated for the new shell, not left as-is.

**Stop and ask the user before proceeding if:** no root layout component can
be found; a UI framework is already present (ask whether to work within it or
around it); there's no `vue-expert`-equivalent delegation target in this
repo's agent config (ask how the user wants `.vue` edits handled).

## Step 2 — Design tokens

Add a `:root { ... }` block at the top of the global stylesheet identified in
Step 1.5 (don't create a second CSS file unless one already exists as the
established pattern). Establish these categories, using values pulled from
the Step 1.6 audit:

- **Color**: surface/canvas backgrounds, border tiers (subtle/default/strong),
  text tiers (primary/secondary/tertiary/muted), brand (base/hover/focus/
  surface-tint), status colors (success/warning/danger/info, each with a
  base/bg/text triplet for badges).
- **Spacing scale**: a 4px-based scale (`--space-1` through `--space-8` or
  similar) built from whatever rem values already repeat in the app.
- **Radius scale**: small/medium/large, based on existing radius values —
  it's fine to round up slightly (e.g. 10px → 12px) for a more polished feel,
  but don't invent a wildly different scale.
- **Shadow scale**: sm/md/lg subtle elevation shadows, since "SaaS polish"
  reads largely through soft shadows and consistent radii.
- **Font scale**: reuse existing font-size steps if present.
- **Sidebar-specific tokens**: width (expanded/collapsed), background, text,
  active-text, accent, hover-bg — decided in Step 3.

## Step 3 — Sidebar component spec

Design a new sidebar component (e.g. `AppSidebar.vue`) with:

- **Brand/logo header** at the top (reuse whatever the old top nav's
  logo/company-name markup was).
- **Nav list**: a **hardcoded config array** of `{ path, label, icon }`
  mirroring the routes found in Step 1.2 (plus any orphaned view the user
  chose to wire in) — not derived live from the router object, since route
  definitions typically carry no `meta.label`/`meta.icon`. Icons: continue
  whatever icon strategy Step 1.7 found (inline hand-rolled SVG is the
  common case for apps with no icon package — do not add one silently).
- **Active-state styling**: adapt the old nav's active pattern (commonly a
  color + tinted background) to a vertical list — a left accent bar reads
  better than a bottom underline in a sidebar.
- **Footer area**: pinned at the bottom, holding whatever secondary nav
  elements lived in the old top nav's right side (profile menu, language
  switcher, etc.) plus a collapse/expand toggle. Persist collapse state in
  `localStorage`.
- **Collapse behavior**: expanded shows icon + label; collapsed shows icon
  only, at a narrower fixed width.
- **Choose a theme** (light or dark sidebar) — ask the user if not already
  specified; it's a real visual decision, not a default to silently pick.

## Step 4 — Layout restructuring

- Change the root layout's markup from a vertical stack (header → content) to
  a horizontal shell: `.app-shell { display: flex; min-height: 100vh; }`
  containing the sidebar and a `.app-main { flex: 1; display: flex;
  flex-direction: column; min-width: 0; }` content column.
- Delete the old top-nav markup and its CSS rules entirely once the sidebar
  replaces it — don't leave dead styles behind.
- Sidebar: `position: sticky; top: 0; height: 100vh; align-self: flex-start;
  flex-shrink: 0;` on desktop, so the whole page scrolls together (avoid a
  nested scroll container unless the app already uses one elsewhere).
- **Recalculate every sticky/fixed offset flagged in Step 1.9** against the
  new shell (most commonly this means changing a `top: <old-header-height>px`
  to `top: 0`, since the old header is gone).
- Move any secondary top-nav elements (profile menu, language switcher) into
  the sidebar footer per Step 3.

## Step 5 — Responsive behavior

If Step 1.8 found no existing `@media` strategy, add one from scratch:

- A single breakpoint (768px is a reasonable default) below which the
  sidebar becomes `position: fixed; transform: translateX(-100%)`, toggled by
  an `.is-open` class.
- A backdrop overlay (`rgba(15, 23, 42, 0.4)` or similar) shown behind the
  open drawer.
- A hamburger toggle button, rendered by the sidebar component itself and
  hidden above the breakpoint.
- All open/closed and collapsed/expanded state lives inside the sidebar
  component — the root layout component passes no layout state as props.

## Step 6 — Card & shared-component polish

Restyle shared global classes (e.g. `.card`, `.stat-card`, `.page-header`,
`.badge`) once, in the global stylesheet, using the new tokens — this
polishes every view without per-view edits. Only recommend a per-view
touch-up if Step 7's visual verification reveals a specific broken or
awkward layout; don't proactively rewrite view-level styles.

## Step 7 — Delegate, don't implement

Once the audit (Step 1) and the concrete spec (Steps 2–6) are assembled —
exact token values, exact file list to create/modify, exact markup/CSS
diffs — invoke the Task tool targeting the `vue-expert` subagent with that
full spec. **This skill does not call Write or Edit on any `.vue` file
itself.** Hand vue-expert the complete plan; it performs the actual file
changes.

## Step 8 — Verify visually

There is typically no frontend test suite to lean on — verification is
visual, via the Playwright MCP tools:

1. Confirm the dev server(s) are running; start them if not.
2. **Before** vue-expert's changes: screenshot every registered route (plus
   any newly-wired route) at a desktop width (e.g. 1440×900) and a mobile
   width (e.g. 375×812).
3. After vue-expert's changes: repeat the same route × viewport matrix.
4. Check `browser_console_messages` on each page for new warnings/errors
   (missing `:key`, prop mutation, missing i18n keys, 404s).
5. Exercise interactions: sidebar collapse/expand (and reload to confirm
   `localStorage` persistence), mobile drawer open/close via the hamburger
   and backdrop, and clicking a nav item on mobile (should navigate and close
   the drawer).
6. Spot-check any component whose sticky offset was recalculated in Step 4 —
   scroll a data-heavy page and confirm no gap or overlap.

## Guardrails

- Never edit a `.vue` file directly — always delegate to `vue-expert`.
- Never silently add a UI framework or icon package — flag it and ask first.
- Never invent a new brand palette — tokens extend the audited colors.
- Always recalculate pixel offsets that assumed the old header/nav height.
- Always verify visually with Playwright before declaring the redesign done.
- Never scaffold new features/routes beyond nav-surfacing an existing,
  already-built view the user explicitly asked to wire in.
