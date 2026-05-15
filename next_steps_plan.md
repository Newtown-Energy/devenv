# Next Steps Implementation Plan

A pickup plan for the work that's still outstanding after the
`wip/demo-implementation` overnight pass. Each item lists a concrete
starting point, the files most likely to change, and acceptance
criteria so a future session (human or agent) can take any of them in
isolation.

Companion docs (read these first if you're new to the project):
- [`gap_analysis.md`](gap_analysis.md) — the original gap survey.
- [`implementation_plan.md`](implementation_plan.md) — the
  already-executed B/F task list. The "Stage V" and "Stage A" sections
  at the bottom are the formal stretch sources for items below.
- [`demo_walkthrough.md`](demo_walkthrough.md) — what already works.
- [`test_plan.md`](test_plan.md) — the manual + automated test
  coverage map.

Branch state at the time of writing:
- `neems-core/wip/demo-implementation` — 12 commits ahead of `main`.
- `neems-react/wip/demo-implementation` — 20 commits ahead of `main`.
- Nothing pushed; both branches are local.

---

## Triage at a glance

| ID | Item | Effort | Priority |
|----|------|--------|----------|
| ~~D1~~ | ~~Seed a second site for the Illinois (`no_grid_charge`) flow~~ ✅ landed 2026-05-14 | S | Medium |
| ~~D2~~ | ~~Seed a second admin user for two-user audit demo~~ ✅ landed 2026-05-14 | S | Medium |
| **C1** | Plumb `forcedNow` into the calendar's "now" indicator | M | Medium |
| **C2** | Cancel-as-of-NOW for in-flight commands (script Step 4 fidelity) | M | Low |
| **C3** | SLD-side curtailment indicator on the diagram | M | Low |
| **A1** | Alarms audit — confirm current state of dismiss / hover / history | S | High before A2–A4 |
| **A2** | Dismiss-with-pause for major alarms | M | High |
| **A3** | Alarm history grouping + per-point counts | M | High |
| **A4** | Fire-alarm conditional context (max stack temperature) | M+ (needs backend) | Medium |
| **T1** | Move Bun unit tests into CI | S | Medium |
| **T2** | Add Puppeteer E2E coverage for the new scheduling features | M | Low |
| **U1** | Reset-localStorage affordance for "never show again" dismissals | S | Low |

Effort key: **S** ≈ < 2 hr, **M** ≈ ½ day, **L** ≈ multi-day.

---

## C — Calendar / scheduler polish

### C1. Plumb `forcedNow` into the calendar's "now" indicator

`DemoControlsDrawer` already writes `forcedNow` into the demo
overrides context. Nothing consumes it yet. The natural consumer is
the calendar's "today" cell highlight and the bar-chart's playhead.

- **Files**:
  - `neems-react/src/utils/scheduleHelpers.ts` (`isToday`, `isPastDate`)
  - `neems-react/src/components/CommandCalendar/CalendarGrid.tsx`
  - `neems-react/src/components/CommandCalendar/DayBarChart.tsx`
- **Changes**: introduce a small `useEffectiveNow()` hook that reads
  `overrides.forcedNow` and falls back to `new Date()`. Refactor
  `isToday`/`isPastDate` to accept a `now: Date` arg (default
  `new Date()`) so they can be called against the forced clock.
- **Acceptance**: setting forced now to a future date highlights that
  cell as "today" and dims the cells before it; clearing the override
  snaps back to wall-clock immediately.

### C2. Cancel-as-of-NOW for in-flight commands

Demo Script Step 4 reads: *"Cancel the schedule as if it is 3:30 pm
(user has determined that the hourly load is not high enough)"* with
an audit row "Now to 4pm scheduled for 0 kW". Today the user
approximates this by editing a command's duration to 0, which writes
an audit row but loses the "scheduled to 4pm at 0 kW" framing.

- **Files**:
  - `neems-react/src/components/CommandCalendar/DayDetailsDialog.tsx`
- **Changes**: add a **"Cancel command as of now"** button on each
  active command row when (effective-now from C1) is between the
  command's start and `start + duration`. The action splits the
  command at now: the original keeps its start/effective-now duration,
  a new `0 kW` command runs from effective-now until the original
  end-time, and both audit rows show the actor + timestamp.
- **Acceptance**: split a 15:00–16:00 discharge at 15:30 from the demo
  controls drawer; calendar shows two rows; audit timeline lists the
  original create + the split as two distinct events.
- **Dependency**: C1 (needs a real "now" value).

### C3. SLD-side curtailment indicator on the diagram

Curtailment currently surfaces in the SiteStatePanel banner. The demo
script also asks for a visual indicator inside the single-line
diagram — a small overlay near the interconnection that reads
"Running at curtailment limit — output capped at NNN kW" while
overrides are active.

- **Files**:
  - `neems-react/src/components/SingleLineDiagram/SingleLineDiagram.tsx`
  - `neems-react/src/components/SingleLineDiagram/CurtailmentBadge.tsx` *(new)*
- **Changes**: read `overrides.curtailmentCeilingKw` (or future real
  data point) via `useDemoOverrides`; when non-null and below site
  `power_kw`, render a badge in the diagram's top-right corner.
- **Acceptance**: setting curtailment to 2500 kW on a 5000 kW site
  draws the badge; clearing the override removes it.

---

## A — Alarms pass

These are the original Stage A items from `implementation_plan.md`,
with a note that A1 should land first because it informs A2–A4's
exact file targets.

### A1. Audit pass

- **Files** (read-only): `neems-react/src/pages/AlarmsPage.tsx`,
  `neems-react/src/pages/FDNYPage.tsx`,
  `neems-react/src/components/SingleLineDiagram/`.
- **Output**: update *this* file (or open a follow-up doc) with
  concrete file/line targets for A2–A4. Catalogue what already exists
  for dismiss-with-pause, mouse-over alarm explanations, history
  grouping, and conditional context.
- **Acceptance**: the rest of the A items have explicit file paths +
  function names; no "find the right place" needed at implementation
  time.

### A2. Dismiss-with-pause behavior

- **Files**: components surfaced by A1 (probably `AlarmIndicator.tsx`
  / `AlarmGlow.tsx` under `SingleLineDiagram/`).
- **Changes**: add an **Acknowledge** action that stops the flashing
  animation for `N` seconds (configurable per severity) but keeps the
  red outline on major / fire alarms. Acknowledgement state lives in
  `sessionStorage` keyed by alarm id; expires automatically.
- **Acceptance**: clicking Acknowledge on a flashing fire alarm stops
  the flash, keeps the red outline, and re-enables the flash after
  the configured timeout.
- **Note (Three Mile Island / Chernobyl reasoning)**: the demo script
  is explicit about this — operators couldn't prioritize because they
  couldn't silence the cascade. Keep the visual presence (red outline)
  so the alarm is still findable.

### A3. Alarm history grouping + per-point counts

- **Files**: `FDNYPage.tsx` and/or `AlarmsPage.tsx`, per A1's audit.
- **Changes**: group history rows by *category*
  (electrical / fire / physical / security) and *system*
  (`"Megapack 1C"`, `"Transformer 1"`); add a per-point count column
  so "this transformer alarm has fired 75 times this summer" is one
  glance away.
- **Acceptance**: the FDNY page shows grouped sections with collapse/
  expand and a count badge per point. A sortable count column lets
  the operator find chronically-tripping points.

### A4. Fire-alarm conditional context (max stack temperature)

When the active alarm is in the Fire family, surface the max stack
temperature reading alongside the alarm row.

- **Backend files**:
  - `neems-core/neems-api/src/api/alarm.rs` (extend response)
  - or a new endpoint that returns the most recent reading by point
    name.
- **Frontend files**: the alarm rendering component identified by A1.
- **Changes**: backend exposes `max_stack_temperature_c` (or similar)
  on whatever endpoint feeds the alarm UI. Frontend conditionally
  renders the reading inline with any active fire alarm.
- **Acceptance**: force a fire alarm via the demo controls;
  temperature reading appears next to it. Clear the alarm; the
  reading disappears.

---

## T — Testing

### T1. Move Bun unit tests into CI

`scheduleWarnings.test.ts` + `demoOverrides.test.ts` run locally via
`bun run test:unit`. They're not wired into CI yet.

- **Files**: `.github/workflows/test.yml` (or whatever workflow runs
  on the react repo).
- **Changes**: add a step that runs `bun test src/utils` in the
  `neems-react` job. Set the existing eslint test-file exclusion as a
  blocker — if a future hand alters `eslint.config.mjs` to lint test
  files, the suite re-fails on `bun:test` import errors.
- **Acceptance**: a PR that breaks one of the unit tests goes red in
  CI. Tests run in well under a minute (Bun is fast).

### T2. Puppeteer E2E coverage for the new scheduling features

See `test_plan.md` for the full surface — the "Auto" column lists
specific tests to add. The high-value ones:

- Site selector + Site Defaults panel round-trip (F1).
- Bar chart renders inside a populated day cell (F3).
- Per-day inline edit happy path (F4).
- Peak-season wizard end-to-end (F5).
- Site State Panel renders for each override type (F6/F8).
- Resulting Schedule pane shows a user email after an edit (F7).

- **Files**: `neems-react/test/jest/tests/scheduler-page.test.ts`,
  `library-page.test.ts`.
- **Acceptance**: each test passes against a fresh `setup-demo-data`
  dev stack. Tests are isolated (don't depend on prior test state).

---

## U — UI nice-to-haves

### U1. "Reset warning dismissals" affordance

`dismissWarningPermanently` writes to `localStorage` under
`neems.scheduleWarnings.dismissed`. There's no UI to un-dismiss
short of clearing localStorage manually. `undismissWarning` is
exported for tests but not wired anywhere.

- **Files**:
  - `neems-react/src/components/SiteDefaultsPanel/SiteDefaultsPanel.tsx`
    (or a new Settings page)
- **Changes**: a small "Reset dismissed warnings" button that calls
  `localStorage.removeItem('neems.scheduleWarnings.dismissed')` and
  toasts.
- **Acceptance**: after using "Never show again" on a warning, the
  reset button surfaces; clicking it brings the warning back on the
  next eval.

---

## Loose threads worth knowing about

- **`forcedNow` in the warning engine**: `ScheduleWarningContext`
  still defines `forcedNow` as a forward-looking placeholder. C1 will
  wire it through the calendar; once it's threaded, consider whether
  the engine should also use it for any "command in the past" checks.
- **`schema.patch`**: the diesel `patch_file` (in
  `neems-core/neems-api/src/schema.patch`) only patches sites
  lat/long to `Double`. If you regenerate the schema and a different
  column flips silently, extend the patch file the same way rather
  than hand-editing `schema.rs`.
- **Audit row `user_id` backfill**: `update_latest_activity_user`
  expects the trigger to have just fired. The commands-only update
  path (fixed in commit `e03c8a4` on `neems-core`) now does a no-op
  UPDATE on the parent template to satisfy that contract. If you add
  a new mutation path on a triggered table, follow the same pattern.
- **The demo controls drawer's `forcedNow` field** has no consumer
  yet (see C1). Setting it from the drawer does nothing until C1
  lands.
