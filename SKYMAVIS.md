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

## Deploying

The host tracks `skymavis`. The compose file bind-mounts the checkout into the
containers (`.:/attendee`), so **the working tree is the deployment** — pulling
changes what runs on the next restart, and an edit made in place is a deploy
nobody reviewed. Make changes here, push, pull there.

[upstream]: https://github.com/attendee-labs/attendee
