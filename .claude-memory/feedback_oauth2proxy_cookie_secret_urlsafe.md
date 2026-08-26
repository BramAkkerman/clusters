---
name: feedback-oauth2proxy-cookie-secret-urlsafe
description: "oauth2-proxy's cookie-secret must be URL-safe base64 (tr +/ -_) - plain openssl rand -base64 32 can silently produce a broken value"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e916db36-de91-4615-91ca-16da7df1e5aa
  modified: 2026-08-26T08:02:22.166Z
---

Every `oauth2-proxy` instance in this project (filestash, jellyseerr,
vaultwarden, uptime-kuma, media, ops) needs a 32-byte `cookie-secret`,
base64-encoded. Generating it with plain `openssl rand -base64 32` is
**not safe** - oauth2-proxy expects **URL-safe** base64 (`-`/`_`), and
standard base64 (`+`/`/`) only happens to work when the random bytes don't
produce a `+` or `/` character. When they do, oauth2-proxy fails to decode
it, falls back to checking the raw string's byte length (44 for a 32-byte
value's base64 form), and errors: `cookie_secret must be 16, 24, or 32
bytes to create an AES cipher, but is 44 bytes`.

**Why this matters**: every cookie-secret generated so far in this repo
before 2026-08-25 (filestash, jellyseerr, vaultwarden) used plain
`openssl rand -base64 32` and happened to work purely because none of
those particular random values contained `+` or `/` - not because the
method was actually correct. They're fine as-is (already deployed and
working), but this was a coin flip each time, not a solved problem.

**How to apply**: always generate oauth2-proxy cookie secrets with:
```bash
openssl rand -base64 32 | tr -- '+/' '-_'
```
Discovered when uptime-kuma's oauth2-proxy hit exactly this failure
(2026-08-25) - the generated secret contained a `+`, oauth2-proxy
rejected it, fixed by regenerating with the `tr` transliteration above.
