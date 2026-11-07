> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.AdProtect/) is the canonical, always-current home of this page.

# Configuration

## appsettings.json (deploy-time only)

```json
"AdProtect": {
  "ApiPrefix": "/api/adprotect",
  "SeedDatacenterAsnRule": false
}
```

| Key | Default | Meaning |
|---|---|---|
| `ApiPrefix` | `/api/adprotect` | Public endpoint prefix for check/click. Kept outside /umbraco/ so static front-ends can point at it. |
| `SeedDatacenterAsnRule` | `false` | On first boot, insert the built-in datacenter-ASN block rules (disabled). Enable once, then manage in Rules. |

Everything else lives in the database and is edited live from **Settings -> AdProtect ->
Settings** — effective on the next request, no restart, no redeploy.

## Live settings (backoffice)

Scoped **Global** or per-host:

| Field | Default | Meaning |
|---|---|---|
| Max clicks per IP per day | 10 | Budget before an automatic cooldown |
| Max clicks per cookie per day | 3 | The primary budget (CGNAT-aware) |
| Cooldown hours | 24 | How long an over-budget visitor stops seeing ads |
| AdSense client id | — | Publisher id used by `<utc-ad>` |
| Placeholder HTML | — | Optional HTML shown in denied slots that opted in (`placeholder` attribute) |
| Store hashed IPs | off | Store IPs as keyed HMAC-SHA256 (still matchable, GDPR-friendlier). Toggling makes old rows unmatchable. |

## Rules

Rule evaluation is **first match wins**, ordered by Priority (lower first), then Id:

1. Runtime cooldown (active blocks) -> deny
2. Allow rule match -> allow
3. Block rule match -> deny
4. Click budget exceeded -> deny
5. Default -> allow

Types: `Ip` (single address), `Cidr` (`10.0.0.0/8`, IPv4/IPv6), `Country` (ISO-2 from
GeoLocation), `Asn` (numeric). Rules can target one host or all, can be disabled without
deleting, and can carry an expiry.

### ASN data source

By default ASN rules match only the `Cf-Asn` edge header. For offline matching drop an
[iptoasn.com](https://iptoasn.com) combined TSV at:

    App_Data/AdProtect/ip2asn.tsv

(range_start, range_end, AS_number, country, description — tab-separated; IPv4 ranges;
IPv6 rows are skipped.) The file is loaded lazily on first use; replace the file and
recycle the app to refresh.

## Privacy notes

- The anonymous cookie `utpro_ap_id` is functional (anti-abuse), HttpOnly, 1 year.
- `POST /click` accepts only HMAC-bound one-use tokens issued by `/check` for that exact
  (cookie, host, slot) — no client-supplied IPs, no replay.
- Public endpoints answer 200 with minimal bodies; deny reasons never leave the server.
