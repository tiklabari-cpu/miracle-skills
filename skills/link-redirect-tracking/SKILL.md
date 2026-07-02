---
name: link-redirect-tracking
description: URL-shortener redirect engine with click tracking and BTAG (affiliate/banner) attribution. Covers the ordered redirect pipeline, async click logging (geoip + UA), the denormalized-counter ↔ analytics-table relationship, and the clean-URL BTAG two-hop loop that attributes clicks without polluting the short link. Use when building a link shortener / redirector that records analytics and per-source (affiliate) click attribution. Pairs with the analytics-panel skill (this produces the data, that reports it).
---

# Link Redirect Engine + Click Tracking + BTAG Attribution

The **write/ingestion** side of a link shortener: `GET /:code` resolves and redirects, records one
analytics event per click, and attributes clicks to a BTAG (affiliate/banner source). Node/Express +
`better-sqlite3`. The reporting/query side lives in the **analytics-panel** skill — they share the
`analytics` table.

## The three tables and how they connect
- **links**: the short link. Has `code`, `target_url`, and a denormalized **`clicks`** counter (fast
  totals without scanning), plus feature columns (`password`, `expire_at`, `cloak`, `redirect_code`,
  `forward_params`, `mobile_targets`, `geo_targets`, `pixels`, and **`btag`** — a BTAG *embedded* in
  the link).
- **analytics**: one row **per click** (the detailed event log). Enriched with geo/device/browser/os/
  language/referrer and a **`btag`** column (the source that drove this click). UTC `created_at`.
- **btags**: user-registered BTAG values (`value`, `label`). Reporting cross-references
  `analytics.btag` against this list.

**The connection:** every logged click does two writes in one transaction — INSERT into `analytics`
(with its `btag`) **and** `UPDATE links SET clicks = clicks + 1`. So `links.clicks` = fast total,
`analytics` = the breakdown, and `analytics.btag` is what ties a click to its affiliate source.

## Redirect pipeline order (this order matters)
`GET /:code`:
1. **Extract code + capture incoming BTAG** from the raw URL (`?BTAG=`/`&BTAG=`, case-insensitive)
   via `req.originalUrl.match(/[?&]btag=([^&#]+)/i)` — not `req.query` (see BTAG loop).
2. **Not found → 404**, **expired (`expire_at`) → 410**.
3. **BTAG loop hop-1** (see below).
4. **Monthly click limit** (owner's plan) → 429. Gate *before* doing work.
5. **Password** → password form (the form preserves BTAG in a hidden field so the loop survives).
6. **Targeting:** geo (country) > mobile (Android/iOS) > default `target_url`.
7. **Param forwarding** (`forward_params`): copy incoming query onto the target, **minus `pw` and any
   `btag`** (BTAG is appended explicitly next).
8. **Append BTAG** to the target URL (so the destination/affiliate sees it).
9. **Output:** cloaking (full-page iframe) > pixel interstitial (FB/Google/TikTok) > normal
   `res.redirect(redirect_code)`.
10. **Async analytics logging** (after the response — see below).

## BTAG two-hop loop (the clever bit — keeps the short link clean)
Goal: attribute a click to `link.btag` **and** pass it to the destination, without the BTAG appearing
in the short URL the user shares.
```js
// hop-1: link has an embedded btag but the request has none yet → bounce to self WITH the btag.
if (link.btag && !btag) {
  const qi = req.originalUrl.indexOf('?');
  const usp = new URLSearchParams(qi >= 0 ? req.originalUrl.slice(qi + 1) : '');
  usp.delete('pw');                                   // don't carry the password param forward
  for (const k of [...usp.keys()]) if (/^btag$/i.test(k)) usp.delete(k);
  usp.append('BTAG', link.btag);
  return res.redirect(302, `/${encodeURIComponent(code)}?${usp.toString()}`);   // NO logging here
}
// hop-2: request now carries BTAG → it's appended to the target AND logged. One click, counted once.
```
- **hop-1 does NOT log** → no double counting.
- BTAG can also arrive from an **external** referrer's URL (a banner links to `/<code>?BTAG=partnerX`).
  Either source (embedded or external) ends up in `analytics.btag`.

## Async click logging (never add latency to the redirect)
Log **after** the response using `setImmediate` so the redirect isn't blocked:
```js
function logClickAsync({ linkId, ip, userAgent, referrer, acceptLanguage, btag }) {
  setImmediate(() => {
    try {
      const ua = new UAParser(userAgent || '');
      const device = ua.getDevice().type ? cap(ua.getDevice().type) : 'Desktop';
      const cleanIp = (ip || '').replace('::ffff:', '').split(',')[0].trim();
      const isLocal = !cleanIp || cleanIp === '::1' || cleanIp.startsWith('127.')
        || cleanIp.startsWith('10.') || cleanIp.startsWith('192.168.')
        || /^172\.(1[6-9]|2\d|3[0-1])\./.test(cleanIp);
      const geo = isLocal ? null : geoip.lookup(cleanIp);         // geoip-lite: local DB, no API
      const country = isLocal ? 'Yerel' : geo ? geo.country : 'Bilinmiyor';
      const tx = db.transaction(() => {
        db.prepare(`INSERT INTO analytics (link_id, ip, country, city, region, timezone,
          language, device, browser, os, referrer, btag) VALUES (?,?,?,?,?,?,?,?,?,?,?,?)`).run(/*…*/);
        db.prepare('UPDATE links SET clicks = clicks + 1 WHERE id = ?').run(linkId);
      });
      tx();
    } catch (e) { console.error('log', e.message); }   // logging failure must not affect redirects
  });
}
```
- **geoip-lite** = local IP→country/city DB, no external API, no latency.
- Private/LAN IPs → label `'Yerel'` (so localhost dev still shows non-empty analytics).
- INSERT + counter bump are in **one transaction** so they never drift apart.

## BTAG management + "discovered" BTAGs
The BTAG page shows registered BTAGs with click counts **and** BTAGs seen in traffic but never
registered (so users discover which affiliate codes are actually driving clicks):
```sql
SELECT a.btag AS value, COUNT(*) AS clicks
FROM analytics a JOIN links l ON l.id = a.link_id
WHERE l.user_id = ? AND a.btag IS NOT NULL AND a.btag != ''
GROUP BY a.btag
```
Registered = from the `btags` table (join counts by value). **Discovered** = keys in the count map
not present in the registered set, sorted by clicks desc.

## Unique vs repeat clicks
Unique visitors = `COUNT(DISTINCT ip)`; repeat = `total - unique`. (A cap like `uniqueClicks` can gate
this per plan.)

## Gotchas
1. **hop-1 must never log** or every BTAG click double-counts.
2. **Build hop-1's redirect from `req.originalUrl`, not `{...req.query}`** — Express turns repeated
   params (`utm=a&utm=b`) into arrays and `URLSearchParams({utm:['a','b']})` mangles them to `utm=a,b`.
   Also strip `pw` so the password never leaks into the rewritten URL.
3. **`forward_params` must exclude `pw` and `btag`** — pw is a secret, btag is appended explicitly.
4. **Log asynchronously** (`setImmediate`, after the response). A synchronous insert on the hot
   redirect path adds latency under load. Wrap in try/catch so a logging error never breaks a redirect.
5. **Define `/:code` AFTER `/api/*` routes** so the catch-all short-code route doesn't shadow the API.
6. **Enforce the monthly click limit before doing work** (targeting/logging), returning 429.

## Verification
- Boot on a temp `DB_PATH`. Request a link with an embedded `btag` and assert: first response is a
  302 to `/<code>?BTAG=...` with **no** analytics row; the followed request logs exactly one row with
  that btag and bumps `links.clicks` by 1.
- Request with repeated query params (`?utm=a&utm=b`) on a btag link → both survive hop-1 (not
  collapsed to `a,b`); `pw` is stripped.
- External BTAG (`?BTAG=partnerX` on a link with no embedded btag) → logged once, appears under
  "discovered".
