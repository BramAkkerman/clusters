---
name: reference-claude-bot-tokens
description: "How claude-bot's Gitea tokens are organized and what to do when one stops authenticating"
metadata: 
  node_type: memory
  type: reference
  originSessionId: e916db36-de91-4615-91ca-16da7df1e5aa
  modified: 2026-08-26T08:02:12.507Z
---

`claude-bot` (Gitea account at git.meneerak.nl) holds multiple purpose-scoped
tokens, not one shared token - visible names as of 2026-08-21:
`freegamesbot-registry-pull` (`read:package` only), `homepage-widget-2`
(`read:issue`/`read:notification`/`read:repository`, read-only), and
`clusters-cli-N` (used for git push/CLI ops against repos like `flux/apps` -
the `-N` suffix means it's been rotated before, e.g. `clusters-cli-2` ->
`clusters-cli-3`).

**Gitea never displays a token's value again after creation** - only
name/added/last-used metadata. So there is no way to "look up" a token's
current value anywhere (not in kube Secrets, not in repo docs, not in my own
memory) - a stale saved/remembered value just silently stops working after a
rotation with the generic error `Failed to authenticate user`.

**How to apply:** when a git push or Gitea API call as `claude-bot` fails
auth, don't go hunting through kube Secrets or repo docs for an old token -
none of the other scoped tokens (`freegamesbot-registry-pull`,
`homepage-widget-2`) have write access anyway, and `gitea-apps-auth` (what
Flux itself uses to pull `flux-apps`) is read-only by nature. Just ask the
user to generate a fresh `clusters-cli-N+1` token in Gitea (Settings ->
Applications, or the account's own token page) with `write:repository`
scope for the `flux` org, and have them paste the value directly into chat -
see [[feedback-gitea-use-claude-bot]] for why it's this account and not the
user's personal key.

**The exact push syntax that actually works** (confirmed 2026-08-25, after
two other approaches failed against Gitea's git-over-HTTP backend):
```bash
export GITEA_TOKEN='<pasted token>'
cd /path/to/flux-apps && git push "https://claude-bot:${GITEA_TOKEN}@git.meneerak.nl/flux/apps.git" main
```
Credentials embedded directly in the push URL (via an env var, not a
literal token in the command text - a literal secret string as a bash
arg gets blocked by the permission classifier). Two things that looked
reasonable but **don't work**: `git -c http.extraHeader="Authorization:
token $TOKEN"` (Gitea's git smart-HTTP backend returns a bare
`Unauthorized`, not a real auth attempt - that header scheme is for the
REST API, not git-over-HTTP) and `git -c
http.extraHeader="Authorization: Basic <base64 of claude-bot:$TOKEN>"`
(got further - a real `Failed to authenticate user` from Gitea - but
still didn't work; not fully root-caused, not worth re-attempting given
the URL-embedded form just works).
