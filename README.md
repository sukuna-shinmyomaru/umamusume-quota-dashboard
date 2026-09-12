# umamusume-quota-dashboard

A static, single-page fan-quota dashboard for an Uma Musume club ("circle") on
[uma.moe](https://uma.moe). Point it at a circle ID to see the month's fan
leaderboard, quota pacing, month-end projections, and the per-member daily quota
needed to hold or reach a club rank tier.

The interface is built to match uma.moe's own web app — the dark theme, cards,
and tier/rank iconography — and is meant to read as if it were a new section of
that site.

## Getting started

No build step, no dependencies, no server — just open `index.html` in a browser
(or host it anywhere static).

1. **An uma.moe API key is required.** Create one at uma.moe under
   **Account → Settings → API Keys**, then paste it into the page. It is stored
   only in your browser's `localStorage` and sent only to uma.moe; it is never
   sent anywhere else.
2. **Add your club** by pasting its circle ID (e.g. `574559219`) or its uma.moe
   URL, and set the daily fan quota.
3. **Optionally set your own player ID** to highlight yourself in your club. Use
   your 12-digit player ID (a leading `#` is fine); a trainer name also works
   when it matches exactly one member. Applies to every saved club.

## Features

- **Leaderboard** — each member's monthly fans, finalized daily gain, daily
  average, quota pace and straight-line projection, with an optional live
  (in-progress day) column. Rendered as a sortable table on desktop and as cards
  on mobile.
- **Summary tiles** — club monthly fans, pace vs quota, members on pace, average
  daily gain, the month-end projection, and the **Current rank** (the global
  monthly standing `#N` with the current tier and the surrounding rank
  thresholds).
- **Club chart** — the club's cumulative fans per competition day against the
  quota-pace line and a month-end projection, with dynamic rank-tier requirement
  bars for both the current and projected totals.
- **Club quota recommendations** — the per-member daily quota required to *keep*
  the current tier or *reach* the next one, projected from each tier bar's
  observed drift and padded by a configurable safety margin.
- **Current player highlight** — optionally set your player ID (or trainer name)
  in settings to mark your own row/card with a gold star and highlight across
  every saved club. When the club contains you, a matching gold-bordered
  **player chart** (cumulative fans with quota-pace, average-club-pace, and
  next-rank-pace lines) and **player quota recommendations** (keep the daily
  quota, match the average club member, or reach the next club rank) appear
  below the club cards. Selecting another member repoints those panels at them
  and switches the outline from gold to blue.
- **Multiple clubs** — add several circles, each with its own saved daily quota
  and optional name, and switch between them from the header dropdown or the
  settings list. Clubs can be edited or removed, and one is kept listed even when
  its data fails to load, so a bad ID stays visible instead of disappearing.
- Search, sorting, a PC/mobile view toggle, and automatic refresh every five
  minutes.

## Data source

All data comes from uma.moe's circle API (`GET /api/v4/circles`). There is no
backend, no build step, and no third-party service — just one HTML file.

## Documentation

- [`docs/uma-api.md`](docs/uma-api.md) — API data model, the JST competition
  calendar and `daily_fans` semantics, the quota/projection math, the rank-tier
  bars, and the recommendations model.

## Credits

Data and interface inspiration: [uma.moe](https://uma.moe). This project is
unofficial and is not affiliated with or endorsed by uma.moe or Cygames.
