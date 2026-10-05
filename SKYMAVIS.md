# This fork

`skymavis/attendee` is Sky Mavis' fork of [attendee-labs/attendee][upstream]. It
exists so that the edits our deployment needs live in git rather than as
uncommitted changes in a working tree on the host, where a single `git pull`
reverts them silently and the breakage surfaces days later as something else.

Upstream is Elastic License 2.0. Forking and modifying it for our own
deployment is squarely within that licence; offering it to third parties as a
managed service is not.

## Branches

- **`main`** — an untouched mirror of upstream. Nothing of ours is committed
  here. Fast-forward it from `upstream/main`; never merge `skymavis` into it.
- **`skymavis`** — what our deployment runs: `main` plus the commits below,
  rebased onto `main` whenever we take an upstream update.

Keeping our work as a rebased branch rather than a merge means `git log
main..skymavis` is always the exact, readable list of what we changed, and a
conflict on rebase is the signal that upstream has moved under one of our edits.

```
git fetch upstream && git checkout main && git merge --ff-only upstream/main
git checkout skymavis && git rebase main
```

## What we changed, and why

Every edit carries a `lanh patch` comment at its site, so a reader who lands
there from a stack trace finds the reason without consulting this file.

- **`chore(deploy)`: `SITE_DOMAIN` and the worker host alias are configurable.**
  Both were hardcoded to localhost, so running behind a reverse proxy meant
  editing the tree in place. `SITE_DOMAIN` now reads its environment, matching
  what `ALLOWED_HOSTS` on the next line already did. The worker's `extra_hosts`
  alias is parameterised via `ATTENDEE_HOST_ALIAS` so that no deployment's
  hostname is committed to this repo — set it in the compose project's `.env`.
- **`fix(bots)`: the websocket server binds an ephemeral port.** Bots run as
  celery tasks in one shared container, and each one used to take a port from a
  fixed range of ten. Ports leaked on any death path that skipped `cleanup()`,
  the range filled, and after that a bot launched while another held the last
  free port died at startup — so a meeting overlapping another was joined only
  once the other bot left. Both this and the readiness-event change beside it
  are upstreamable; offering them is the cheapest way to shrink this branch.
- **`fix(bots)`: a Meet audio frame is labelled by who is contributing to its
  receiver now.** Meet gives the bot three continuous audio receivers, and the
  per-participant stream labelled each receiver's frame with the loudest source
  in `getContributingSources()` — a ten-second history, read from a cache Meet's
  own UI refreshes. So a speaker was stamped on two or three receivers at once
  and reached the consumer at up to two and a half times real time, mostly as
  another receiver's quiet audio under their name; downstream that overran a
  real-time recogniser until it stopped reading. The sources are now read live
  per frame, kept only if heard within a few packets of the newest, and a
  participant chosen on two receivers in the same instant is sent from one.
  Verifiable on any call from the consumer's side: per-participant frames per
  minute must not exceed real time while the total across labels stays at
  three receivers' worth. Likely upstreamable.
- **`fix(bots)`: `cleanup()` stops the virtual display.** Upstream starts an
  Xvfb per bot and never stops it, which is harmless with one bot per pod.
  Here every bot's Xvfb outlived its worker child and ran until the container
  restarted; about three hundred piled up over a month, a new one could no
  longer start, and every bot then died with `No reply from program Xvfb`
  before reaching the lobby. Verifiable on the host: once no bot is running,
  `ps -eo cmd | grep -c '^Xvfb'` is zero. Upstreamable.

## Deploying

The host tracks `skymavis`. The compose file bind-mounts the checkout into the
containers (`.:/attendee`), so **the working tree is the deployment** — pulling
changes what runs on the next restart, and an edit made in place is a deploy
nobody reviewed. Make changes here, push, pull there.

[upstream]: https://github.com/attendee-labs/attendee
