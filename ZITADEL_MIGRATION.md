# Migrate Zitadel to a new, clean 4GB VM

## Context

`zitadel-vm` is a repurposed box that's picked up cruft over time (the
forgotten old-Dawarich-via-Docker stack we just cleaned up being the latest
example). Goal: a clean 4GB VM with just Zitadel and its actual dependencies
on it, carrying over the real data (users, every app's OIDC client
registrations, etc.) rather than starting from an empty instance - all the
app-by-app SSO wiring done in this project would otherwise need redoing.

This is a personal homelab, single user, no one else depending on uptime -
so this is a "do it in one sitting, nobody's trying to log in anyway" job,
not a zero-downtime production migration. The main things that actually
matter: don't lose the encryption key, don't lose the database, keep the old
box around for a few days in case something's wrong.

## What's actually needed

- Zitadel's data (users, orgs, every dependent app's OIDC client
  registrations) lives only in its Postgres database (CNPG-managed) - that
  DB is the real thing being migrated, not just config.
- `zitadel-masterkey` (a Secret, not in git) encrypts sensitive data at
  rest - copy it over exactly or existing encrypted data becomes unreadable.
  4 other secrets (`postgresql-auth`, `postgresql-superuser`, `zitadel-smtp`,
  `zitadel-admin`) also need copying over.
- The chart versions in git are floating ranges (`zitadel: "8.x"`, etc.) -
  worth pinning to what's actually running before bootstrapping a new
  cluster, so it doesn't silently pull a newer version and run unexpected
  migrations against restored data. Currently running: Zitadel chart
  `8.13.4` (app `v2.67.2`), CNPG operator `0.23.2` (Postgres image
  `ghcr.io/cloudnative-pg/postgresql:17.4`), cert-manager `v1.16.5`, k3s
  `v1.36.2+k3s1`.
- `infrastructure/zitadel/traefik-config/helmchartconfig.yaml` must stay the
  empty `{}` it already is - it used to carry an HTTP→HTTPS redirect that
  broke ACME renewal (already-documented incident); don't reintroduce it.

## Steps

1. **Note down the 5 secret values** (`postgresql-auth`, `postgresql-superuser`,
   `zitadel-masterkey`, `zitadel-smtp`, `zitadel-admin`) somewhere safe -
   you'll recreate them on the new box.
2. **Pin the floating chart versions** in `apps/zitadel/zitadel-release.yaml`,
   `infrastructure/zitadel/cert-manager/release.yaml`,
   `infrastructure/zitadel/cnpg-operator/release.yaml` to the exact versions
   above, plus add `imageName: ghcr.io/cloudnative-pg/postgresql:17.4` to
   `apps/zitadel/postgresql-cluster.yaml` (currently unset). Commit + push.
3. **Snapshot the old VM** at the Cyso console - cheap insurance before
   touching anything.
4. **Build the new VM**: Talos Linux (via Image Factory - `Nocloud` platform,
   Secure Boot off, `qemu-guest-agent` system extension), 4GB RAM, a
   persistent (Cinder-backed) 30GB volume, then
   `flux bootstrap github --owner=BramAkkerman --repository=clusters --branch=main --path=clusters/zitadel --personal`
   - same repo/branch/path as the old one; it's a separate cluster/context,
   so this just stands up its own independent copy of everything (cert-manager,
   CNPG operator, an empty CNPG Postgres, and a fresh throwaway Zitadel
   instance you're about to overwrite anyway). Let it fully come up.
5. **Stop the old Zitadel** (scale its Deployment to 0) - freezes the real
   data so the dump you're about to take is the final one.
6. **Dump the old database**: `pg_dumpall --globals-only` (gets role
   passwords as SCRAM hashes) + `pg_dump -Fc zitadel`. Copy both to the new
   VM.
7. **Recreate the 5 secrets** on the new cluster with the values from step 1.
8. **Wipe the new cluster's fresh/throwaway Zitadel DB and restore the real
   dump**: restore globals first, then the database, into the new cluster's
   Postgres (expect and ignore "role already exists" - CNPG's own bootstrap
   already created it). Restart the Zitadel pods on the new cluster so they
   pick up the restored data.
9. **Sanity check**: log into the Zitadel console on the new instance (port-forward
   or hosts-file override for `auth.meneerak.nl` pointed at the new IP) and
   confirm your users/orgs/apps are actually there - not a fresh empty instance.
10. **Copy the `zitadel-tls` Secret** (cert+key) from old to new so it's
    serving a valid cert immediately - avoids waiting on a fresh Let's
    Encrypt issuance right at cutover. (Not critical for a personal box,
    but it's a 30-second copy that avoids a dumb wait.)
11. **Flip DNS** (`auth.meneerak.nl` → new VM's IP).
12. **Spot-check 2-3 dependent apps** actually still log in against the new
    instance without touching their client-id/secret (e.g. Sonarr's
    oauth2-proxy, homepage's ops view) - confirms the app registrations
    really did come across.
13. **Keep the old VM around, powered off or just untouched, for a few days**
    before deleting it - if something's wrong, flip DNS back and turn its
    Zitadel back on.

## Rollback

Old VM's data is never touched (only read via `pg_dump`), so rollback at any
point before you delete it is just: flip DNS back, scale its Zitadel back up.
