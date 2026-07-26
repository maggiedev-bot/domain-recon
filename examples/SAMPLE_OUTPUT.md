# Sample output — domain-recon

Real output captured against generic public targets (`example.com`, `8.8.8.8`,
`AS15169`). JSON is the default; `--human` gives the compact form shown here.

## `dns example.com --type A,MX --human`
```
DNS (google) — example.com
  A     104.20.23.154, 172.66.147.243
  MX    0 .
```

## `ip 8.8.8.8 --human`
```
ip-api — 8.8.8.8
  country: United States
  region: Virginia
  city: Ashburn
  isp: Google LLC
  org: Google Public DNS
  as: AS15169 Google LLC
  timezone: America/New_York
  flags: hosting
```

## `asn 8.8.8.8 --human`   (IP → owning ASN + prefix)
```
RIPEstat IP — prefix 8.8.8.0/24
  ASNs: 15169
```

## `asn AS15169 --human`   (ASN → holder + announced prefixes, capped in human mode)
```
RIPEstat ASN 15169 — GOOGLE - Google LLC
  announced: True
  prefixes (1409): 152.65.214.0/23, 142.251.165.0/24, 34.0.234.0/24, ... (+1394 more)
```

## `rdap 8.8.8.8`   (JSON, default)
```json
{
  "endAddress": "8.8.8.255",
  "entities": [{"handle": "GOGL", "roles": ["registrant"]}],
  "events": {
    "last changed": "2023-12-28T17:24:56-05:00",
    "registration": "2023-12-28T17:24:33-05:00"
  },
  "handle": "NET-8-8-8-0-2",
  "ipVersion": "v4",
  "kind": "ip",
  "name": "GOGL",
  "source": "rdap.org",
  "startAddress": "8.8.8.0",
  "status": ["active"],
  "type": "DIRECT ALLOCATION"
}
```

## `certs example.com --human`
```
crt.sh — example.com
  subdomains (N):
    example.com
    *.example.com
    www.example.com
    ...
  certificates: N
```
> Note: crt.sh is frequently rate-limited / slow under load. The helper retries
> with backoff; if it is temporarily down you'll get a clean `error:` on stderr
> and a non-zero exit, not a crash.

## `wayback example.com`
```json
{
  "archived": true,
  "snapshot_url": "http://web.archive.org/web/20260726083637/http://example.com/",
  "source": "wayback",
  "status": "200",
  "timestamp": "20260726083637",
  "url": "example.com"
}
```
