---
name: feedback-gitea-use-claude-bot
description: "Use the claude-bot Gitea account (not the user's personal SSH key) for git/API operations against git.meneerak.nl"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 9de6531a-fba9-46c5-a896-7454b760d00c
  modified: 2026-08-06T12:44:50.728Z
---

When pushing commits, opening PRs, or doing any other git/API operation against
`git.meneerak.nl` (self-hosted Gitea), use the dedicated `claude-bot` account, not the
user's own SSH key (`~/.ssh/meneerak_gitea`, account `meneerAk`) or the `bram` admin
account.

**Why:** stated explicitly after pushing a `FreeGamesBot` feature branch over the
user's personal SSH remote instead of `claude-bot` — the whole point of setting up
`claude-bot` (see the org/token setup work in this project) was so git operations run
under a distinct identity, not the user's own.

**How to apply:**
- REST API calls (repo/org/PR management): use `claude-bot`'s personal access token
  (Authorization: token header). Already the established pattern.
- `git push`/`git pull`/`git clone` for repo content: use `claude-bot`'s token over
  HTTPS (`https://claude-bot:<token>@git.meneerak.nl/...`, or `-c
  http.extraHeader="Authorization: token <token>"` against the `clone_url`), not the
  SSH remote tied to the user's personal account. `claude-bot` has no SSH key
  configured — HTTPS+token is the way to authenticate as it for git operations, not
  just the API.
- This applies across every repo `claude-bot` has access to (currently: `projectjes`,
  `Projects`, `flux` orgs — see the Gitea org/account setup notes for this project),
  not just `FreeGamesBot`.
