# Plans

Ideas discussed for this homelab that haven't been built yet - a running
list so they don't get lost between sessions. Not commitments or a
roadmap with dates, just "worth doing eventually." Move an entry out once
it's actually built (and mention where, e.g. a README).

## Monitoring / observability

**Status: partially started** - Uptime Kuma (`apps/uptime-kuma`) went in
2026-08-25 for basic up/down status + alerting. Still open:

- **Metrics + alerting (Prometheus + Grafana or similar)**: would have
  caught several incidents earlier as trends rather than after-the-fact
  investigations - the node RAM exhaustion that cascaded into Gitea's
  Postgres crash, qBittorrent's disk filling silently, Dawarich's CPU
  throttling tripping its startup probe (see `MEMORY.md` for all three).
  Real cost: a full Prometheus stack is heavy for this cluster's scale
  (500Mi-1Gi+ just for Prometheus itself) and is another thing to
  maintain. Worth revisiting if Uptime Kuma's simpler up/down checks turn
  out to be insufficient.
- **Log aggregation (Loki)**: pairs with Grafana if that route is taken -
  would have sped up the Jellyfin ancestor-index bug and the qBittorrent/
  gluetun firewall debugging, both of which involved a lot of manual
  `kubectl logs` across containers.

## Secrets management

**SOPS** (or similar) for encrypting secrets into git, replacing the
current "paste the value into chat when needed, `kubectl create` by hand,
never committed" pattern used for every secret in this project. Would
remove the recurring friction of re-deriving/re-requesting a value nobody
wrote down anywhere (hit this directly with `claude-bot`'s rotating Gitea
tokens). Real cost: a genuine workflow change from what's established, and
was explicitly dropped as a blocking prerequisite for the Zitadel VM
migration (`ZITADEL_MIGRATION.md`) to keep that plan simple - so the
tradeoff has already been considered once and punted on.

## Backups

- **Cluster-wide backup (e.g. Velero)** for PVCs/resources in general,
  rather than the current per-app approach (only Vaultwarden has a real
  backup - `apps/vaultwarden/backup.yaml`). Most apps here are either
  re-fetchable (media library) or low-stakes if lost; worth a pass to
  identify which ones actually hold irreplaceable state the way
  Vaultwarden and Dawarich's location history do.

## oauth2-proxy cookie scoping

Every oauth2-proxy in this project (filestash, jellyseerr, cividle,
vaultwarden, uptime-kuma, media, and now stremio - the one exception)
sets `OAUTH2_PROXY_COOKIE_DOMAINS: .meneerak.nl` (domain-wide) instead of
scoping to its own host. Since each app already has its own dedicated
Zitadel client + unique cookie name, this provides no actual shared-login
benefit - it just means the browser sends *every* app's session cookie
on every request to *any* subdomain, growing every time a new app is
added. Hit a real "Request Header Or Cookie Too Large" error on stremio
(2026-09-07, see `apps/stremio/README.md`) once 7 apps' cookies stacked
up against its bundled nginx's stricter header-size limits - fixed for
that one app by scoping it to `stremio.meneerak.nl` specifically. The
other 6 apps are equally exposed to this, just haven't tripped it yet.
Retrofitting all of them to host-specific cookie domains is the real
fix - not done broadly yet since changing cookie domain invalidates
existing sessions on each app touched, so it's a deliberate one-at-a-time
or coordinated change, not something to slip in as a side effect of
something else.

## Infrastructure consistency

- **Migrate `zitadel-vm`'s Flux `GitRepository` source from GitHub to
  Gitea** - analyzed in depth 2026-08-19, concluded safe but low priority
  (full reasoning and the circular-dependency risk that was ruled out:
  `MEMORY.md`). Modest benefit (drops a GitHub dependency, consistent with
  `ak-oidc`'s self-hosted-where-practical pattern), not fixing any actual
  pain point. Do whenever convenient.
- **Zitadel VM migration to a clean Talos box** - plan is written
  (`ZITADEL_MIGRATION.md`), execution is on the user, in progress as of
  2026-08-21.
