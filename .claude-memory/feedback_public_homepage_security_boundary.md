---
name: feedback-public-homepage-security-boundary
description: meneerak.nl (public homepage) never lists credential/security-adjacent tools like Vaultwarden - only ops.meneerak.nl does
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e916db36-de91-4615-91ca-16da7df1e5aa
  modified: 2026-08-26T08:02:44.029Z
---

`meneerak.nl` (the public homepage config, `apps/homepage/configmap.yaml`)
has **no auth gate by design** - that's the entire point of splitting it
from `ops.meneerak.nl` (gated behind oauth2-proxy + the "ops" Zitadel
role, `apps/homepage/ops-configmap.yaml`). Anything listed on the public
one is effectively announced to any random visitor who loads the page.

**How to apply**: when adding a new app's tile to the homepage dashboards,
default to adding it to `ops.meneerak.nl` only if the app is
security/credential-adjacent (a password manager, an admin panel, anything
whose mere existence-at-this-URL is worth not advertising) - even if the
app's own domain is separately reachable/necessary to be public (e.g.
Vaultwarden's main site has to be ungated for native clients, see its own
README). The two questions are independent: "does this app's *traffic*
need to be public" vs "should this app's *link* be on the public
dashboard" - Vaultwarden (2026-08-24) is the precedent: ungated site, but
ops-only homepage tile, since there's no upside to making a password
manager's URL more discoverable for zero user benefit (single-user setup).
Apps with genuinely no downside to public discovery (Filestash, Pass/
yopass, Dawarich, Jellyfin, Jellyseerr) go on both.
