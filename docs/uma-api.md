# uma.moe circle API — data model & quota math

Reference for the dashboard in `index.html`. Based on direct inspection of the live
API and cross-checked against uma.moe's own numbers and the open-source
`UmaCore` / `DustBunnyLeaderBot` projects.

## Endpoint

```
GET https://uma.moe/api/v4/circles?circle_id=<id>&year=<Y>&month=<M>
Header: X-API-Key: uma_k_...
```

- CORS is `Access-Control-Allow-Origin: *` and `x-api-key` is allowed, so the browser
  can call the API directly: each visitor supplies their own key, which is stored in
  `localStorage` and sent only to uma.moe. There is no backend.
- `month` is **1-based** (9 = September). Requests for a future month return HTTP 400.
- An invalid/missing key returns `401 {"error":"invalid_api_key"}`. Requests without a
  key may return `403 {"error":"browser_proof_required"}`.

### Browser storage

Everything is kept client-side in `localStorage`:

- `umaQuota.v1` — the active club's circle ID, daily quota, and the browser-wide API key.
- `umaQuota.circles.v1` — the saved clubs, `{id, name, quota, ts}` each, newest first,
  capped at 8. Each club carries its own quota; `name` is the API name unless the user
  set a `customName`.
- `umaQuota.margin.v1` — the safety margin %.
- `umaQuota.view.v1` — the PC/mobile layout choice.
- `umaQuota.tz.v1` — whether the header/footer clocks show `jst` (default) or `local`.
- `umaQuota.player.v1` — the optional current player, `{id, name}`, used to highlight that
  member's row/card. `id` is the 12-digit numeric `viewer_id` (a leading `#` in the input is
  stripped); `name` is only used when it matches exactly one member. Browser-wide, so it
  applies to every saved club.

A club is written to the list as soon as it is added, so a club whose fetch fails stays
listed (with an error marker) instead of disappearing.

## Response shape

```jsonc
{
  "circle": {
    "circle_id": 574559219,
    "name": "Dynasty",
    "leader_viewer_id": 652861639566,
    "member_count": 30,
    "last_updated": "2026-09-10T15:01:07Z",     // finalize marker
    "last_live_update": "2026-09-10T22:21:10Z",  // in-progress refresh
    "monthly_point": 877119814,   // club total through the last CLOSED day
    "yesterday_points": 797998430,
    "live_points": 906062662,     // club total including the in-progress day
    "monthly_rank": 605, "yesterday_rank": 581, "live_rank": 605,
    "last_month_rank": 669, "last_month_point": 2967314581,
    "join_style": 3, "policy": 8, "archived": false
  },
  "club_rank": 7,                      // club tier (1..11); drives the rank icon on uma.moe
  "fans_to_next_tier": 136307906,      // fans needed to reach the tier above (gap, not a total)
  "fans_to_lower_tier": 285934843,     // buffer (fans above the tier below)
  "yesterday_fans_to_next_tier": 110344399,
  "yesterday_fans_to_lower_tier": 258611376,
  "members": [
    {
      "viewer_id": 150527576367,
      "trainer_name": "あゆみ",
      "shame_score": 2,
      "year": 2026, "month": 9,
      "daily_fans": [718800438, 722125054, /* … 32 slots … */, 0, 0],
      "last_updated": "2026-09-10T22:21:10Z"
    }
  ]
}
```

Notes:
- `members` keeps people who left mid-month (their later slots are `0`).
- There is **no `role`/`membership` field** on this endpoint. Leader is inferred from
  `circle.leader_viewer_id`; officers cannot be distinguished.
- `daily_fans` is always 32 slots.

## The `daily_fans` array

`daily_fans[i]` = that member's **lifetime cumulative** fan total at the end of
JST day `i+1`.

| index | meaning |
|---|---|
| `0` | baseline — total at the previous month's close |
| `D` | competition day `D` (which is JST day `D+1`) |

- Competition months run on JST and finalize at **15:00 UTC** (00:00 JST).
- `last_updated` is written at `15:01:00Z`; use it to confirm a day is finalized.
- Negative entries are **transfer markers** from a prior circle — treat as gaps and
  use the first positive entry as the baseline.
- Internal zero gaps mean a missed scrape; forward-fill from the previous value.

### Slot resolution (used by `resolvePeriods()` in `index.html`)

Let `j` = today's JST date.

| target | computation | example (JST day 11) |
|---|---|---|
| finalized competition day | `j − 2` | Sep 9 → slot `9` |
| finalized API month | month of `j − 2` | `2026-09` |
| in-progress competition day | `j − 1` | Sep 10 → slot `10` |

The finalized slot (`9`) reproduces `circle.monthly_point` after ignoring today's
in-progress slot. Verified on circle `574559219`: derived `873,935,509` vs API
`877,119,814` — **0.36%** (the documented residual).

## Quota & projection math

Per member, for the finalized slot `F` (competition days completed) and daily quota `Q`:

```
baseline      = first positive daily_fans value
monthly       = daily_fans[F] − baseline
presentDays   = F − baselineIndex        // mid-month joiners are not back-charged
expected      = Q × presentDays
deficit       = monthly − expected       // ≥ 0 → on pace, < 0 → behind
dailyAvg      = monthly / max(1, presentDays)
projected     = monthly + dailyAvg × (daysInMonth − finalSlot)  // join-aware to month end
quotaTarget   = Q × (daysInMonth − baselineIndex)   // join-aware month-end obligation
projectedDeficit = projected − quotaTarget
```

Every month-end figure is join-aware: a member who joined mid-month is charged only from
their join slot, so their projection never credits fans from before they were in the club.

Club aggregates sum the members above. The "Projected month" summary stat shows
`clubProjected = Σ member.projected` and compares it against the join-aware full-month
obligation `clubQuotaTarget = Σ quotaTarget`. The chart plots club cumulative fans per slot
(solid blue) against a dotted blue straight-line projection to month end ending at
`clubProjected`, and the rank boundaries below. The **quota pace** is a forward amber segment
from the current total:

```
quotaFromNowEnd = C + Q · members · (daysInMonth − finalSlot)
```

Projecting from the actual total answers "if we earn exactly the quota from now on, where do
we finish?", so a club that is ahead of the schedule keeps its month-to-date surplus. A
from-origin schedule line would instead end at `Q · members · daysInMonth` regardless of the
surplus and can make a quota that comfortably holds the rank look like it barely does; the
chart deliberately omits it. The "vs quota pace" summary stat still tracks schedule
adherence.

## Rank-tier requirement bars

`fans_to_next_tier` / `fans_to_lower_tier` are **gaps**, not totals, and they are relative
to the club's own total (`circle.monthly_point`). The absolute tier thresholds are therefore:

```
nextNow  = monthly_point + fans_to_next_tier   // green: to reach the tier above
floorNow = monthly_point - fans_to_lower_tier  // red:   to hold the current tier
```

These thresholds are **global** scores set by other clubs, so they are projected
independently of our club's data: keep each threshold's month-to-date pace and extend it to
the end of the month. A threshold at `B` after `finalSlot` of `daysInMonth` days projects to:

```
thresholdProj = thresholdNow · daysInMonth / finalSlot
```

E.g. a floor at 10M at the halfway point projects to 20M. (`yesterday_*` is not needed.)
The chart draws four 50%-opacity horizontal bars, localized to the relevant x position
(`now` → a short bar around the current day; `proj` → a short bar over the last two days of
the month): `nextNow` / `nextProj` (green) and `floorNow` / `floorProj` (red). The exact
same boundaries drive the quota recommendations (below), so the chart and the tiles can
never disagree.

The legend carries the meaning ("next rank <N>" / "current rank <C>"), so each bar is
labelled only `now` or `projection`. "next" labels sit above their bar and "current
rank" labels below, so a line never crosses its label; labels go left of the data point
at day 15+ and right otherwise (projection is always left). Two buttons in the chart
header toggle the groups independently (`Ranks now`, `Ranks projected`), both on by default.

`club_rank` is the club's tier (1..11) and maps to a name. `monthly_rank` is the global
monthly standing shown as the "Current rank" summary value (`#N`); its sub-line names the
current tier and the surrounding "now" thresholds (`<tier> floor` = `nextNow`/`floorNow`
above, from `monthly_point`).

## Quota recommendations

The dashboard's recommendations card turns the same projected tier boundaries the chart
draws (see above) into an actionable per-member daily fan target. With `C = clubMonthly` (the
finalized total the chart's actual line ends at), `R = daysInMonth − finalSlot`, and `M` the
present-member count:

```
boundaryNow  = monthly_point ± gap                 // nextNow or floorNow, see above
boundaryProj = boundaryNow · daysInMonth / finalSlot
requiredPace = (boundaryProj − C) / R              // club fans / day
quota        = max(0, requiredPace) / M · (1 + margin)   // recommended per member / day
```

The threshold is a global score that moves independently of our own total, so the card
targets its month-to-date-pace projection rather than our own projected total. For "Reach",
this charges enough pace to close the remaining gap to where the bar is heading; for "Keep",
the projected boundary is usually below `C`, so the required pace can fall below the club's
current pace.

The margin (default **10%**, edited in the card and persisted in `localStorage` under
`umaQuota.margin.v1`) is the safety cushion applied to the required pace. A boundary whose
projected total falls below `C` requires no quota. Such a tile shows the **buffer** instead:
the absolute cushion `C − boundaryNow` as the headline, plus the equivalent per-day
**cushion** `(C − boundaryNow) / (R · M)`.

Every tile (safe or not) also shows the recommended total and two signed deltas:

```
vs quota = quota − current daily quota            // the field in the controls
vs pace  = quota − (P − C) / (R · M)              // current per-member / day pace
```

`vs pace` compares against how fast present members are actually gaining, i.e. the same
straight-line `clubProjected` used for the clear/short label.

Edge cases: at the top tier (`club_rank = 11`, `fans_to_next_tier` null) there is no
next-rank target; a missing gap simply hides that tile. `R = 0` (month finalized) shows
"month complete"; a payload without tier fields shows an empty state.

Note: the "projected to clear/short" label compares `clubProjected` `P` with the
**projected** boundary `boundaryProj`, and the chart's projection segment ends at the same
`clubProjected` while its projected rank bars use the same boundaries. The "Projected month"
summary stat, the chart and the recommendations therefore always agree.

| club_rank | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| name | D | D+ | C | C+ | B | B+ | A | A+ | S | S+ | SS |

## Player recommendations

When a member is focused (the configured current player by default, or whichever
member is selected), two extra cards describe that single member: a cumulative
chart of their own series, and per-day targets:

```
remaining = daysInMonth − finalSlot
keepQuota = max(0, (quota · (daysInMonth − joinedSlot) − memberMonthly) / remaining)
clubPace  = (clubMonthly / finalSlot) / presentMembers
reachNext = max(0, nextQuota)      // the club "Reach" quota, per member / day
```

`keepQuota` is the daily rate needed from now on to finish the month exactly on
the club's daily quota — a catch-up figure when the member is behind, otherwise 0.
`clubPace` is the average present member's finalized daily rate. `reachNext` is
the per-member share of the club pace needed to reach the next rank tier (the same
time-scaled and margin-adjusted figure as the club card's "Reach" tile). Each tile shows
its target next to the member's own current daily average (`memberMonthly /
presentDays`); the third tile names the next tier and, at the top tier, explains
that there is nothing higher. The player chart's **quota line** runs from the member's
current total to their join-aware month-end target `quota · (daysInMonth − joinedSlot)`,
so its slope is exactly `keepQuota` and it always agrees with the "Keep up with quota"
tile (it is flat when the member is already at or above target). The chart also draws the
average club pace and the next-rank pace, both from the member's join day to month end.
The two player cards are outlined and tinted gold when they show the
configured current player, blue when they follow a selected other member, and stay
hidden entirely in a club the player does not belong to until a member is selected.

## Hosting

The app is a single static `index.html` — no build step, no server, no dependencies.
Host the repository (or just that file) on any static host, or open it locally. Each
visitor enters their own uma.moe API key; it is stored in that browser's `localStorage`
and sent only to uma.moe. The repository never contains any key. The browser talks to
uma.moe cross-origin; uma.moe's CORS allows this.

## Local dev

No build step and no dependencies — just open `index.html` in a browser.
