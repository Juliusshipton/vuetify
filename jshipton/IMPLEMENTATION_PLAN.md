# Implementation Plan: `first-day-of-week` on `v-heatmap`

Issue #23183. Branch `feat/23183-heatmap-first-day-of-week` off `upstream/dev`.
See `REPO_ANALYSIS.md` for the why; this is the what.

## Decisions (signed off)

1. Rotate the `rows` array positionally by the effective first day (0 = Sunday).
2. Only in calendar mode, i.e. when columns are inferred (`hasExplicitColumns` is false).
   The issue is entirely about the calendar view; explicit-column grids are untouched.
3. Prop omitted → fall back to the locale's first day via the date adapter. The issue
   asks for this outright ("or when the date adapter's locale week start is used as the default").
4. Validate by reusing `adapter.startOfWeek`, which already warns and falls back to
   Sunday. Same semantics as the date picker: `7` wraps to Sunday silently, `-1` or
   `'x'` warn.
5. **Rotation only applies when there are exactly 7 rows.** The offset is "days from
   Sunday", which only maps onto positions in a full week. Needed because decision 3
   makes rotation automatic: the docs example `prop-thresholds.vue` has five inferred
   rows Mon..Fri and would break in a German locale otherwise. Also leaves
   `[1,2,3,4,5]` (weekends hidden) alone.

## Scope notes for the PR description

- **Group-gap geometry stays as is.** The issue's third bullet reads as if
  `group-gap` should not be part of the overlap shift. Today the shift is
  `groupGap - cellWidth`, which deliberately staggers the shared week; the docs
  example's "Group offset" slider demonstrates it. Rotation puts a Monday-first
  lone Sunday through the same detection and shift path as any partial week, which
  is the reporter's actual complaint. Changing the stagger would alter existing
  Sunday-first output, so it is out of scope unless a maintainer asks.
- **`rows` and `item-row` are still required.** The issue's last line hints at the
  heatmap deriving weekday rows from dates itself. That is a date-aware heatmap, a
  separate feature. The docs example keeps `rows = [0..6]` and
  `item-row = getDay()`, it just no longer reorders them by hand.

## Changes

### `src/labs/VHeatmap/pivot.ts`

Add an option so pivot stays date-agnostic:

```ts
export interface PivotOptions<T, C> {
  transformCell?: …
  rowOffset?: MaybeRefOrGetter<number>   // positions to rotate the row axis by
}
```

In the `rows` computed: after building the base list, if
`!hasExplicitColumns.value`, `base.length === 7` and offset is non-zero, return
`[...base.slice(k), ...base.slice(0, k)]` with `k = offset % 7`. No cycle:
`hasExplicitColumns` does not read `rows`.

Everything downstream (`rowIndexByKey`, wrap rule, `rowItems`) already keys off
this order, so nothing else changes here.

### `src/labs/VHeatmap/heatmap.ts`

- `HeatmapProps` gains `firstDayOfWeek?: number | string`.
- `const adapter = useDate()`.
- Effective first day, copied from `useCalendar`:
  ```ts
  const firstDayOfWeek = toRef(() =>
    adapter.toJsDate(adapter.startOfWeek(adapter.date(), props.firstDayOfWeek)).getDay()
  )
  ```
  This gives the locale default and the range warning for free.
- Pass `rowOffset: firstDayOfWeek` into `usePivot(props, { transformCell, rowOffset })`.

Overlap logic (`hasOverlap` on `cells[0] == null`) is untouched and stays correct
after rotation.

### `src/labs/VHeatmap/VHeatmap.tsx`

Add to `makeVHeatmapProps`:

```ts
firstDayOfWeek: [Number, String],
```

Already flows into `useHeatmap(props)`. No render changes.

### `src/labs/VHeatmap/__tests__/heatmap.spec.ts`

Extend `setup()` to accept a locale so a test can run under `de`.
New `describe('useHeatmap firstDayOfWeek')`:

- rotates rows: `[0..6]` + `1` → `[1,2,3,4,5,6,0]`
- wrap point moves: items Sun, Mon, Tue with `1` → Sun alone in column 0, Mon/Tue in column 1
- month starting on the first day has no leading blank (so no overlap)
- month starting on Sunday under Monday-first overlaps and keeps Sunday in the shared column
- string `'1'` behaves like `1`
- locale default: `de` with no prop → Monday first; `en` → unchanged
- invalid `'x'` → warns (`toHaveBeenTipped`) and falls back to Sunday
- 5 rows → not rotated
- explicit columns (`itemColumn` set) → not rotated

### Docs and API

- `packages/docs/src/data/new-in.json`: add `"VHeatmap": { "props": { "firstDayOfWeek": "4.3.0" } }`.
- `packages/api-generator/src/locale/en/VHeatmap.json`: add a short heatmap-specific
  description, since `generic.json`'s wording is about calendars, not row rotation.
- `packages/docs/src/pages/en/components/heatmaps.md`: one paragraph under
  Calendar view.
- `packages/docs/src/examples/v-heatmap/misc-calendar.vue`: add a small
  `v-btn-toggle` (Sun / Mon) wired to `first-day-of-week`.

### Playground

Three stacked cases as drafted in `REPO_ANALYSIS.md` (default, `1`, `6` with
`group-gap="0"`), plus a fourth with `locale: 'de'` set in `dev/vuetify/locale.js`
to eyeball the default.

## Order of work

1. Tests first, see red.
2. `pivot.ts` option → `heatmap.ts` → `VHeatmap.tsx`.
3. Playground check, including RTL is not affected (row headers only).
4. Docs, API, `new-in.json`.
5. `pnpm lint:fix`, commit `feat(VHeatmap): add first-day-of-week prop`.

## Not doing

- No changes to `VHeatmapLegend`, styles, or locale strings.
- No browser spec; all logic is in the composable and unit-testable.
