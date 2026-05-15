# Demo Walkthrough — NEEMs for NYC

Click-by-click playthrough of [`Demo Script v2`](./Demo%20Script%20v2(1).docx)
against the `wip/demo-implementation` branches in `neems-core` and
`neems-react`. Steps that didn't land yet are flagged
**(not implemented)**.

## Setup

| Service | URL | Notes |
|---|---|---|
| Frontend | <http://localhost:5173> | Vite dev server (port from `NEEMS_REACT_PORT`, defaults to 5173) |
| Backend  | <http://localhost:8000> | Rocket API |
| Login    | `superadmin@example.com` / `admin` | Default seeded admin (from `setup-demo-data`) |

**A note on site naming.** The demo script talks about a NYC site
called "Brigis 1A" — that's the real-world site Soltage will deploy on.
The dev DB seeded by `bin/setup-demo-data` doesn't include it; it ships
with placeholder sites **Bat Farm 1**, **Bat Farm 2**, and **Dembowski 1**
(the Illinois site for Part 3, seeded with the `no_grid_charge` variant
and ready-to-use defaults). Throughout this walkthrough, **Bat Farm 1**
stands in for Brigis 1A.

**Start the stack** (from `devenv/`):

```bash
docker compose up -d
docker compose ps        # confirm neems-api + neems-react are "Up"
```

If anything looks stale, watch logs with `docker compose logs -f neems-react`.

**Before running the demo,** open the Demo Controls drawer once and hit
**Reset all overrides** — sessionStorage from a prior tab can carry forced
SoC / curtailment / breaker state into the new session.

**Roles in play:** the admin-only Demo Controls drawer and the peak-season
wizard need `admin`, `newtown-admin`, or `newtown-staff`. The seeded
`superadmin@example.com` has `newtown-admin`, which is sufficient.

---

## Part 1 — Scheduling arc (Bat Farm 1, NYC)

### Step 1 — Confirm site defaults

**Go to** <http://localhost:5173/scheduler>

The header has a site selector — pick **Bat Farm 1** (the seeded site).

Open the **⋮** overflow menu in the top-right of the scheduler header
and pick **Site defaults**. The dialog opens with every value the wizard
depends on.

> ⚠️ **Heads-up on a freshly-seeded DB.** The "Bat Farm 1" row was
> created by `bin/setup-demo-data` *before* the B1 migration added the
> demo columns. So on first open most fields are empty — `Site power`,
> `Site capacity`, both window pairs, and `Interconnection max output`
> all show blank inputs. Only `Max ramp duration` (120), `Closed-loop
> control` (on), `Rebound protection SoC floor` (2), and `Site variant`
> (Standard interconnect) have meaningful defaults from the migration.
> This is the moment to fill them in.

Fill in (or confirm) each field:

| Field | Demo value |
|---|---|
| Site power | **5000 kW** |
| Site capacity | **23500 kWh** (script implies 5 MW / 23.5 MWh) |
| Max ramp duration | **120 s** (already populated by the migration) |
| Closed-loop control | **on** (already populated) — banner appears on the page if toggled off |
| Off-peak window | **00:00 – 08:00** |
| Peak-revenue window | **16:00 – 20:00** |
| Interconnection max output | **5000 kW** |
| Rebound protection SoC floor | **2 %** (already populated) |
| Site variant | **Standard interconnect** (already populated) |

Click **Save defaults**. The "Defaults saved" toast confirms the round-trip.

> 💡 The demo script's "Maximum ramp rate? 41 kW per second" framing is
> stored as `ramp_duration_seconds` (120 s) rather than kW/s. Same idea,
> different unit on the wire — the UI doesn't translate it back to kW/s.

### Step 2 — Build the season default with the wizard

Still on `/scheduler`, click the blue **Peak-season wizard** button. Step
through:

1. **Site power** — confirm `5000 kW` and the closed-loop toggle.
2. **Off-peak charging** — `00:00 – 08:00`, charging power `2950 kW`.
3. **End-of-charge SoC** — `100`.
4. **Peak revenue** — `16:00 – 20:00`.
5. **Interconnection** — `5000 kW`.
6. **Rebound protection** — `2 %`.
7. **Season range**
   - Schedule name: `Bat Farm 1 site defaults for 2026 New York VDER`
   - Start date: `2026-06-24`
   - End date: `2026-09-15`
   - Weekdays only: **on**
   - Skip US federal holidays: **on**
8. **Review** → **Apply**.

The success dialog reports `Created Bat Farm 1 site defaults for 2026 New
York VDER and applied it to N days` — for 2026-06-24 → 09-15 weekdays
minus Jul 3 (4th observed) and Sep 7 (Labor Day), `N = 58`.

Close the dialog. **Scroll to June 24, 2026** on the calendar. Each
weekday cell has:
- A small chip naming the schedule.
- An orange downward bar from 00:00–08:00 (charging at 2950 kW).
- A blue upward bar from 16:00–20:00 (discharging at 5000 kW).

> Color contract: orange = charge, blue = discharge, muted-orange =
> trickle charge. Green and red are reserved for breaker / fault state.

### Step 3 — Add a one-off peak discharge (3pm–4pm @ 5000 kW)

Click any weekday in the season range. The day details dialog opens.

For the demo's "start the BESS at 3pm" beat, you want to override the
default just for that day. Click **Edit Schedule**, choose **Create a
copy and edit it** in the next dialog, accept the auto-generated name
(`Bat Farm 1 site defaults for 2026 New York VDER (2)`), click
**Continue**.

The calendar refreshes; the day is now on its specific-date copy. Click
the day again. Because the rule is now `specific_date`, you can edit
inline:

- Click **Add command** in the Commands section.
- Set Time `15:00`, Type `Discharge`, Duration `1 h`, Target SoC blank.
- Save.

**Expected warnings** (in the edit dialog as you fill the form):

- "Discharge scheduled outside the peak-revenue window — revenue per
  kWh will be lower." *(15:00 is just outside 16–20.)*

After save the day cell shows the new 15:00 blue bar alongside the
default 16:00 bar.

> The script's exact phrasing ("Con Edison will not trip for this
> schedule") is not implemented — the warning here is the peak-revenue
> one. Same intent, different wording.

### Step 4 — Cancel the 3pm command "as if it's 3:30 pm"

The script wants the audit trail to show the cancel as a *new* event,
not a delete. The cleanest way today is to set the command to 0 kW (or
edit its duration to 0).

- Click the day again, click the pencil on the 15:00 command, change
  Duration to `0 h 0 min`, Save.

The Commands table updates. **Resulting Schedule** pane below the
warnings shows both the prevailing rule and the audit timeline ("Updated
by superadmin@example.com at …" plus the original "Created by …").

> The script's "Now to 4pm scheduled for 0 kW" with full audit pane is
> partially implemented — the audit timeline is wired (F7) but the
> "force time = 3:30 pm" interaction with auto-cancel is **not
> implemented**.

### Step 5 — Charge outside off-peak / discharge inside off-peak

These exercises live in the same per-day edit dialog. Pick a fresh
weekday so warnings aren't muddied by Step 3's commands.

1. Click the day → **Edit Schedule** → **Create a copy** → **Continue**.
2. Click the day again → **Add command**.
3. **Charge at 16:00**: Time `16:00`, Type `Charge`. Expected warning:
   "Charging scheduled outside the off-peak window — energy costs will
   be higher." *(Dismissible with "Never show again".)*
4. Cancel without saving.
5. **Discharge at 01:00**: Time `01:00`, Type `Discharge`. Expected
   warning: "Discharge scheduled inside the off-peak charging window —
   this fights the charging plan." *(Dismissible.)*

> The script's exact "Con Edison may trip" phrasing isn't used; the
> warnings keep the same *intent* but the language is generic
> ("revenue", "fights the charging plan", etc.).

### Step 6 — Low SoC site-state indicator

> **A note before this step.** The demo script reads the SoC warning as
> if it lives inside the schedule editor. In this implementation it
> doesn't. SoC, breaker state, Megapack availability, and curtailment
> are properties of the *current* site, not properties of a future
> command — a 4pm discharge tomorrow doesn't care what SoC reads right
> now, because by tomorrow afternoon the morning charge will have
> moved it. So those readings surface in a **persistent app-wide
> indicator** and the **SLD page**, not in the schedule edit dialog.
> The schedule edit dialog keeps only schedule-shape warnings (window
> mismatches, variant mismatch, interconnection cap).

Click the demo bug icon (top-right of `/scheduler`) to open the **Demo
Controls drawer**.

- **Current SoC:** `10` %
- Leave everything else empty.

Close the drawer. A site-state **banner** appears at the top of the
page (on every page *except* `/sld`, which mounts its own copy):

> 🟡 **Low SoC** — Battery state of charge low: 10% (rebound floor 2%).

Now set SoC to `1`% in the drawer — the row escalates to red because
the value is at or below the rebound floor.

The same banner is rendered on `/sld` as the "Site State Panel" above
the diagram (one copy, not two — the app-wide banner is suppressed
when you're already on the operational view).

### Step 7 — Open breakers / Megapack offline

Re-open the **Demo Controls drawer**.

- In **Breakers open**, pick `B-1` from the dropdown.
- In **Megapacks offline**, pick `Megapack-A`.

Close the drawer. The site-state banner now lists three rows (the low
SoC from Step 6 + the two new ones). On `/sld` the same content shows
up as the panel above the diagram:

- 🔴 *Open breakers* — "Breakers open: B-1. Commands will not execute."
- 🟡 *Megapack offline* — "Megapacks offline: Megapack-A. Available power is reduced."
- 🔴 *Low SoC* — "Battery state of charge low: 1% (rebound floor 2%)."

(Variant of the Illinois flow: set **Utility curtailment** to `2500` kW
on a 5000 kW site to add a curtailment-active row.)

Open a day's edit dialog on the calendar to confirm these issues do
*not* show up inline — the dialog stays focused on schedule-shape
warnings. Clear everything via **Reset all overrides** in the drawer
when you're done.

### Step 8 — Two-user audit trail

The audit timeline in the Resulting Schedule pane shows the actor's
email. To exercise the multi-user scenario:

1. Edit a day as `superadmin@example.com`.
2. Log out (sidebar account menu).
3. Log in as `operator@example.com` / `operator` (seeded by
   `setup-demo-data` as a second admin on Sunny Solar — the company
   that owns Bat Farm 1).
4. Edit the same day.
5. Click the day; the audit timeline in the Resulting Schedule pane
   shows both names with timestamps.

---

## Part 2 — Alarms / SLD arc

**Go to** <http://localhost:5173/sld>

The script's "force a bit to make the screen object change" workflow
runs through the existing `SingleLineDiagram` component plus the new
**Trigger alarms** section in the Demo Controls drawer.

### Step 9 — Manual breakers / E-stop

- Click breaker switches `89L-1` / `89L-2` to toggle position. The
  whole switch icon is a click target (not just the stroked outline).
- Trigger the red **E-STOP** button for a confirmed site-wide lockout
  (the `Alert` banner appears at the top of the page).
- The stale-data banner appears if the alarm service falls behind the
  60-second threshold.

### Step 10 — Trigger alarms from the drawer

Open the **Demo Controls drawer** (bug icon, bottom-right) and scroll to
**Trigger alarms**.

1. Pick an alarm from the dropdown. The list is sorted by alarm number
   and each row shows the alarm name plus a severity chip
   (Emergency/Critical/Warning/Info). The set covers all RTAC-defined
   alarms — useful demo picks:
   - `#401 fire_alarm` (Emergency, FACP zone) — drives the fire-alarm
     panel into Emergency state and lights up the FACP element.
   - `#104 estop` (Critical, BreakerRelay zone) — same logical effect
     as clicking the E-STOP button.
   - `#101 bps_89l1_open` (Info) — milder signal on the line breaker.
2. The chosen alarm appears as a deletable chip below the dropdown and
   immediately shows up on `/sld` (AlarmGlow + AlarmIndicator on the
   affected element) and in `/alarms`. Both pages are polling
   `/api/1/Alarms/Active` — the backend overlays the forced set into
   that response, so the SLD, alarms page, and FDNY view all stay in
   sync.
3. Add more alarms; drop one by clicking its chip's × icon; clear all
   at once with **Reset all overrides**.

Forced alarms live on the server (in-memory `Mutex<HashSet<u16>>`,
exposed via `/api/1/Alarms/Forced`), so they persist across page
reloads and tab switches but reset when the API restarts. The endpoint
is gated to admin / newtown-admin / newtown-staff — same gate as the
drawer itself.

> Why server-side? Tab-local overrides (SoC, breakers, curtailment)
> only need to reshape *what the operator sees*. Forced alarms also
> need to drive everyone polling `/Alarms/Active` (SLD, alarms page,
> FDNY view, any future consumer), so they belong on the server.
> The endpoint is marked temporary — it goes away when the real RTAC
> feed is hooked up.

**Not implemented** in this branch:
- Dismiss-with-pause for major alarms (still flashes the red outline
  until cleared).
- Per-system alarm history grouping with per-point counts.
- Conditional context (max stack temperature on fire alarms).
- Mouse-over alarm explanations.

The implementation plan has these queued as stretch tasks (`A1`–`A4`)
and they're not in the `wip/demo-implementation` branch.

---

## Part 3 — Illinois "Dembowski" site

The script's `no_grid_charge` variant is exercised by the seeded
**Dembowski 1** site under Sunny Solar — pre-populated with the same
5 MW / 23.5 MWh shape as Bat Farm 1 but with `site_variant`
= `no_grid_charge`.

1. Go to `/scheduler` and pick **Dembowski 1** from the site selector.
2. Open the **⋮** overflow menu → **Site defaults** to confirm the
   variant reads **No grid charge (inverters cannot charge from grid)**
   and the windows/power are already populated.
3. Build a season default with the **Peak-season wizard** the same way
   as Part 1 (or click a day directly if you skip the wizard for this
   site).
4. Click a day → **Edit Schedule** → **Create a copy** → **Continue**.
5. Click the day → **Add command** → Time `16:00`, Type `Charge`.

Expected hard error (red, non-dismissible):

> "Inverters at this site cannot charge from the grid — remove this
> command or switch the variant."

Now exercise the curtailment-ceiling flow. Per the site-state vs.
schedule-shape split (see Step 6's note), curtailment is a current
site fact, not a property of a future command, so it surfaces in the
app-wide banner and on the SLD page rather than in the schedule
editor:

6. Open the **Demo Controls drawer**.
7. Set **Utility curtailment** to `2500` kW.

The site-state banner now shows:

> 🟡 **Curtailment active** — Utility curtailment active: output capped at 2500 kW (site power 5000 kW).

**Reset** when you're done. Switching the site selector back to
**Bat Farm 1** is enough — no variant-flip cleanup needed.

---

## Cleanup

| Action | How |
|---|---|
| Reset demo overrides | Demo Controls drawer → **Reset all overrides** |
| Remove the season's library item | `/library` → find `Bat Farm 1 site defaults for 2026 New York VDER` (and its copies) → delete |

If anything sticks, `sessionStorage.clear()` in the browser devtools
will wipe demo overrides; `localStorage.clear()` wipes the persisted
"never show again" warning dismissals.

---

## Quick reference — gap vs. demo script

| Script beat | Branch status |
|---|---|
| Site defaults panel | ✅ implemented (F1) |
| Peak-season wizard | ✅ implemented (F5) |
| Calendar with charge/discharge bars | ✅ implemented (F3) |
| Specific-date inline command edits | ✅ implemented (F4) |
| Schedule-shape warnings (off-peak / peak-revenue / variant / interconnection) | ✅ inline in the edit dialog |
| Site-state warnings (breakers / megapacks / SoC / curtailment) | ✅ rendered as an app-wide banner via SiteStatePanel, suppressed on /sld where the page already hosts its own copy |
| Demo controls drawer (forced now, curtailment, SoC, breakers, megapacks) | ✅ implemented (F8) — `forcedNow` is plumbed but not yet consumed by the calendar's "now" indicator |
| Force an alarm to drive the SLD / alarms page | ✅ implemented — server-side `/api/1/Alarms/Forced` + drawer "Trigger alarms" section |
| Resulting Schedule pane with provenance | ✅ implemented (F7) |
| Multi-user audit | ✅ implemented (F7) — `operator@example.com` seeded as a second admin |
| Alarm dismiss-with-pause | ❌ stretch (A2) |
| Alarm history grouping + per-point counts | ❌ stretch (A3) |
| Fire-alarm max stack temp | ❌ stretch (A4) |
| Dembowski site seeded as its own row | ✅ implemented (D1) — `Dembowski 1` under Sunny Solar |
