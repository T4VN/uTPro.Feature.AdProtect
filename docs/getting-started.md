> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.AdProtect/) is the canonical, always-current home of this page.

# Getting started

AdProtect decides per visitor whether your ad code is served at all, and keeps a
click-intent log with automatic cooldowns. The package self-wires: installing the
NuGet package is enough â€” no `Program.cs` changes.

## Install

    dotnet add package uTPro.Feature.AdProtect

This pulls in `uTPro.Feature.GeoLocation` (client-IP + country resolution).

## First boot: tables + backoffice

1. The four tables are created automatically on first start; optionally the built-in
   datacenter-ASN rules are seeded (disabled) when `AdProtect:SeedDatacenterAsnRule` is true.
2. Open the backoffice -> **Settings -> AdProtect**. Five tabs: Dashboard, Rules,
   Blocks, Click log, Settings.

## Show an ad slot

1. Add the tag helper to `Views/_ViewImports.cshtml`:

       @addTagHelper *, uTPro.Feature.AdProtect
       @using uTPro.Feature.AdProtect.TagHelpers
       @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

2. Replace your ad markup with the slot:

       <utc-ad slot="banner-top" render-ins />

3. Set the AdSense client id in **Settings** or per-slot:

       <utc-ad slot="banner-top" client-id="ca-pub-0000000000000000" ad-slot="123123123" />

## Load the AdSense script

AdProtect does not load the AdSense library itself. Keep your existing script tag for
`adsbygoogle.js` in the page head; the slot JS only pushes to `adsbygoogle` after the
gate allows the visitor.

## How the gate works

- The slot renders **empty** (cache-safe). `adprotect.js` calls
  `GET /api/adprotect/check?slot=...`.
- Allowed -> the response carries a one-use token; the ad is appended and the click-intent
  tracker arms on `mouseenter` (3 s decay). A `window blur` / tab-hide while armed posts
  `POST /click {slot, token}` (fire-and-forget, 5 s de-bounce, one token = one click).
- Denied -> the slot collapses, or shows the placeholder when the slot sets `placeholder`
  and Settings has Placeholder HTML.

Every visitor gets a functional anonymous cookie (`utpro_ap_id`) from the check
endpoint; budgets count per-IP **and** per-cookie, so CGNAT mobile users are not
collateral damage (default: 10 clicks/IP/day, 3 clicks/cookie/day, 24 h cooldown).

## Next steps

- [Configuration](configuration.md) â€” all knobs, live settings, ASN TSV data source
- [Integration with uTPro](integration-utpro.md) â€” multi-site + static front-end setup
