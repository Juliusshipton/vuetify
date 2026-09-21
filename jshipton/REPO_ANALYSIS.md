# Repository Analysis: Vuetify monorepo, oriented around issue #23183

Orientation notes for adding a `first-day-of-week` prop to `v-heatmap`
(labs component) so its calendar layout can start weeks on any weekday, matching
what `v-date-picker` already does.

| | |
|---|---|
| Issue | [vuetifyjs/vuetify#23183](https://github.com/vuetifyjs/vuetify/issues/23183) — labels `C: VHeatmap`, `T: feature`, milestone v4.x.x |
| Component | `packages/vuetify/src/labs/VHeatmap` |
| Base branch | `dev` (new prop = feature) |
| Ships in | 4.3.0 (repo is at 4.2.1) |
| Commit type | `feat(VHeatmap): …` |

## 1. The ask

`v-heatmap` can already look like a GitHub-style contribution calendar: one
column per week, one row per weekday, months split into groups. It does this
without knowing anything about dates. The user passes `rows="[0,1,2,3,4,5,6]"`
and an `item-row` accessor that returns `getDay()`. The component starts a new
column every time the row index wraps back, so week boundaries are implicitly
Sunday-first.

The requester wants a `first-day-of-week` prop (0 = Sunday … 6 = Saturday) that:

1. orders the weekday rows starting from that day,
2. makes week columns break on that day, and
3. keeps the month-overlap trick working, so `group-gap` remains spacing
   between months only.

```
Today (Sunday-first)              Wanted: first-day-of-week="1"

Sun  . # # # # #                  Mon  . # # # # +
Mon  . # # # # #                  Tue  . # # # # +
Tue  . # # # # +                  Wed  # # # # # +
Wed  # # # # # +                  Thu  # # # # # +
Thu  # # # # # +                  Fri  # # # # # +
Fri  # # # # # +                  Sat  # # # # # +
Sat  # # # # # +                  Sun  # # # # # +

.  blank    #  this month    +  next month sharing the last column
```

A month starting Wednesday leaves the rows above it blank in its first column,
and the following month shares that last column. Rotating the rows so Monday is
on top preserves this, with a new column starting when the weekday index falls
back to Monday.

## 2. Repo at a glance

A pnpm workspace with three packages. Almost everything touched for this
feature is under `packages/`; the rest is tooling.

| Path | What it is |
|---|---|
| `AGENTS.md` (included by `CLAUDE.md`) | Maintainer expectations in one page: branches, commands, principles, and the review rubric applied to PRs. Read once. |
| `.claude/rules/components.md` | Component anatomy, how to add a prop (docs + api entries), passing props as getters. **Read before coding.** |
| `.claude/rules/composables.md`, `code-readability.md` | Composable API stability, naming, no narrating comments. |
| `.claude/skills/playground/SKILL.md` | How the Playground is meant to be shaped and used as the PR demo. |
| `.claude/skills/tests/SKILL.md` | Unit vs browser specs, helpers, flakiness rules. |
| `packages/vuetify/` | **The `vuetify` npm package.** Source, tests, the dev playground. |
| `packages/api-generator/` | **Builds the API tables the docs render.** One prop description may be added here. |
| `packages/docs/` | **vuetifyjs.com.** Markdown pages, live examples, and `new-in.json` which stamps the version badge on new props. |
| `scripts/` | Repo-level node scripts. `dev.js` forwards `pnpm dev` to a package's own dev script (default target `vuetify`; `pnpm dev docs` targets the docs site). Also build, commit-message lint, release helpers. |
| `templates/` | Scaffolds: `component.tsx`, `unit-test.ts`, `browser-test.tsx`, `page.md`. Reference only; this feature extends an existing component. |
| `patches/` | pnpm patches applied to third-party dependencies at install. Not relevant here. |
| `pnpm-workspace.yaml`, `lerna.json` | Workspace membership, dependency catalog, release versioning. |

## 3. Inside `packages/vuetify`

No build step during development. Vite serves `dev/` with `@/` aliased to
`src/`, so edits to components, composables and Sass hot-reload into the
Playground.

| Path | Role |
|---|---|
| `dev/Playground.vue` | **Gitignored scratch file.** The reproduction, and later the demo pasted into the PR. All components auto-register, labs included. |
| `dev/vuetify.js`, `dev/vuetify/{date,locale,defaults,icons}.js` | `createVuetify` options for the playground. Change `locale.js` to test a Monday-first locale default. |
| `src/components/` | 104 stable components, one folder each. `VDatePicker/` and `VCalendar/` are the reference points for week handling. |
| `src/labs/` | Pre-release components. Same anatomy as stable ones, API may still change. Exported from `labs/components.ts`. |
| `src/labs/VHeatmap/` | **The component.** Detailed below. |
| `src/composables/` | Shared reactive logic. `calendar.ts` declares the `firstDayOfWeek` prop the date picker uses; `date/` holds the adapter that knows locale week rules. |
| `src/util/` | Helpers such as `propsFactory`, `genericComponent`, `getPropertyFromItem`, type guards. Use these instead of raw `typeof` checks. |
| `src/locale/*.ts` | UI strings per language. The heatmap already has `heatmap.less` / `heatmap.more`. This feature adds no new strings. |
| `src/styles/` | Global Sass, settings and the `tools.layer` mixin every component stylesheet wraps itself in. |
| `vitest.config.ts` | Two projects: `unit` (jsdom) for `*.spec.ts` and `browser` (Playwright Chromium) for `*.spec.browser.tsx`. |

## 4. VHeatmap anatomy

The folder follows the house pattern: a thin `.tsx` for markup, local
composables for the heavy lifting, colocated tests.

| File | Responsibility |
|---|---|
| `pivot.ts` | **Core.** `usePivot`. Generic, date-agnostic: turns flat items into rows, column groups and columns. Contains the row-wrap rule that makes calendar columns. |
| `heatmap.ts` | **Core.** `useHeatmap`. Wraps `usePivot`, colours each cell from `thresholds`, computes pixel geometry and the month-overlap offset. |
| `VHeatmap.tsx` | Declares props via `makeVHeatmapProps`, calls `useHeatmap(props)`, renders labels plus an SVG grid. |
| `VHeatmapCell.tsx` | One SVG `<g>` per cell with an optional `foreignObject` for the `cell` slot. |
| `VHeatmapLegend.tsx`, `VHeatmapLegendCell.tsx` | Clickable bucket legend or gradient bar. Untouched by this feature. |
| `colorScale.ts` | Bucket vs linear threshold types and interpolation helpers. |
| `VHeatmap.scss`, `_variables.scss` | Styles inside `tools.layer('components')`, tunables as `!default` Sass variables. No style changes expected. |
| `index.ts` | Exports the four components only. Never export composables or types from here; it breaks api-generator. |
| `__tests__/heatmap.spec.ts` | Unit tests that mount `useHeatmap` directly. The *inferred columns* block is where new cases go. |

### Data flow on every render

```
Props (VHeatmap.tsx)
  items, rows, columns, itemRow, itemColumn, itemValue, groupBy,
  thresholds, sizes, gaps, NEW: firstDayOfWeek
        │
        ▼
usePivot (pivot.ts)                         <── week start lands here
  rows            = explicit prop or insertion order      (line 55)
  hasExplicitColumns = is itemColumn set?                 (line 68)
  if not, buildInferredColumns starts a new column
    when rowIndex <= lastRowIndex                         (line 166)
  returns groups[].columns[].cells[]
        │
        ▼
useHeatmap (heatmap.ts)
  colour per cell (bucket index or mix %)
  group x, width, labelOffset
  hasOverlap when a group's first column has an empty first row   (line 183)
  total SVG width / height
        │
        ▼
Render (VHeatmap.tsx → VHeatmapCell)
  group labels, column headers, row headers
  <svg> with one VHeatmapCell per non-null cell
  slots: cell, row-header, group-header, legend
```

Two facts matter most:

1. Row **order** is decided once, in the `rows` computed at the top of
   `pivot.ts`, and every downstream index refers to that order.
2. The wrap rule and the overlap rule both compare against index 0 of that
   order. Rotate the rows so the chosen weekday sits at index 0 and both rules
   produce the requested behaviour without further changes.

**Why the overlap still works.** In `heatmap.ts` a group overlaps its
predecessor when `group.columns[0].cells[0] == null`, meaning the month did not
start on the first display row. After rotation that reads "the month did not
start on Monday", which is exactly the condition for sharing a week column.
`group-gap` is then applied only between months.

## 5. How the date picker does it

The issue asks for parity with `v-date-picker`. Its week-start logic lives in
two shared places, and neither is heatmap-specific.

- **Prop declaration.** `makeCalendarProps` in `src/composables/calendar.ts`
  (line 77) declares `firstDayOfWeek: { type: [Number, String], default: undefined }`.
  `VDatePickerMonth` spreads those props in. Same shape for the new prop.
- **Locale-aware default.** When the prop is undefined the date adapter falls
  back to the locale's week info. In `src/composables/date/adapters/vuetify.ts`,
  `startOfWeek`, `getWeekdays` and `getWeekArray` all do
  `firstDayOfWeek ?? weekInfo(locale)?.firstDay ?? 0`. `startOfWeek` also warns
  on values outside 0 to 6.
- **Reading the effective value.** `useCalendar` resolves the actual start day
  with `adapter.toJsDate(adapter.startOfWeek(adapter.date(), props.firstDayOfWeek)).getDay()`
  (line 131). That line is reusable verbatim if the heatmap should honour locale
  defaults too.
- **API description already exists.**
  `packages/api-generator/src/locale/en/generic.json` has a `firstDayOfWeek`
  entry that any component with a prop of that name inherits. A heatmap-specific
  description is only needed if the wording should differ.

Note what the heatmap does *not* have: a date adapter dependency. Items carry
whatever the user's `item-row` accessor returns. Deciding whether to introduce
`useDate()` into `useHeatmap` is the main design call (section 7).

## 6. Files to touch

Ordered roughly as they will be met. The rubric in `AGENTS.md` treats a missing
`new-in.json` entry or api description as a review finding.

| File | Change | Notes |
|---|---|---|
| `packages/vuetify/dev/Playground.vue` | Reproduce the calendar view with and without the new prop. | Dark theme, stacked cases labelled with a `<code>` line, deterministic data. Starter in section 8. |
| `src/labs/VHeatmap/VHeatmap.tsx` | Add `firstDayOfWeek: [Number, String]` to `makeVHeatmapProps` (line 34). | The whole `props` object is already passed to `useHeatmap(props)`, so no further wiring. |
| `src/labs/VHeatmap/heatmap.ts` | Extend `HeatmapProps`; hand `usePivot` a rotated `rows` getter when a week start is set. | Keeps `usePivot` generic (headed for `@vuetify/v0`). Pass a getter, never `props.firstDayOfWeek` unwrapped. |
| `src/labs/VHeatmap/pivot.ts` | Probably no change. If rotating here, add an option rather than a date concept. | The wrap rule at line 166 and `rows` at line 55 define behaviour. |
| `src/labs/VHeatmap/__tests__/heatmap.spec.ts` | New cases in the *inferred columns* block (line 149). | Rotation of `rows`; wrap point moves to the new first day; a month starting on the first day does not overlap; string `"1"` coerces; explicit-column grids unaffected. |
| `packages/api-generator/src/locale/en/VHeatmap.json` | Optional prop description. | `generic.json` already describes `firstDayOfWeek`. Add a local one only if the heatmap meaning (rotating rows) needs saying. |
| `packages/docs/src/data/new-in.json` | Add `"VHeatmap": { "props": { "firstDayOfWeek": "4.3.0" } }`. | No VHeatmap block exists yet. Keep alphabetical order. |
| `packages/docs/src/pages/en/components/heatmaps.md` | Mention the prop under Guide, next to the Calendar view section. | Docs pages embed examples with `<ExamplesExample file="v-heatmap/…" />`. |
| `packages/docs/src/examples/v-heatmap/misc-calendar.vue` | Add a week-start control, or create `prop-first-day-of-week.vue`. | The existing example already builds `rows = [0..6]` and `itemRow = getDay()`. |

## 7. Decisions to settle before coding

Each of these changes the diff. Pick, then write the tests to match.

**1. What does the prop rotate?**
*Recommended:* rotate the `rows` array positionally by `Number(firstDayOfWeek)`,
assuming rows are supplied Sunday-first, exactly as the date picker assumes
0 = Sunday.
*Alternative:* look up the row whose key equals the prop value and rotate to it.
Works for `[0..6]` but not for string rows like `['Sun', 'Mon', …]`.

**2. Apply it only in calendar mode?**
*Recommended:* rotate whenever the prop is set, regardless of
`hasExplicitColumns`. A rotated row axis on an explicit-column grid is a coherent
meaning and simpler to document and test.
*Alternative:* ignore the prop unless columns are inferred. A silently ignored
prop is its own surprise.

**3. Locale default when the prop is omitted?**
*Recommended:* match the date picker. When undefined, resolve the start day
through `useDate()` so a German locale gets Monday automatically. This is the
"locale-specific week starts" the issue asks for, and it is why the PR targets
`dev`: existing Monday-first locales will render differently.
*Alternative:* undefined means no rotation, preserving today's output. Safer,
but the user must pass the prop by hand in every locale.

**4. Validation.**
Mirror the adapter's `consoleWarn` for values outside 0 to 6 and fall back to 0.
Tests assert expected warnings with `expect('…').toHaveBeenTipped()`; an
unexpected warning fails the test.

## 8. Workflow, start to PR

The order the maintainers describe in `AGENTS.md`: reproduce, change, test,
lint, then demo in the PR.

### 1. Branch from `dev`

New props are features and go to `dev`, not `master`. Never commit to the base
branches; PRs with unrelated commits are closed.

```bash
git fetch upstream
git switch -c feat/23183-heatmap-first-day-of-week upstream/dev
```

### 2. Set up the Playground

Keep the server running from the repo root with `pnpm dev`. This snippet uses
deterministic data and stacks the cases reviewers will want to see. The
`first-day-of-week` attributes are ignored until the prop exists, which gives a
useful before/after.

```vue
<template>
  <v-app theme="dark">
    <v-container>
      <div class="d-flex flex-column ga-6">
        <div>
          <code>default</code>
          <v-heatmap v-bind="shared" />
        </div>
        <div>
          <code>first-day-of-week="1"</code>
          <v-heatmap v-bind="shared" first-day-of-week="1" />
        </div>
        <div>
          <code>first-day-of-week="6" group-gap="0"</code>
          <v-heatmap v-bind="shared" first-day-of-week="6" group-gap="0" />
        </div>
      </div>
    </v-container>
  </v-app>
</template>

<script setup>
  import { useDate } from 'vuetify'

  const adapter = useDate()
  const start = adapter.date('2026-06-01')
  const items = Array.from({ length: 122 }, (_, i) => {
    const date = adapter.addDays(start, i)
    return { date, value: (i * 7) % 13 }
  })

  const shared = {
    items,
    rows: [0, 1, 2, 3, 4, 5, 6],
    itemRow: v => adapter.toJsDate(v.date).getDay(),
    groupBy: v => adapter.format(v.date, 'monthAndYear'),
    groupGap: 16,
    cellSize: 18,
    gap: 3,
    thresholds: [
      { min: 1, color: '#2d5f3a' },
      { min: 4, color: '#3f8a4f' },
      { min: 8, color: '#5cb86a' },
      { min: 11, color: '#9be3a3' },
    ],
  }
</script>
```

To check the locale-default decision, set `locale: 'de'` in
`dev/vuetify/locale.js` and confirm Monday-first appears without the prop.

### 3. Write the failing tests first

Add cases to the inferred-columns block, run them, see red, then implement.

```bash
cd packages/vuetify
pnpm test src/labs/VHeatmap
```

### 4. Implement

In `heatmap.ts` and `VHeatmap.tsx`, following the components rule: pass
reactive sources down, use `@/util` guards, no comments that narrate code or
cite the issue number.

### 5. Docs and API plumbing

`new-in.json`, the api-generator description if needed, the heatmaps page, and
the calendar example. Run the docs locally to check the example renders.

```bash
pnpm dev docs      # from the repo root, localhost:8095
```

### 6. Lint and typecheck

```bash
cd packages/vuetify
pnpm lint:fix
```

### 7. Commit and open the PR

Conventional commit with the component scope. Paste the Playground into the PR
description as the demo and reference the issue there, not in code comments.

```bash
git commit -m "feat(VHeatmap): add first-day-of-week prop"
```

Target the PR at `vuetifyjs/vuetify` branch `dev`. Write "fixes #23183" in the
description so the issue closes on merge.
