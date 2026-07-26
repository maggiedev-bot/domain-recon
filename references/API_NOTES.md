# API Notes — domain-recon sources

All sources below are **keyless** (no API key, no account, no auth header). Each
row lists the exact endpoint the helper calls and the fields it normalizes.

## crt.sh — Certificate Transparency (`certs`)
- Endpoint: `https://crt.sh/?q=%25.<domain>&output=json` (`%25` = URL-encoded `%`, the CT wildcard).
- Returns a JSON array of certificate rows. `name_value` is **newline-delimited**
  and may contain wildcard (`*.`) SANs; the helper splits, lowercases, strips
  trailing dots, keeps only names within the queried apex, and de-duplicates.
- Quirk: on **zero results** crt.sh returns an *empty body* (not `[]`) — handled.
- Quirk: crt.sh is frequently slow or returns `502`/`503` under load. The helper
  retries; the live smoke test treats a 5xx as a skip, not a failure.

## rdap.org — RDAP / modern WHOIS (`rdap`)
- Endpoints: `https://rdap.org/domain/<d>`, `/ip/<ip>`, `/autnum/<asn>`.
- `rdap.org` is a **redirector** — it 3xx-redirects to the authoritative RDAP
  server (Verisign, ARIN, RIPE, …). `urllib` follows redirects automatically.
- Normalized fields: `handle`, `status[]`, `events{action:date}`, `entities`,
  plus per-kind fields (domain: `ldhName`, `nameservers`, `dnssec`; ip:
  `startAddress`/`endAddress`/`country`; asn: `startAutnum`/`endAutnum`).

## DNS-over-HTTPS — Google + Cloudflare (`dns`)
- Google: `https://dns.google/resolve?name=<n>&type=<T>`.
- Cloudflare: `https://cloudflare-dns.com/dns-query?name=<n>&type=<T>` with
  `Accept: application/dns-json`.
- Supported types: A, AAAA, MX, TXT, NS, CNAME, SOA, CAA.
- Response `Status` is an RCODE (0=NOERROR, 3=NXDOMAIN, …); answers carry numeric
  `type` codes which the helper maps back to mnemonics.

## ip-api.com — IP geolocation / ISP (`ip`)
- Endpoint: `http://ip-api.com/json/<ip>?fields=...` (free tier is HTTP-only).
- **Rate limit: 45 requests/minute** per source IP — respect it (batch, space out).
- API-level failures come back as `{"status":"fail","message":...}` (e.g. reserved
  ranges) — the helper raises a clean error rather than returning junk.

## RIPEstat Data API — ASN / prefix / IP ownership (`asn`)
- ASN: `https://stat.ripe.net/data/as-overview/data.json?resource=AS<n>` plus
  `.../announced-prefixes/data.json?resource=AS<n>`.
- IP: `https://stat.ripe.net/data/network-info/data.json?resource=<ip>` →
  announcing ASN(s) + covering prefix.
- Free, keyless, well-maintained by RIPE NCC. **This replaces BGPView**, which was
  permanently shut down in November 2025 (see `CHANGELOG.md`).

## archive.org Wayback — availability + CDX (`wayback`, bonus)
- Availability fast-path: `http://archive.org/wayback/available?url=<u>` — returns
  the closest snapshot (if any) under `archived_snapshots.closest`.
- Capture history: `https://web.archive.org/cdx/search/cdx?url=<u>&output=json`
  (array-of-arrays; first row is the header). The helper lists the most recent
  captures (`--cdx-limit`, default 10) and can compute active-months with
  `--cdx-count` (an extra, slower scan). `--no-cdx` skips this step entirely.
- Quirk: the CDX endpoint returns an *empty body* for a never-archived URL — the
  parser treats a missing/header-only response as "no captures", not an error.

## Rate limits & gotchas
A consolidated cheat-sheet of the per-source caveats to be a good citizen:

- **ip-api.com** — free tier is **HTTP-only** (no HTTPS) and **non-commercial use
  only**; commercial use requires the paid/keyed plan. Hard limit **45 requests /
  minute** per source IP (HTTP 429 on breach). The `profile` orchestrator spaces
  its `ip` enrichment calls to stay under this; if you script `ip` in a loop, add
  your own delay.
- **crt.sh** — frequently slow and prone to `502`/`503` under load; large domains
  can return very large responses. The helper retries with backoff, and the live
  smoke test treats a 5xx as a *skip* (transient) rather than a failure (schema
  drift). Use `certs --limit` / `--max-certs` to cap output on busy domains.
- **rdap.org** — a redirector; the authoritative server it forwards to sets its own
  (usually generous) limits. Redirects are followed automatically.
- **DoH (Google / Cloudflare)** — generous public limits; the `dns` command lets
  you switch providers with `--provider` if one is degraded.
- **RIPEstat** — free and well-maintained, but a heavy `announced-prefixes` pull is
  the slow part of an ASN lookup; use `asn --no-prefixes` when you only need the
  holder. This is the **BGPView replacement** (BGPView shut down Nov 2025).
- **Wayback CDX** — `--cdx-count` walks the full capture index and can be slow for
  heavily-archived sites; the default lists only the most recent captures.
- **General courtesy** — every request carries a descriptive `User-Agent`, uses a
  per-request timeout, and backs off on `429`/`5xx` honoring an integer
  `Retry-After`. Nothing here probes the target directly — all lookups are passive
  queries against third-party databases.
