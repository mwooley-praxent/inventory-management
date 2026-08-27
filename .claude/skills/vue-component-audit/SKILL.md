---
name: vue-component-audit
description: >-
  Analyze Vue 3 component structure under client/src for performance issues
  and code-reuse opportunities, then produce a single prioritized, file:line-
  grounded report. Use when the user asks to "audit Vue components", "find
  performance issues in the frontend", "look for duplicate code in
  components", "suggest Vue optimizations", "review component structure",
  "find reusable components/composables", "reduce re-renders", "clean up
  component duplication", or similar structural/performance review requests
  about client/. This is an ANALYSIS skill — it reads and reports; it never
  edits .vue files itself. If the user wants fixes applied, it hands the
  exact spec to the vue-expert subagent (per this repo's mandatory rule that
  vue-expert owns all .vue changes) and only after the user confirms.
---

# Vue Component Audit — Performance & Reuse

Reads `client/src` (or a user-specified subset) and produces one prioritized
report of concrete, file:line-grounded findings across two dimensions:
**performance** (unnecessary re-renders, wasted computation, wasted network
calls) and **code reuse** (duplicated markup/logic that should be a shared
component or composable). This skill plans and reports; it does not write
code.

## Scope & Non-Goals

**In scope:** reading and analyzing `client/src/views/*.vue`,
`client/src/components/*.vue`, `client/src/composables/*.js`,
`client/src/api.js`, and `client/src/App.vue`. Producing a prioritized
findings report with a concrete fix recommendation per item.

**Out of scope:** backend/API changes, new features, visual/design changes
(use `frontend-design` for that), and — unless the user explicitly asks for
fixes to be applied — any file edits at all. This skill never calls Write or
Edit on a `.vue` file; see "Delegate, don't implement" below.

**Default target:** if the user doesn't name specific files, analyze all of
`client/src/views/*.vue` and `client/src/components/*.vue`. If they name a
component or view, scope the audit to that file plus anything it visibly
shares patterns with (e.g. auditing one modal should glance at its sibling
modals to check whether a finding is isolated or systemic).

## Step 1 — Establish the baseline

Before flagging anything, read (don't assume):

1. This repo's own conventions in `CLAUDE.md`: "Raw data in refs
   (`allOrders`, `inventoryItems`), derived data in computed properties" and
   the 4-filter system (Time Period, Warehouse, Category, Order Status)
   flowing `Vue filters → api.js → FastAPI → computed properties`. Findings
   should be judged against these stated conventions, not generic Vue
   best-practice in the abstract.
2. `client/src/composables/useFilters.js` and `useI18n.js` — the two
   composables most views are expected to reuse. A view reimplementing logic
   these already provide is a reuse finding, not a fresh pattern to praise.
3. The list of views (`Dashboard`, `Inventory`, `Orders`, `Demand`,
   `Backlog`, `Reports`, `Spending`) and modals (`BacklogDetailModal`,
   `CostDetailModal`, `InventoryDetailModal`, `ProductDetailModal`,
   `ProfileDetailsModal`, `TasksModal`) — these six modals in particular tend
   to duplicate structure (overlay/close-button/header markup, translation
   helper calls) and are a natural first place to check for reuse wins.

## Step 2 — Performance checklist

For each file in scope, check for and cite file:line evidence of:

- **Reactive/computed misuse**: a plain function called directly in a
  `<template>` (e.g. `getOrdersByStatus('shipped')`) that re-executes on
  every render instead of being a `computed`. Cross-check against this
  repo's own stated rule that derived data belongs in computed properties.
- **Unindexed lookups in loops**: `.find()`/`.filter()` called inside a
  `computed` or loop over another array (O(n·m)) where a `Map` keyed by
  id/sku built once would be O(n).
- **Watcher fan-out**: a `watch([...several filter refs], loadData)` where
  `loadData` re-fetches data that doesn't actually depend on all of those
  filters (e.g. it fetches an endpoint with no query params, then filters
  client-side in a separate computed) — this means every filter change
  triggers a needless network round-trip.
- **Missing debounce / stale-response protection**: filter-triggered watchers
  with no `AbortController` or sequence guard, so a slow response for an
  older filter state can overwrite a newer one on screen.
- **`:key="index"`** in any `v-for` — flag every occurrence; it silently
  breaks Vue's reuse/patch logic on reorder and is a common source of subtle
  bugs, not just a lint nit.
- **Blocking native dialogs**: `window.alert()`/`window.confirm()` used for
  anything beyond trivial debug output — these block the main thread and
  can't be styled, localized, or tested.
- **Dead or no-op watchers/refs**: a `watch()` whose callback does nothing
  meaningful, or a ref that's set once and never read reactively.
- **Large static data in reactive refs**: big lookup tables or config arrays
  wrapped in `ref()`/`reactive()` when they never change — `markRaw()` or a
  plain module-level constant avoids needless reactivity tracking.
- **Route-level code splitting**: check `client/src/main.js` for
  `component: Component` (static import) vs `component: () => import(...)`
  (lazy). Only worth flagging above low severity if the view list is large
  or a view pulls in a heavy dependency.
- **Redundant unfiltered fetches**: the same data endpoint fetched
  separately by multiple views/components when it could be loaded once and
  shared (via a composable or a store) — check for repeated
  `api.get*()` calls with identical/no params across views.

## Step 3 — Code-reuse checklist

- **Sibling modal duplication**: diff the six modal components structurally
  (overlay markup, close button, header, translation-helper calls like
  `translateProductName`/`translateCategory`/`translateWarehouse`). If 3+
  share the same skeleton, recommend a shared wrapper (e.g. a `BaseModal.vue`
  using a default slot, or a `useModal()` composable for open/close/escape
  behavior) rather than continued copy-paste.
- **Repeated view-load/filter pattern**: `Dashboard.vue`, `Backlog.vue`, and
  `Demand.vue` (and similar) each hand-roll their own
  `loadData()` + `watch([...filters], loadData)` + client-side filtering
  `computed`. If the shape is truly repeated 3+ times, recommend extracting
  a `useFilteredResource(fetchFn, filterRefs)`-style composable rather than
  leaving each view to reinvent it — but only if the extraction would
  actually simplify each call site; don't force an abstraction onto views
  whose filtering logic meaningfully differs.
- **Duplicated formatting/translation helpers**: grep for near-identical
  small functions repeated across files (currency formatting beyond
  `client/src/utils/currency.js`, date formatting, status-badge class
  lookups, translation helper functions defined locally in a component
  instead of imported from `useI18n.js` or a shared util). Recommend
  consolidating into the existing util/composable rather than a new one.
- **Copy-pasted table/list markup**: near-identical `<table>` or list
  structures (headers, row rendering, empty-state handling) repeated across
  `Inventory.vue`, `Orders.vue`, `Demand.vue`, etc. Recommend a shared
  presentational component (e.g. `DataTable.vue` with slots) only if the
  columns/behavior are similar enough that the slot API stays simple —
  call out if forcing a shared component would need more slot plumbing than
  the duplication it removes.
- **Props/emits drift across similar components**: if two components serve
  the same purpose (e.g. two detail modals) but have diverged prop/emit
  names for the same concept, flag the inconsistency even if you don't
  recommend merging them — it's a maintenance hazard on its own.

## Step 4 — Prioritize and report

Rate each finding **P0–P3** using impact × frequency, not novelty:

- **P0**: causes visibly wrong behavior (stale data shown, broken
  interaction) — rare for pure perf/reuse findings, but a debounce/stale-
  response gap that regularly shows wrong data qualifies.
- **P1**: real, felt cost today — a redundant network call on every filter
  change, an O(n·m) scan on a hot render path, 3+ duplicated non-trivial
  blocks.
- **P2**: correct but wasteful, or a moderate duplication (2 near-identical
  blocks) — worth fixing opportunistically.
- **P3**: micro-optimizations or stylistic reuse cleanups with no measurable
  effect at current data volume — still worth listing (don't silently drop
  them), but label them as low-urgency.

Report format: one table per dimension (Performance, Reuse) with columns
Priority | Finding | File:Line | Recommendation. Every row must cite an
actual file:line — no finding without evidence. If a performance finding and
a reuse finding point at the same code (e.g. a duplicated `loadData` pattern
that's also the source of the redundant-fetch performance issue), say so
explicitly rather than listing it twice as if unrelated.

## Step 5 — Delegate, don't implement

This skill's output is the report from Step 4. If the user then asks to
apply some or all of the fixes, do not edit `.vue` files directly — hand the
selected findings, with their exact recommendations, to the `vue-expert`
subagent as a concrete spec (files to touch, what to extract, what to keep
inline). Confirm with the user which findings to act on before delegating if
they haven't already specified ("fix everything" is enough to proceed with
all of them).

## Step 6 — Verify, if fixes were applied

If `vue-expert` made changes, verify with the Playwright MCP tools rather
than assuming: load each affected route at `http://localhost:3000`, exercise
the specific interaction the fix touched (e.g. change a filter and confirm
no extra network request fires, open/close a refactored modal, resize to a
narrow viewport if a shared component's layout changed), and check
`browser_console_messages` for new warnings.

## Guardrails

- Never edit a `.vue` file directly — this skill reports; `vue-expert`
  implements, and only on request.
- Never list a finding without a concrete file:line citation — no generic
  "consider using computed properties" without pointing at the exact
  offending call site.
- Don't force a shared abstraction (base component, composable) onto code
  that only superficially looks similar — note the duplication but recommend
  against merging if the divergence is meaningful.
- Don't silently drop low-severity findings to shorten the report — list
  them as P3 instead.
- Judge findings against this repo's own stated conventions in `CLAUDE.md`
  (refs vs. computed, the 4-filter data flow) before generic Vue best
  practice.
