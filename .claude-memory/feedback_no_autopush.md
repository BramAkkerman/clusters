---
name: feedback-no-autopush
description: Never git push (or otherwise ship/deploy) changes without the user reviewing them first
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 4e3dfc11-27cb-4632-a376-73db070f8a06
  modified: 2026-08-06T11:45:34.003Z
---

Don't push commits, or otherwise deploy/apply changes to a live system, without the user reviewing them locally first. Stop after committing/writing code and let the user decide when to push.

**Why:** stated explicitly after a code rewrite (FreeGamesBot's already-seen tracking moved to SQLite) was left uncommitted pending review — the user wants that as the default, not just for that one instance.

**How to apply:** after writing or rewriting code (in this `clusters` repo, the Gitea-hosted `flux/apps` mono-repo, or any other repo in this workspace like `FreeGamesBot`), leave it staged/local and describe what changed. Wait for an explicit go-ahead ("push it", "go ahead", etc.) before running `git push`. This is distinct from the existing infra convention of applying live `kubectl`/Flux-reconcile changes during active development — this rule is specifically about the git push (and by extension, remote-visible) step being a separate, explicit action.
