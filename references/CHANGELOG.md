# Changelog / Design decisions — domain-recon

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
