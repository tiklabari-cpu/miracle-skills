---
name: analytics-panel
description: Plan-gated, timezone-aware analytics dashboard on SQLite (Node/Express + React). Use when building a click/event analytics panel with date-range filters, per-plan feature gating, geo/device breakdowns, CSV export, and shareable public reports. Reusable pattern extracted from the Tıklatbari project.
---

# Analytics Panel (SQLite, plan-gated, timezone-aware)

A production-tested pattern for a click/event analytics dashboard. Backend: Node/Express +
`better-sqlite3`. Frontend: React + Tailwind + Recharts. Every piece here is battle-tested; the
**Gotchas** section lists real bugs found via review + live testing — read it.

## Data model
`analytics` table, one row per event (append-only). Timestamps stored **UTC**:
```
id, link_id (or entity_id), ip, country, city, region, timezone, language,
device, browser, os, referrer, created_at TEXT DEFAULT (datetime('now'))   -- UTC
```
Indexes: `(link_id)`, `(created_at)`. Log asynchronously (`setImmediate`) after the response so
event logging never adds latency to the hot path (e.g. a redirect).

## Backend: buildScope → buildSummary
Centralize the WHERE-clause builder so summary, export, and public report share it.

`buildScope(userId, { linkId, range, tzOffset, plan, tag })` returns `{ where, params, off, ids }`:
- **Retention:** `a.created_at >= datetime('now','-<historyMonths> months')` (per-plan).
- **Scope:** `a.link_id IN (...)`, optional single-link, optional tag filter.
- **Date range** (see below). Returns `off` (tz offset) so summary/export reuse it.
- Returns `null` when the user has **no entities** (distinct from `{invalid:true}` for a bad id).

`buildSummary` runs COUNT/GROUP BY queries off `FROM analytics a WHERE ${where}`: totalClicks,
uniqueVisitors (`COUNT(DISTINCT ip)`), repeatClicks, clicksToday, timeline (grouped by local date),
and breakdowns (device/browser/os/country/referrer; detailed geo only when the plan allows).

## Date ranges (enum → SQLite lower-bound, injection-safe)
Map a **whitelisted** range key to a constant SQLite expression (never interpolate user input):
```js
const signedMinutes = (m) => `${m >= 0 ? '+' : ''}${m} minutes`;
function rangeLowerBound(range, off) {   // off = tz offset in minutes (east +)
  switch (range) {
    case '1h':  return "datetime('now','-1 hours')";
    case '24h': return "datetime('now','-24 hours')";
    case '7d':  return "datetime('now','-7 days')";
    case '30d': return "datetime('now','-30 days')";
    case '90d': return "datetime('now','-90 days')";
    case 'day': return `datetime('now','${signedMinutes(off)}','start of day','${signedMinutes(-off)}')`;
    default:    return null; // 'all'
  }
}
```
`off` is parsed to an integer and clamped `[-720,840]` → the expression is a fixed whitelist string,
so there is **no SQL injection** despite string-building.

## Timezone (offset-driven, no tz database)
The client sends `tzOffset` (minutes east of UTC) derived from a user preference (e.g. "Local" =
`-new Date().getTimezoneOffset()`, or a fixed zone like Istanbul = `+180`). Backend uses it for:
- The **`day`** range (calendar-day boundary at the user's midnight).
- **clicksToday / timeline**: bucket by **local** date, not UTC:
  `date(a.created_at,'${signedMinutes(off)}') = date('now','${signedMinutes(off)}')`.
- **CSV export timestamps**: `strftime('%Y-%m-%d %H:%M', a.created_at,'${signedMinutes(off)}')`, header
  labeled with the zone (`UTC+03:00`), so users in any country get consistent times.

## Plan gating
Single source of truth `config/plans.js`: each plan has `caps` (dateRange, detailedGeo, csvExport,
publicReports, uniqueClicks…) and `analyticsRanges: ['all','7d',…]`. Summary response returns
`caps` (incl. `analyticsRanges`) so the frontend gates UI. Backend **also** enforces (never trust the
client): an unauthorized range falls back to `'all'`; export gates columns + rows by plan.

## CSV export (plan-aware)
- Gate `csvExport`; compute columns array from `plan.caps.detailedGeo` (region/city/timezone/language
  only for the top tier); convert timestamps to the user's tz; `ORDER BY created_at DESC LIMIT
  <csvLimit>`; BOM (`﻿`) prefix for Excel UTF-8.
- **No links → 200 + header-only CSV**, not 404 (empty is a valid state).

## Frontend
- `RANGES` array of `{id,label}`; render all buttons, `disabled={!caps.analyticsRanges.includes(id)}`
  (show-but-disable, don't hide). Reset the selected range to `'all'` if the plan loses it.
- `buildParams()` → `{ range, tzOffset, linkId?, tag? }`; refetch on change.
- Reusable `ProgressList` card (label + horizontal gradient bar + value), grid
  `sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4` so cards stay compact.
- Recharts for timeline/device donut; hardcode hex colors (Recharts isn't Tailwind-aware).
- Public report: a token (`public_report_token` on user) → auth-less `/report/:token` route that
  calls `buildSummary` with `range:'all'` and returns only aggregate metrics (never raw IPs).

## Gotchas (real bugs found in review + live testing)
1. **clicksToday/timeline must use the selected tz**, or the "Today" counter is off by up to the
   offset near midnight while the `day` range is correct — an inconsistency users notice.
2. **Empty state ≠ error:** `buildScope` returns `null` for "no entities"; export must send an empty
   CSV (200), not 404. Keep this distinct from `{invalid:true}` (bad linkId → 404).
3. **Enforce caps on BOTH paths** — panel routes AND any public REST API. A cap checked only in the
   panel is bypassable via the API. When you add a plan capability, gate it everywhere it's writable.
4. String-built SQL is safe **only** if every interpolated value is a whitelisted constant or a
   clamped integer. Row data always goes through bound `?` params.

## Verification
- Boot the server against a temp DB (`DB_PATH`), seed rows with known `created_at` at both UTC and
  local midnight, assert each range + `off ∈ {0,180}` returns the right set (especially `day`).
- Assert empty-user export → 200 with header only; free vs pro export column sets differ.
