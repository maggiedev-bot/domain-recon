# Changelog / Design decisions — domain-recon

## 0.3.1 — bound the Wayback step so `profile` can't read as hung

Live QA (2026-07-26) ran the full surface against `groupvault.io` and
`kraken.com`. Both round-3 fixes (CT fallback + IANA/supplement RDAP bootstrap)
fired correctly on real infra, but the `profile` orchestrator on a large domain
(`kraken.com`, 90 subdomains) repeatedly ran **past 2 minutes** — long enough to
read as a hang.

### Fixed
- **`profile` now bounds the Wayback CDX step** (`run_profile` gains
  `wayback_timeout=8.0`, `wayback_retries=0`). Root cause: the profile passed its
  full `timeout=20, retries=3` budget to the Wayback step, and the Wayback CDX
  index is by far the slowest source (a single subdomain lookup was measured at
  **38 s**), so three archive targets could consume several minutes on their own.
  Wayback is the lowest-value part of a profile and is already fault-isolated, so
  it now runs on a tight single-attempt budget. Measured effect on `kraken.com`:
  **>120 s → ~78 s**; `groupvault.io` ~92 s, both completing cleanly (exit `0`).
  Documented the expected 1–1.5 min profile runtime in SKILL.md so it doesn't
  read as hung; `--no-wayback` / `--ip-limit` remain available to go faster.
- Regression test `test_wayback_step_runs_on_bounded_budget` locks the tight
  budget (offline suite 116 → 117 tests, all green).

## 0.3.0 — source-outage resilience

Both changes were driven by a live run that hit two real upstream failures at
once: crt.sh returning `HTTP 502` on every query (a hard outage), and rdap.org
`404`ing a `.io` domain (its bootstrap doesn't cover several ccTLDs/newer TLDs).

### Added
- **CT fallback (crt.sh → certSpotter)** — `certs` now tries crt.sh first and,
  on any failure (`5xx`/`4xx`/timeout/bad-JSON), falls back to **certSpotter**
  (`api.certspotter.com`, keyless) so subdomain enumeration survives a crt.sh
  outage. Names from every source that answered are merged + de-duplicated. The
  result records `sources_used` (which answered) and `errors[]` (which failed)
  for auditability, and a `200` with an empty set is treated as a genuine "no
  certs logged" answer (**not** an outage → no needless fallback). `--all-ct`
  queries both sources and merges for wider coverage. The `profile` orchestrator
  surfaces a partial CT failure under its own `errors[]` and reports `ct_sources`.
- **RDAP three-tier resolution** — `rdap` domain lookups now resolve the
  authoritative RDAP server via the **IANA bootstrap registry**
  (`data.iana.org/rdap/dns.json`, fetched once per run and memoized) for the
  ~1200 TLDs it lists (e.g. `.ai`/`.dev`/`.xyz`/`.uk`), a curated **supplement**
  for TLDs IANA omits, and rdap.org last. `rdap_source` records which tier
  answered (`iana-bootstrap` / `supplement` / `rdap.org`). IP/ASN lookups stay on
  rdap.org (complete RIR coverage). `--no-bootstrap` forces rdap.org.

### Decision (solo, flagged): certSpotter as the CT fallback
Candidates were certSpotter, Cloudflare merklemap, and crt.sh mirrors.
**certSpotter** was chosen: it is keyless, passive, has a clean documented JSON
schema (`/v1/issuances` with `expand=dns_names`), and covers the same CT ground.
merklemap and the mirrors were rejected as less stable / less documented for a
publish-ready skill. Kept stdlib-only. **Verified live** on 2026-07-26 during a
real crt.sh hard-`502` outage: `certs example.com` fell back to certSpotter and
returned subdomains, with the crt.sh `502` recorded in `errors[]`.

### Decision (solo, flagged): the IANA bootstrap alone does NOT fix `.io` — a supplement does
This is the important one. The obvious fix ("use the IANA bootstrap") turned out
to be **insufficient for the exact reported case**: a live pull of
`data.iana.org/rdap/dns.json` shows `.io` is **not in it at all** (nor are `.co`,
`.me`, `.us`). The bootstrap fixes many other TLDs (`.ai`, `.dev`, `.xyz`, `.uk`,
…), but not `.io`. Identity Digital *does* serve `.io` RDAP at
`rdap.identitydigital.services`, so the real fix is a small **curated supplement**
of TLD→RDAP-base for IANA-omitted TLDs, tried after the bootstrap and before
rdap.org. Current entries, each verified live (HTTP 200 for a real domain) on
2026-07-26: `.io`/`.sh`/`.ac` → Identity Digital, `.us` → `rdap.nic.us`.
`.co`/`.me` were left out (no public RDAP endpoint found on the obvious hosts) —
extend the map when one is confirmed. Caught only because the live smoke test
exercised the real `.io` path; the first cut (bootstrap-only) shipped green
offline but failed live.

### Decision (solo, flagged): domain-only resolution
The live failure was a `.io` **domain**; IP/ASN RDAP via rdap.org worked. IANA
also publishes `ipv4.json`/`ipv6.json`/`asn.json`, but adding them was out of
scope for the reported problem and would add fetch cost + fixtures for a path
that already works. Domains use the bootstrap+supplement; IP/ASN stay on rdap.org.

### Tests
- Offline suite expanded to **116 deterministic tests** (+24): certSpotter parse
  (scope/dedup/wildcards/error-object), CT fallback (crt.sh-down→certSpotter,
  both-down→clean error, empty-200-is-an-answer, `--all-ct` merge/dedup, profile
  surfacing); RDAP three-tier (bootstrap parse, `.ai` via bootstrap, `.io` via
  supplement, supplement-works-when-bootstrap-down, unmapped-TLD → rdap.org,
  authoritative-`404` → rdap.org, caching, `--no-bootstrap`, IP/ASN still via
  rdap.org). Live smoke: `certs` source-agnostic + `.io` (supplement) + `.ai`
  (bootstrap) — all verified passing live on 2026-07-26 under a real crt.sh
  outage.

## 0.2.0 — orchestration, archive history, hardening

### Added
- **`profile` subcommand** — a one-shot orchestrator that chains `certs → dns →
  ip → asn → wayback` for a domain into a single merged report. Deduplicates
  resolved IPs, isolates per-source failures (records them under `errors[]`
  instead of aborting), bounds each stage (`--cert-limit`/`--resolve-limit`/
  `--ip-limit`), and spaces `ip` enrichment calls to respect ip-api's 45/min.
- **Wayback CDX capture history** (`wayback`) — beyond the availability
  fast-path, the command now lists recent captures from the CDX API
  (`web.archive.org/cdx/search/cdx`), with `--cdx-limit`, optional `--cdx-count`
  (active-months scan), and `--no-cdx` to skip.
- **`certs` hardening** — `--limit` / `--max-certs` to cap subdomain and
  cert-history output on busy domains (full counts still reported), and richer
  per-cert fields (issuer, common name, validity window, serial, entry time).

### Docs
- Consolidated **"Rate limits & gotchas"** section in `API_NOTES.md` (crt.sh
  flakiness, ip-api HTTP-only + non-commercial free tier, RIPEstat prefix cost,
  CDX scan cost).
- `homepage` moved to `github.com/maggiedev-bot/domain-recon`; `metadata` block
  normalized (single-line JSON, `version`).

### Tests
- Offline suite expanded to **92 deterministic tests** (added profile
  orchestration: wiring/dedup, fault isolation, per-source failure, ip-limit
  truncation, rate-limit spacing, human render; plus CDX parse/fetch paths).

## 0.1.0 — initial build

### Sources
Five keyless sources plus a bonus, each a subcommand of `scripts/recon.py`:
`certs` (crt.sh), `rdap` (rdap.org), `dns` (Google/Cloudflare DoH), `ip`
(ip-api.com), `asn` (RIPEstat), and `wayback` (archive.org, bonus).

### Decision: BGPView → RIPEstat Data API
The original spec named **BGPView** (`api.bgpview.io`) for ASN/prefix/IP
ownership. During the build, `api.bgpview.io` was found to be **NXDOMAIN**, and
research confirmed BGPView was **permanently shut down (~November 2025, after
acquisition by Recorded Future)**. Rather than ship a dead dependency or drop the
capability, the `asn` subcommand was pointed at the **RIPEstat Data API**
(`stat.ripe.net`) — a free, keyless, well-maintained RIPE NCC service that covers
the same ASN / prefix / IP-ownership ground:
- ASN lookup → `as-overview` (holder, announced) + `announced-prefixes`.
- IP lookup → `network-info` (announcing ASN(s) + covering prefix).

This keeps the skill at five sources and preserves the intended capability.

### Design choices
- **Zero third-party runtime deps** — standard-library `urllib`/`json` only, so
  the skill gates on nothing but `python3`. `pytest` is a *tests-only* dependency.
- **Pure parse functions** (`parse_*`) split from **fetch functions** (`fetch_*`)
  so normalization is unit-testable offline against captured fixtures.
- **Single HTTP seam** (`_open` / `http_get_json`) so tests can deterministically
  simulate 429s, 5xx, timeouts, DNS failures, and malformed JSON without a network.
- **Safety:** inputs validated and IDNA/punycode-encoded; passed only as URL-
  encoded parameters; no shell is ever invoked. Descriptive User-Agent, per-
  request timeouts, and exponential backoff honoring `Retry-After`.
