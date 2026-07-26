---
name: domain-recon
description: Passive domain/infra OSINT over five keyless public APIs — subdomains, RDAP/WHOIS, DNS-over-HTTPS, IP geo/ISP, and ASN/prefix ownership.
metadata: {"openclaw": {"requires": {"bins": ["python3"]}, "emoji": "🛰️"}, "homepage": "https://github.com/maggiedev-bot/domain-recon", "version": "0.2.0"}
---

# domain-recon

Passive reconnaissance for a domain, IP, or ASN using **five keyless public
APIs** — no API keys, no accounts, no secrets. Everything is *passive*: the skill
only queries public third-party databases; it never connects to or probes the
target host directly.

Use this when the user wants to investigate a domain or IP's public footprint:
enumerate subdomains, look up WHOIS/RDAP registration, resolve DNS records, geo-
locate an IP, or identify which network/ASN owns an address.

All work runs through one Python helper (standard library only — no `pip install`):

```bash
python3 {baseDir}/scripts/recon.py <subcommand> <target> [options]
```

Output is JSON by default (easy to parse and chain); add `--human` for a compact
readable summary.

## Subcommands

| Subcommand | Source | What it does |
|------------|--------|--------------|
| `certs <domain>`   | crt.sh (Certificate Transparency) | Enumerate subdomains + certificate history |
| `rdap <resource>`  | rdap.org | Modern WHOIS for a domain, IP, or ASN (auto-detected) |
| `dns <name>`       | Google / Cloudflare DoH | Resolve DNS records (A, AAAA, MX, TXT, NS, CNAME, SOA, CAA) |
| `ip <ip>`          | ip-api.com | IP → geo, ISP, ASN, and proxy/hosting flags |
| `asn <resource>`   | RIPEstat Data API | ASN → holder + announced prefixes; IP → owning ASN + prefix |
| `wayback <url>`    | archive.org | Wayback snapshot availability + CDX capture history |
| `profile <domain>` | all of the above | One-shot orchestration: certs → dns → ip → asn → wayback into a single report |

### Common options

- `--human` — readable text instead of JSON.
- `--timeout <sec>` — per-request timeout (default 20).
- `--retries <n>` — retry attempts on 429/5xx/network errors (default 3).
- `certs --no-wildcards` — drop `*.` wildcard subdomains.
- `dns --type A,MX,TXT` — comma-separated record types in one call.
- `dns --provider cloudflare` — use Cloudflare DoH instead of Google (default).
- `rdap --kind domain|ip|asn` — force the resource kind instead of auto-detecting.
- `asn --no-prefixes` — skip the announced-prefixes list for a faster ASN lookup.
- `certs --limit <n>` / `certs --max-certs <n>` — cap the returned subdomains / cert-history rows (full counts still reported).
- `wayback --no-cdx` — availability only; `wayback --cdx-limit <n>` — number of recent captures to list.
- `profile --cert-limit <n>` / `--resolve-limit <n>` / `--ip-limit <n>` — bound how much of each stage the orchestrator pulls; `--no-wayback` to skip the archive step.

## Example invocations

```bash
# Subdomains + cert history for a domain
python3 {baseDir}/scripts/recon.py certs example.com --human

# WHOIS/RDAP (auto-detects domain vs IP vs ASN)
python3 {baseDir}/scripts/recon.py rdap example.com
python3 {baseDir}/scripts/recon.py rdap 8.8.8.8
python3 {baseDir}/scripts/recon.py rdap AS15169

# DNS records — several types at once, via Cloudflare
python3 {baseDir}/scripts/recon.py dns example.com --type A,AAAA,MX,TXT --provider cloudflare

# IP geolocation / ISP / hosting flags
python3 {baseDir}/scripts/recon.py ip 8.8.8.8 --human

# Who owns this network? (ASN or IP)
python3 {baseDir}/scripts/recon.py asn AS15169
python3 {baseDir}/scripts/recon.py asn 8.8.8.8

# Is it archived? (availability + recent capture history)
python3 {baseDir}/scripts/recon.py wayback example.com

# One-shot: profile a domain across every source in a single report
python3 {baseDir}/scripts/recon.py profile example.com --human
```

A typical "profile this domain" flow: run `certs` for subdomains, `rdap` for
registration, `dns` for the live records, then `ip` + `asn` on the resolved
address to see who hosts it. The `profile` subcommand chains exactly this
sequence for you (respecting ip-api's rate limit between enrichment calls) and
returns one merged JSON/`--human` report.

## Behavior & safety notes

- **Passive only.** No port scans, no direct connections to the target — just
  public database lookups.
- **Keyless.** No credentials are ever required, read, or transmitted.
- **No shell injection surface.** Inputs are validated (domains IDNA/punycode-
  encoded, IPs and ASNs parsed) and passed only as URL-encoded query parameters;
  the helper never invokes a shell.
- **Courteous.** Sends a descriptive User-Agent, uses per-request timeouts, and
  backs off with retries on `429`/`5xx` (honoring `Retry-After`). ip-api.com is
  rate-limited to 45 requests/minute — batch and space out calls.
- **Exit codes:** `0` success, `2` on a handled error (bad input, upstream
  failure) with a message on stderr.

## For maintainers — running the tests

The parsing/normalization logic is covered by an offline, deterministic pytest
suite (fixtures in `scripts/tests/fixtures/`, HTTP mocked — no network):

```bash
python3 -m pip install pytest        # only dependency, tests-only
python3 -m pytest {baseDir}/scripts/tests/test_recon.py -q
```

An opt-in live smoke test verifies the real endpoints still match the expected
schema (never part of the normal gate):

```bash
RECON_LIVE=1 python3 -m pytest {baseDir}/scripts/tests/test_live_smoke.py -q
```

See `references/API_NOTES.md` for endpoint details and `references/CHANGELOG.md`
for notable decisions (including the BGPView → RIPEstat swap).
