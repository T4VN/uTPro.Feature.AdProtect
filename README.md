# uTPro AdProtect for Umbraco

Invalid-click protection and ad-serving control for Umbraco sites monetized with
Google AdSense (or any ad network). Think "Ad Invalid Click Protector" (AICP) for
WordPress — rebuilt native for Umbraco/.NET, cache-safe and multi-site aware.

Works with **Umbraco 17 and 18**.

**Documentation:** <https://docs.utpro.dev/uTPro.Feature.AdProtect/>

## Why

AdSense accounts get "ad serving limits" for *invalid traffic* — bots, scanners and
a handful of accidental or abusive clicks. On small sites a few anomalous clicks are
enough to trigger a limit, and repeated flags risk account termination.

AdProtect decides **per visitor, per request** whether ad code should be served at
all, and logs ad-click intent so repeat clickers stop seeing ads.

## Features

- **Visitor gate** — hide ads for:
  - blocked IPs / CIDR ranges (manual list + bulk import)
  - blocked countries (reuses `uTPro.Feature.GeoLocation`)
  - datacenter/hosting ASNs (built-in list, disabled by default — enable with one click)
  - visitors who exceeded the click budget (auto cool-down, durable across restarts)
- **Click-intent tracker** — lightweight vanilla JS (~2 KB, no jQuery) using the
  `mouseenter` + `window blur` / `visibilitychange` heuristic on the ad container;
  click events are HMAC-token bound, de-duplicated and never accepted from
  client-supplied IPs
- **Cache-safe by design** — the Razor tag renders an empty slot; a tiny fetch to
  `/api/adprotect/check` decides whether the ad script loads. Works with output
  cache, works on any host (including static front-ends pointing at the API)
- **Backoffice workspace** (under Settings) — dashboard (serves / blocks / top IPs /
  14-day trend), rules CRUD, active blocks with manual unban, click log, live
  settings — no restart needed
- **CGNAT-aware** — budgets counted per anonymous cookie *and* per IP, so mobile
  users behind carrier-grade NAT (common in VN) are not collateral damage
- **Privacy option** — store IPs as keyed hashes (HMAC-SHA256), still matchable

## Install

    dotnet add package uTPro.Feature.AdProtect

The package self-wires: the database tables are created on first boot, the
**Settings → AdProtect** workspace appears in the backoffice, and the gate
endpoints register themselves — no `Program.cs` changes.

## Show an ad slot

Register the tag helper in `Views/_ViewImports.cshtml`:

    @addTagHelper *, uTPro.Feature.AdProtect

Replace your ad markup with:

    <utc-ad slot="banner-top" render-ins />

Set your AdSense client id in **Settings → AdProtect → Settings** (or per slot with
`client-id="ca-pub-..."`). Keep your existing `adsbygoogle.js` script tag in the page
head — AdProtect only pushes to `adsbygoogle` after the gate allows the visitor.

Requires `uTPro.Feature.GeoLocation` (installed automatically) for client-IP and
country resolution.

## Links

- Documentation: <https://docs.utpro.dev/uTPro.Feature.AdProtect/>
- Issue tracker: <https://github.com/T4VN/uTPro.Feature.AdProtect/issues>

---

&copy; uTPro. Free (proprietary) license — same as other uTPro feature packages.
