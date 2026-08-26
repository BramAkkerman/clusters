---
name: project-cyso-volume-quota-blocked
description: "Cyso Cloud Cinder volume quota (15 max) is exhausted account-wide - uptime-kuma's PVC is stuck Pending until the user frees one up or requests an increase"
metadata: 
  node_type: memory
  type: project
  originSessionId: e916db36-de91-4615-91ca-16da7df1e5aa
  modified: 2026-08-26T08:02:55.294Z
---

As of 2026-08-25, creating `uptime-kuma-data` (a new 2Gi PVC on `ak-oidc`)
failed provisioning with `cinder.csi.openstack.org`:
```
ResourceExhausted: VolumeLimitExceeded: Maximum number of volumes allowed
(15) exceeded for quota 'volumes'
```

**Why**: this is an account-wide Cyso Cloud quota, not a per-cluster or
per-namespace one. `ak-oidc` itself only accounts for 13 PVs (checked via
`kubectl get pv` - all 13 are legitimately in use, nothing orphaned to
reclaim there: gitea, media x2, vaultwarden, cividle, dawarich x3,
filestash, wireguard, jellyseerr, freegamesbot). The remaining 1-2 volumes
against the quota are outside this cluster's visibility from `kubectl` -
most likely `zitadel-vm`'s disk(s), since that's the only other Cyso
Cloud resource in this project. No `openstack`/`cyso` CLI is available in
this environment to inspect the full account-wide volume list directly.

**How to apply**: this blocks nothing except `uptime-kuma`'s pod from
actually starting (the Deployment/Service/Ingress/cert all deployed fine
and are just waiting - safe to leave as-is, no cleanup needed on the k8s
side). It will self-heal automatically the moment quota frees up - no
need to re-touch the uptime-kuma manifests. The user needs to either
delete an unused volume via the Cyso Cloud console, or request a quota
increase - I have no way to do either myself. Worth checking with the
user whether the Zitadel VM migration ([[reference-claude-bot-tokens]]
territory aside, see `ZITADEL_MIGRATION.md`) is relevant here too, since
that migration's old-VM-kept-around-as-rollback step means *two* VMs'
worth of volumes will exist simultaneously for a while, temporarily
making this quota pressure worse, not better, until the old VM is deleted.
