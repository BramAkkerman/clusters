# Project memory

This repo is a Flux GitOps mono-repo managing multiple clusters. Layout convention:

- `clusters/<name>/` — Flux bootstrap (`flux-system/`) + top-level `apps.yaml` / `infrastructure.yaml` Kustomizations for that cluster
- `apps/<name>/` — actual app manifests (namespace, HelmRelease, HelmRepository, CNPG Cluster, etc.)
- `infrastructure/<name>/` — supporting infra (cert-manager, cnpg-operator, traefik config, etc.), one subdirectory per component

Both clusters sync from `https://github.com/BramAkkerman/clusters` (this repo), `main` branch, each with its own `path:` into `clusters/<name>`.

## Clusters

### `zitadel-vm` (kubeconfig context: `zitadel-vm`)
Single-node k3s VM, public IP `81.24.6.55`. Hosts Zitadel at `auth.meneerak.nl`.

- Postgres via CNPG (`apps/zitadel/postgresql-cluster.yaml`), not the chart's bundled Bitnami postgres.
- TLS terminates at k3s's **bundled** Traefik (`kube-system/traefik`, not Flux-managed as a HelmRelease — only its `HelmChartConfig` override is, under `infrastructure/zitadel/traefik-config/`).
- cert-manager + `letsencrypt-prod` ClusterIssuer, HTTP-01 via Traefik.

### `ak` / EMK (kubeconfig contexts: `ak` (external, use this one), `ak-wildcard`, `ak-internal`)
Cyso Cloud "Enterprise Managed Kubernetes" — Gardener-based (Shoot terminology, `core.gardener.cloud` API), backed by Fuga Cloud's OpenStack. Project `xc3wlbs`, shoot `ak`. Hosts Gitea at `git.meneerak.nl`, was freshly bootstrapped (nothing pre-installed) — Traefik, cert-manager, cnpg-operator are all Flux-managed HelmReleases here (`infrastructure/emk/`), unlike zitadel-vm.

- Access: dashboard kubeconfigs expire in ≤24h. Longer-lived path is a 90-day service-account kubeconfig used to mint fresh 24h `adminkubeconfig`s on demand (see chat history for the exact `kubectl create --raw .../adminkubeconfig` one-liner). True no-renewal access would need OIDC (Zitadel is a supported provider — nice synergy with our own instance — but not set up).
- Gitea admin user: `bram`, credentials in `gitea-admin` secret (namespace `gitea`, manually created, not in git).
- Gitea DB: CNPG, bootstrap-only (no superuser dance needed — unlike Zitadel, Gitea only needs its own DB user).
- Gitea chart: bundled Bitnami postgres/valkey-cluster subcharts explicitly disabled (default ships a 3-node Valkey cluster — overkill for personal use); using Gitea's built-in in-memory cache/session instead.

## Secrets (all "bring your own" — created manually via `kubectl create secret`, never committed)

| Secret | Namespace | Cluster | Keys |
|---|---|---|---|
| `postgresql-auth` | zitadel | zitadel-vm | username, password |
| `postgresql-superuser` | zitadel | zitadel-vm | username, password |
| `zitadel-masterkey` | zitadel | zitadel-vm | masterkey |
| `zitadel-smtp` | zitadel | zitadel-vm | user, password |
| `postgresql-auth` | gitea | ak | username, password |
| `gitea-admin` | gitea | ak | username, password |

Pattern used throughout: passwords are wired into HelmReleases via top-level `env:` + `secretKeyRef`, using each app's own env-var config override convention (`ZITADEL_<SECTION>_<KEY>` / Gitea's `GITEA__<section>__<KEY>` via `additionalConfigFromEnvs`) — never placed directly in `values:` blocks, which would land in a plaintext ConfigMap or Helm release secret.

## Non-obvious gotchas hit along the way

- **CNPG disables the `postgres` superuser by default** (`enableSuperuserAccess: false`) — no password login exists at all. Zitadel's init job needs that superuser to bootstrap. Fix: `enableSuperuserAccess: true` + explicit `superuserSecret.name`. Once, the CNPG operator didn't pick up this spec change on its own — needed a manual annotation bump to force reconciliation. Gitea didn't need any of this (only needs its own app-user DB access).
- **Zitadel v2.67.2 defaults `TLS.Enabled: true`** internally and crashes without a cert/key if so — must explicitly set `configmapConfig.TLS.Enabled: false` when TLS terminates at the ingress (Traefik).
- **Traefik's HTTP→HTTPS redirect is not automatic**, and the values key differs by chart-version generation. `ports.web.redirectTo.port: websecure` **silently no-ops** on the k3s-bundled chart (helm-install job succeeds, deployment never actually changes) — the real key on the pinned chart versions here is `ports.web.http.redirections.entryPoint.{to,scheme,permanent}`. Without this, Zitadel's own OIDC console client (registered with an `https://` redirect_uri via `ExternalSecure: true`) rejects logins reached over plain `http://` with `redirect_uri is http and is not allowed`.
- **Always verify Helm values against the actual pinned chart version's real `values.yaml`**, not whatever's on the chart repo's `main`/default branch — got burned twice by schema drift between versions (traefik redirect key above, and Zitadel `main`-branch chart docs showing features like `FirstInstance.Org.Machine` that don't exist in the pinned `8.13.4`).
- **`flux bootstrap` only auto-commits its own generated `flux-system/*` files** — hand-authored Kustomizations/manifests (`apps.yaml`, `infrastructure.yaml`, app manifests) still need a manual `git add/commit/push`. Nearly shipped the whole EMK/Gitea setup without ever pushing it — Flux only had the bootstrap files and nothing else reconciled until the missing commit was pushed. Always `git status` before assuming something's live.
- **`flux bootstrap github` needs a GitHub token with deploy-key management permission** — fine-grained tokens need `Administration: Read and write` (not just `Contents`), classic tokens need full `repo`. A 403 on `GET .../repos/.../keys` mid-bootstrap means the token scope, not the bootstrap logic, is the problem; rerunning after fixing the token is safe (idempotent).
- **The original `zitadel-admin` bootstrap password was lost.** Zitadel's setup job prints the auto-generated first-admin password to its own pod logs exactly once, at first-ever setup — and that job re-ran (and its pod got recycled) multiple times during earlier debugging (postgres-auth fix, TLS fix), so the credential is gone. `DefaultInstance.*` config (SMTP, initial admin password) only applies at first-ever instance creation and can't retroactively configure an already-bootstrapped instance. Planned fix: wipe the CNPG database and let Flux redeploy fresh, this time with `DefaultInstance.SMTPConfiguration` (and ideally a known admin password) baked in from the start.

## DNS (`meneerak.nl`)

Migrating DNS delegation from Strato (registrar, `rzone.de` nameservers) to Cyso Cloud DNS (same place the EMK/zitadel-vm servers live). DNSSEC was active during the migration — correct/safe unsigning order is: remove the `DS` record at the registry (`.nl`/SIDN) first, then let the child zone's own signing (`DNSKEY`/`RRSIG` at Strato) wind down after — verify via `dig +short DS meneerak.nl` returning empty, not by trusting Strato's dashboard status flag alone (it lagged behind the actual registry state).

Records in play: `auth.meneerak.nl` → zitadel-vm Traefik (`81.24.6.55`), `git.meneerak.nl` → EMK Traefik LB (`81.24.6.106`, service name `traefik-traefik` — Helm fullname template concatenates release+chart name since both are literally `traefik`).

SMTP for Zitadel: Cyso Cloud's Transactional Email Service ("pidge"), `send.pidge.cyso.cloud:587`, sender `noreply@meneerak.nl` — needs SPF/DKIM TXT records added at Cyso Cloud DNS once their dashboard generates the domain-specific values (not knowable in advance, unlike a fixed SPF include).

## Open follow-ups

- Wipe zitadel's Postgres DB + redeploy fresh so SMTP and a known admin password take effect (approved, not yet executed as of last session).
- Add SPF/DKIM records for `meneerak.nl` once Cyso's Transactional Email Service dashboard shows the domain-specific values.
- Migrate `clusters/zitadel/flux-system/gotk-sync.yaml`'s `GitRepository` source from GitHub to a repo hosted on the new Gitea instance, once Gitea itself is confirmed reachable and mirrored — deliberately not done yet, sequenced after Gitea is stable.
- Possible future: set up EMK OIDC auth using the user's own Zitadel as identity provider (via `kubelogin`), for no-manual-renewal `kubectl` access. Discussed as a nice synergy, not started — real setup cost (K8s ≥1.30, `AuthenticationConfiguration`, RBAC mapping, local `kubelogin`).
- **Latent bug on `zitadel-vm`**: its Traefik also has the entrypoint-level HTTP→HTTPS redirect (`ports.web.http.redirections.entryPoint`) — this fundamentally breaks ACME HTTP-01 challenges, since it redirects the challenge path itself to HTTPS before any cert exists (confirmed by removing the equivalent config on EMK's Traefik, added for Gitea, after it caused `gitea-tls` to get permanently stuck `pending`). It hasn't bitten `zitadel-vm` yet because `zitadel-tls` was already issued *before* that redirect was added. It **will** break the next automatic renewal (~90 days from issuance) unless fixed first — the entrypoint-level redirect needs replacing with a per-ingress opt-in (Traefik `Middleware` + annotation on just the app's own Ingress), which leaves cert-manager's dynamically-created ACME solver Ingress un-redirected.
## Resolved: `DefaultInstance.Org.Human.Password` doesn't actually work — use `FirstInstance`

Set the bootstrap admin's password via `DefaultInstance.Org.Human.Password` (env var `ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_PASSWORD`) and every login attempt failed with "Password is invalid" — confirmed via `claude-in-chrome` driving the actual login form with the byte-verified password directly (ruling out any copy/paste error), and via the eventstore showing real `user.human.password.check.failed` events. The password gets embedded in the `user.human.added` creation event fine, but this specific path is a known, confirmed-by-a-maintainer Zitadel Helm chart bug (see [zitadel/zitadel-charts#156](https://github.com/zitadel/zitadel-charts/issues/156) — "I think you need to use `FirstInstance` instead").

`FirstInstance` is a dedicated section (`cmd/setup/steps.yaml` in zitadel core) specifically designed to reliably override `DefaultInstance` for the first-ever setup — same nesting (`FirstInstance.Org.Human.Password`, env `ZITADEL_FIRSTINSTANCE_ORG_HUMAN_PASSWORD`), and it's the one that's actually reliable. Note `DefaultInstance.SMTPConfiguration` (a *different* subsection) worked correctly the whole time — this bug is specific to `Org.Human` user creation, not `DefaultInstance` as a whole. There's no `FirstInstance.SMTPConfiguration` equivalent, so SMTP config correctly stays under `DefaultInstance`.

## Another gotcha: EMK's Traefik `IngressClass` is `traefik-traefik`, not `traefik`

On EMK, the Traefik HelmRelease is named `traefik` and the chart is *also* named `traefik` — Helm's fullname template concatenates release+chart name when they're literally identical, so every object it creates is `traefik-traefik` (Service, Deployment, and critically the `IngressClass`). Any Ingress (or cert-manager `ClusterIssuer` HTTP-01 solver config) that sets `ingressClassName: traefik` silently matches nothing — Traefik never registers a router for it at all, and requests fall through to Traefik's own generic `404 page not found` (indistinguishable at a glance from a real app-level 404). This cost a long detour debugging a "broken ACME challenge" that was actually just this. Fixed in both `apps/gitea/gitea-release.yaml` (`ingress.className`) and `infrastructure/emk/cert-manager-config/cluster-issuer.yaml` (`solvers[].http01.ingress.ingressClassName`) → `traefik-traefik`. `zitadel-vm`'s k3s-bundled Traefik doesn't have this problem — its `IngressClass` is genuinely named `traefik`, since it's k3s's own add-on, not a same-named Flux HelmRelease. Always check `kubectl get ingressclass` on EMK rather than assuming the name matches the release.
