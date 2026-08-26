---
name: feedback-avoid-disproportionate-fixes
description: "Weigh infra complexity/coupling cost against the actual value of a fix, especially cosmetic ones - don't add a volume/ConfigMap just to change a browser tab title"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e916db36-de91-4615-91ca-16da7df1e5aa
  modified: 2026-08-26T08:02:34.746Z
---

When fixing something low-value or cosmetic, don't reach for a solution
that adds real infrastructure or version-coupling risk just to get it
"fully" fixed.

**Concrete incident (2026-08-22)**: yopass's browser tab title couldn't be
changed via its own `--app-name` flag (that only brands in-app UI text,
not the actual `<title>` tag, which is static HTML baked into the built
frontend). The technically-complete fix was a ConfigMap overriding the
served `index.html` via a `subPath` volumeMount - built and confirmed
working, but it hardcoded that exact image version's Vite-generated
`/assets/*-<hash>.js/css` filenames, meaning it would silently break on
the next routine image bump unless kept in lockstep by hand. User's
reaction: *"We don't need to wire a whole extra volume for this fix only."*
Reverted the ConfigMap/volume entirely, kept only the free `--app-name`
flag, left the actual tab title as the app's default.

**Why**: a tab title has near-zero functional value. Trading that for a
new, easy-to-forget failure mode (an image upgrade silently breaking the
page because a ConfigMap's hardcoded asset hashes no longer match) is a
bad trade for this project's personal-homelab scale and risk tolerance.

**How to apply**: before reaching for a ConfigMap override, a sidecar, a
volume, or anything that creates ongoing coupling to a specific upstream
version/internal detail, ask whether the thing being fixed is actually
worth that maintenance burden. Cosmetic/nice-to-have fixes generally
aren't - prefer the free/cheap partial fix (or no fix) over a "complete"
one that adds a new way for things to quietly break later. This is the
same underlying instinct as [[feedback-no-autopush]] and the general
"personal cluster, not a 100 FTE company" framing the user gave when
rejecting the first Zitadel migration plan - keep solutions sized to the
actual stakes.
