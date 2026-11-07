> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.AdProtect/) is the canonical, always-current home of this page.

# Integration with uTPro

AdProtect is the ad-revenue protection layer of the uTPro feature suite. On a uTPro site
it needs no extra wiring — the same composer mechanism that registers other features
registers AdProtect.

## Multi-site behavior

- **Rules** carry a `Host` column: empty = every site, otherwise the exact host
  (`foto.t4vn.com`). Use it to block a scanner subnet only on the site it hits.
- **Settings** are scoped Global + per-host. A per-host row overrides the global row for
  that host only (click budgets, placeholder, AdSense client id, hashed-IP privacy).
- **Country data** comes from `uTPro.Feature.GeoLocation`, which resolves the real client
  IP through CDN/proxy chains (Cloudflare CF-Connecting-IP, X-Forwarded-For, ...) and the
  offline MMDB. AdProtect reuses that resolution — it never trusts client-supplied IPs.

## Static front-ends (Jekyll/other hosts)

The gate is cache-safe: the decision happens in a fetch to the AdProtect API, not in
Razor. Point a slot on any static site at the API:

    <div data-utpro-ad-slot="banner-top"
         data-utpro-ad-api="https://api.t4vn.com/api/adprotect"></div>
    <script src="https://api.t4vn.com/App_Plugins/utpro-ad-protect/adprotect.js" defer></script>

CORS: serve the API on the same registrable domain (or add CORS headers) so the check
fetch and the anonymous cookie flow. The check sets the `utpro_ap_id` cookie on the API
host; `credentials: 'same-origin'` from a different site will not carry it — same-site
embedding (or a subdomain-scoped Domain cookie) is the supported setup.

## Compliance notes (AdSense ToS)

- AdProtect only **withholds** ad code from selected visitors — it never modifies ad
  markup, never frames clicks, never asks users to click.
- Slot "Auto Ads" are out of scope: Auto injects everywhere, so per-slot gating is
  impossible. Use manual units.
- Changing the click-budget defaults drastically to inflate revenue would be a ToS
  violation; keep the defaults (10 IP / 3 cookie / day) unless you have a reason.
