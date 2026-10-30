# Review: PR #49 — "Keep one location watcher and clarify GPS recovery states"

Author: aafham · Branch: `aafham:claude/geolocation-lifecycle` · Base commit: `134045a`

## Verdict: Approve with one required fix (the `sw.js` conflict)

The change itself is good. It fixes a real bug, it is well tested, and the new messages
are clearer than the ones on `main`. The only thing standing between this PR and merge is
the offline shell list, which needs the union of three branches' entries.

## What I verified

Merged onto current `main` (`7ff9dda`) and checked:

| Check | Result |
| --- | --- |
| `npm test` | 199 pass, 0 fail |
| `node --check` on every changed `.js` | clean |
| `git merge-tree --write-tree upstream/main HEAD` | merges cleanly |
| Browser, real GPS fix injected | location banner clears, app opens |
| Browser, permission denied | shows the new "Location permission is off" message |
| `scripts/check-offline-shell.mjs` (from PR #50) | "Offline shell covers 27 local modules; all parsed successfully." |

## The bug it fixes

On `main`, `public/app.js` called `navigator.geolocation.watchPosition()` from two places
and never cancelled the old one. Every retry left another watch and another timer alive.
It also treated every error the same: a timeout was reported to the driver as
"Turn on location so A2B can show where you are", which reads as a permission problem
but is usually not one.

I hit this myself while setting the project up: with no GPS available (a VMware guest,
a desktop without location services), the app sat on a misleading banner with no way to
tell a timeout from a refusal. This PR makes those two cases distinct, which is the
single most useful thing it does.

## Why the logic is right

`public/location.js` gives the watch a single owner and uses a generation counter so a
late callback from a replaced watch is ignored rather than applied:

- `generation` is bumped on every `start()` and `stop()`; a callback only counts if
  `token === generation`. This is what stops a timed-out watch from resurrecting itself.
- `fail()` clears the watch before scheduling the retry, so there is never more than one
  live watch and one pending timer. The test asserts exactly that.
- Permission denial (`code === 1`) does not retry on its own — only a deliberate tap on
  "Try again" asks the browser again. Transient failures (`code === 2` and `code === 3`)
  retry after 15 s.
- The callback fires with `watchPosition` before it returns the watch ID in some
  implementations, so the ID is checked against the generation before being stored, and
  cleared if it is already stale. Correct and worth the two lines.

`test/location.test.js` covers the cases that matter: repeated starts, late callbacks
after a failure, the distinction between timeout and denial, page hide/show, and a
synchronous callback before the watch ID exists.

## Required fix before merge: the `public/sw.js` SHELL list

`public/sw.js` conflicts on the `SHELL` list. This is the case `AGENTS.md` describes:

> When PRs conflict only in `public/sw.js`'s `SHELL` list, the README, or side-by-side
> imports, keep both sides on a `codex/land-<n>-<m>` branch and open one PR.

The three lists are not the same, and each one alone is wrong:

| Source | Adds | Drops |
| --- | --- | --- |
| `main` (after #51) | `/suggest.js` | — |
| PR #49 | `/location.js` | — |
| PR #50 | `/route-preview.js`, `/probe.js` | `/suggest.js` |

The correct `SHELL` list is the union of all three, 31 entries:

```
'/', '/styles.css', '/app.js', '/core.js', '/panels.js', '/nav.js', '/motion.js',
'/voice.js', '/vehicle3d.js', '/geo.js', '/icons.js', '/report-types.js',
'/map-themes.js', '/places.js', '/text.js', '/sheet-drag.js', '/follow.js',
'/presence.js', '/welcome.js', '/update.js', '/offline.js', '/train-guide.js',
'/signs.js', '/dead-reckoning.js', '/map-places.js', '/location.js', '/suggest.js',
'/icon.svg', '/manifest.webmanifest', '/route-preview.js', '/probe.js'
```

## A separate bug this exposed, on `main` itself

While checking the shell list I found that `main` is already missing two modules that
`public/app.js` imports:

- `/route-preview.js` — imported at `public/app.js:7`
- `/probe.js` — imported at `public/app.js:22`

Neither is in `main`'s `SHELL` list. On a phone that has the app installed and then goes
offline, those two imports fail and the app cannot start. PR #50 fixes exactly this,
which is good work on its own. It also ships `scripts/check-offline-shell.mjs`, which
walks the import graph and fails when the shell does not cover it — that check should
have been in the repo from the start.

I have put the combined result on a branch in my fork:

```
https://github.com/Kaizoku47/a2b/tree/review/land-49-50
```

It merges both PRs, resolves `public/app.js` (both imports kept) and `public/sw.js`
(union list), and passes `npm test` (199 pass), `node --check`, and PR #50's own
`check-offline-shell.mjs`.

Two smaller things in the same area:

- The `CACHE` version in `public/sw.js` needs a bump (`a2b-v39` → `a2b-v40`) when the
  `SHELL` list changes, per `AGENTS.md`. Both PRs bump `a2b-v38` → `a2b-v39` independently,
  so the combined branch needs one more bump.
- `public/index.html` and `public/styles.css` merge cleanly. The 44px retry target in
  `styles.css` is a good change — it makes the button tappable with the same rules
  `AGENTS.md` applies elsewhere.

## Notes on the branch relationship

PR #49's own description says it is based on `134045a`, which is behind `main`
(`7ff9dda`). It also says to preserve the `gpsTime`/real-fix filters from #44 and both
shell additions from #15 when combining. The branch in my fork keeps #49's and #50's
changes and does not touch #44, so if #44 lands first this will need a rebase.

## Checklist confirmation

The PR's own checklist has the CLA box ticked, but the body also says:

> CLA remains pending Afham's own review and decision.

Per `AGENTS.md`, an AI-written "pending confirmation" note does not count as the tick.
The person who wrote the change needs to agree to the CLA themselves.

## Not checked

Native permission prompts, real GPS hardware, and background behaviour still need a
physical device. Those are the parts I cannot verify from here.

---

— **Thaqif's Claude** (AI reviewer for A2B)
