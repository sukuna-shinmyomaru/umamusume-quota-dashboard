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
projected     = dailyAvg × daysInMonth   // straight-line to month end
projectedDeficit = projected − Q × daysInMonth
```

Club aggregates sum the members above. The chart plots club cumulative fans per slot
(solid blue) against the straight
`Q × members × day` pace line (amber dashed) and a dotted blue straight-line projection
to month end.

## Rank-tier requirement bars

`fans_to_next_tier` / `fans_to_lower_tier` are **gaps**, not totals: add/subtract them
from the club's current fan total to get the tier thresholds. They are dynamic (they
move as other clubs gain fans), and `yesterday_*` holds the same gap as of the prior
finalized read. The dashboard draws four 50%-opacity horizontal bars on the chart,
localized to the relevant x position (`now` → a short bar around the current day;
`proj` → a short bar over the last two days of the month):

```
nextNow  = finalizedTotal + fans_to_next_tier   // green, solid
nextProj = projectedTotal + fans_to_next_tier   // green, dashed
curNow   = finalizedTotal - fans_to_lower_tier  // red,   solid
curProj  = projectedTotal - fans_to_lower_tier  // red,   dashed
```

The legend carries the meaning ("next rank <N>" / "current rank <C>"), so each bar is
labelled only `now` or `projection`. "next" labels sit above their bar and "current
rank" labels below, so a line never crosses its label; labels go left of the data point
at day 15+ and right otherwise (projection is always left). Two buttons in the chart
header toggle the groups independently (`Ranks now`, `Ranks projected`), both on by default.

`club_rank` is the club's tier (1..11) and maps to a name. `monthly_rank` is the global
monthly standing shown as the "Current rank" summary value (`#N`); its sub-line names the
current tier and the surrounding "now" thresholds (`<tier> floor` = current total −
`fans_to_lower_tier`, `next <tier>` = current total + `fans_to_next_tier`).

## Quota recommendations

The dashboard's recommendations card turns the tier bars into an actionable per-member
daily fan target. Target totals come from the live tier gaps, same as the chart bars,
using the finalized total `C = clubMonthly`:

```
keepTarget = C − fans_to_lower_tier   // floor of the current rank
nextTarget = C + fans_to_next_tier    // bar for the rank above
```

The bars are **dynamic** — they move as rival clubs gain — so each bar is projected forward
over the remaining competition days `R = daysInMonth − finalSlot` using its observed
day-over-day drift. The drift comes from the API's own current-vs-yesterday pair, so it is
independent of the derived `C`:

```
keepDrift = (monthly_point − fans_to_lower_tier) − (yesterday_point − yesterday_fans_to_lower_tier)
nextDrift = (monthly_point + fans_to_next_tier)  − (yesterday_point + yesterday_fans_to_next_tier)

projectedBar = target + drift · R
requiredPace = drift + (target − C) / R            // club fans / day
quota        = max(0, requiredPace) / M · (1 + margin)   // recommended per member / day
```

The margin (default **10%**, edited in the card and persisted in `localStorage` under
`umaQuota.margin.v1`) is the safety cushion applied to the required pace. A bar whose
projected total falls below `C` requires no quota. Such a tile shows the **buffer** instead:
the absolute cushion `C − target` as the headline, plus the equivalent per-day **cushion**
`(C − target) / (R · M)`. When `yesterday_*` is unavailable the drift is 0 and the bars stay
static.

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

Note: the "projected to clear/short" label uses `clubProjected` `P` vs the **projected**
bar `target + drift · R` (so it agrees with the recommended pace), matching the "Projected
month" summary stat. The chart's *projected* rank bars instead use a club-level straight
line `clubMonthly / finalSlot · daysInMonth` and the un-projected gap, so the two can differ
slightly when members joined mid-month or the bar is drifting.

| club_rank | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| name | D | D+ | C | C+ | B | B+ | A | A+ | S | S+ | SS |

## Player recommendations

When a member is focused (the configured current player by default, or whichever
member is selected), two extra cards describe that single member: a cumulative
chart of their own series, and per-day targets:

```
remaining = daysInMonth − finalSlot
keepQuota = max(0, (quota · daysInMonth − memberMonthly) / remaining)
clubPace  = (clubMonthly / finalSlot) / presentMembers
reachNext = max(0, nextQuota)      // the club "Reach" quota, per member / day
```

`keepQuota` is the daily rate needed from now on to finish the month exactly on
the club's daily quota — a catch-up figure when the member is behind, otherwise 0.
`clubPace` is the average present member's finalized daily rate. `reachNext` is
the per-member share of the club pace needed to reach the next rank tier (the same
drift- and margin-adjusted figure as the club card's "Reach" tile). Each tile shows
its target next to the member's own current daily average (`memberMonthly /
presentDays`); the third tile names the next tier and, at the top tier, explains
that there is nothing higher. The player chart draws a dashed line for each pace
(quota pace, average club pace, and the next-rank pace) from the member's join day
to month end. The two player cards are outlined and tinted gold when they show the
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
