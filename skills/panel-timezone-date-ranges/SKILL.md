---
name: panel-timezone-date-ranges
description: A user-selectable panel timezone (Local/Istanbul header clock + toggle) that drives timezone-consistent analytics date-range filters (All / Last Hour / Last Day / Last 24h / Last 7·30·90 days) and CSV export. One UTC offset flows from the header preference through every date query and the export, so users in different countries see consistent "today"/"day" boundaries and timestamps. Use when a dashboard needs a switchable timezone plus calendar-aware date filters and exports. Deep-dive companion to the analytics-panel skill.
---

# Panel Timezone + Date Ranges + CSV Export (offset-driven)

One coherent mechanism: a header **timezone toggle** produces a UTC **offset (minutes)** that flows
through every analytics date query and the CSV export. Result: "Last Day", "today", and exported
timestamps are correct for whatever zone the user picked — no IANA tz database on the server, no
per-country drift. Node/Express + `better-sqlite3` + React.

## Why offset-minutes (not zone names)
The server stays generic and stateless per request: it never needs a tz database. The client computes
the effective offset from the user's preference and sends it as `tzOffset` on each request. Only the
**calendar-day** boundary and timestamp *display* need it; rolling windows (last N hours/days) are
timezone-independent.

## 1. Preference storage + header toggle
- **Column:** `users.panel_tz TEXT NOT NULL DEFAULT 'local'` — values `'local' | 'istanbul'`. Include
  it in **every** user payload (`/me`, login, register), same as `plan`.
- **Endpoint:** `PUT /auth/timezone { tz }` → validate `∈ {local, istanbul}`, update, return it.
- **Frontend util** (`utils/timezone.js`):
  ```js
  export const TZ_LABEL = { local: 'Yerel', istanbul: 'İstanbul' };
  export function getTzOffsetMinutes(panelTz) {        // minutes east of UTC
    if (panelTz === 'istanbul') return 180;            // UTC+3 fixed (see gotcha)
    return -new Date().getTimezoneOffset();            // browser local, DST-aware
  }
  export function formatClock(panelTz) {               // 24h HH:MM, no seconds
    const o = { hour: '2-digit', minute: '2-digit', hour12: false };
    if (panelTz === 'istanbul') o.timeZone = 'Europe/Istanbul';
    return new Date().toLocaleTimeString('tr-TR', o);
  }
  ```
- **Header** (right of the theme button): a live clock (`setInterval(1000)` → `formatClock`) + a
  two-option `[Yerel | İstanbul]` segmented toggle. On change: **confirm first**, then persist, then
  update context so the clock and all analytics refetch with the new offset:
  ```js
  const choose = async (next) => {
    if (next === tz) return;
    if (!window.confirm(`Saat diliminizi ${TZ_LABEL[next]} olarak değiştiriyorsunuz, emin misiniz?`)) return;
    try { await api.put('/auth/timezone', { tz: next }); setUserLocal({ ...user, panel_tz: next }); }
    catch { alert('Saat dilimi güncellenemedi.'); }
  };
  ```

## 2. The date ranges (enum → SQLite lower-bound, injection-safe)
Seven ranges. `off` = `tzOffset` in minutes (only `day` uses it).
| key | label | SQLite lower bound |
|-----|-------|--------------------|
| `all` | Tümü | (none — only the retention window applies) |
| `1h` | Son Saat | `datetime('now','-1 hours')` |
| `day` | Son Gün | `datetime('now','<+off> minutes','start of day','<-off> minutes')` |
| `24h` | Son 24 Saat | `datetime('now','-24 hours')` |
| `7d` | Son 7 Gün | `datetime('now','-7 days')` |
| `30d` | Son 30 Gün | `datetime('now','-30 days')` |
| `90d` | Son 90 Gün | `datetime('now','-90 days')` |
```js
const signedMinutes = (m) => `${m >= 0 ? '+' : ''}${m} minutes`;
function rangeLowerBound(range, off) {
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
**"Son Gün" = the selected zone's midnight → now.** `+off minutes` shifts UTC-now into the local
wall clock, `start of day` truncates to local midnight, `-off minutes` shifts that instant back to
UTC for comparison against the UTC `created_at`. `off` is parsed to an integer and clamped
`[-720,840]`, so the interpolated string is a fixed whitelist → **no SQL injection**.

Also apply `off` to **clicksToday** and the **timeline** grouping, or they disagree with `day`:
`date(a.created_at,'${signedMinutes(off)}') = date('now','${signedMinutes(off)}')`.

## 3. The bridge (frontend → backend)
```js
const off = getTzOffsetMinutes(user.panel_tz);
// every request:
api.get('/analytics/summary', { params: { range, tzOffset: off, linkId, tag } });
api.get('/analytics/export',  { params: { range, tzOffset: off, linkId, tag }, responseType: 'blob' });
```
Changing the header timezone re-renders (context), which changes `off`, which refetches — so the
whole dashboard and any export instantly reflect the new zone.

**Plan gating:** each plan has `analyticsRanges: ['all','7d',…]`. Render all 7 buttons, disable those
not in the plan (`disabled={!allowed.includes(id)}` — show-but-disable). Backend re-validates: an
unauthorized range falls back to `'all'` (never trust the client).

## 4. CSV export uses the same offset
```js
const off = s ? s.off : normOffset(req.query.tzOffset);
const sign = off >= 0 ? '+' : '-', abs = Math.abs(off);
const tzLabel = `UTC${sign}${String((abs/60|0)).padStart(2,'0')}:${String(abs%60).padStart(2,'0')}`;
const dateExpr = `strftime('%Y-%m-%d %H:%M', a.created_at, '${signedMinutes(off)}')`; // local wall time
```
- **Range filter:** same plan-gated range (identical `buildScope`).
- **Timestamps converted** to the user's zone; the "Tarih" header is labeled with the zone
  (`Tarih (UTC+03:00)`) so a user in any country reads consistent times.
- **Columns plan-gated** (detailed geo only on the top tier); `ORDER BY created_at DESC LIMIT
  <csvLimit>`; BOM prefix for Excel.
- **No links → 200 + header-only CSV**, not 404.

## Gotchas
1. **Istanbul = fixed UTC+3.** Turkey dropped DST in 2016, so the `+180` constant is safe and
   correct year-round. Do NOT compute it from the browser.
2. **"Local" must send the browser offset per request** (`-getTimezoneOffset()`), because it is
   DST-aware and device-specific — never store a hardcoded local offset server-side.
3. **Rolling ranges are tz-independent** — only `day` (and timestamp display + clicksToday/timeline)
   use `off`. Don't tz-shift `24h`/`7d` etc.
4. **clicksToday & timeline must use `off` too**, or the "Today" counter disagrees with the `day`
   range near midnight (off by up to the offset).
5. **String-built SQL is safe only** because `off` is an integer clamped to a range; all row values
   still go through bound `?` params.

## Verification
Seed analytics rows at known instants (e.g. yesterday 23:00 and today 06:00 Istanbul, stored UTC).
Assert `day` with `off=180` (Istanbul midnight) and `off=0` (UTC midnight) return different sets;
assert export timestamps shift with the offset and the header label matches; assert an unauthorized
range for a Free plan falls back to `all` server-side.
