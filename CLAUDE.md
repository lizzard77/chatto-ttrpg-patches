# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`lizzard77/chatto-ttrpg-patches` is a **patch overlay** on [Chatto](https://github.com/chattocorp/chatto), not a fork of its source. It holds only what is needed to build the modified Chatto (AGPL-3.0 corresponding source): `patches/*.patch` (`git am` format), the image build and the scripts to apply, build and export the patches. `CHATTO_VERSION` pins the upstream tag (`v<version>`) the patches apply to.

Everything else (bots, dev server, update automation, deployment, agent skills) lives in a separate private repository (Phil: `selfhosted-services/chatto/`, see its `CLAUDE.md`). Keep this repo minimal: **this repo is public** — no bots, tooling, deployment details, secrets or private IDs here.

Upstream source lives in `upstream/` (gitignored), a **git worktree** on branch `lizzard-patched` = upstream tag + one commit per patch. For anything inside `upstream/` (Go backend in `cli/`, SvelteKit frontend in `apps/frontend/`, protos, upstream tests), follow `upstream/AGENTS.md`.

## Patch workflow

Never edit files in `patches/` by hand. Edit code in `upstream/`, then export:

```sh
scripts/upstream.sh                # (re)create upstream/ = v$(cat CHATTO_VERSION) + patches (git am --3way); refuses if upstream/ is dirty
# edit + commit in upstream/ — one commit per patch; amend an existing patch with
#   git commit --fixup <commit> && git rebase -i --autosquash v<version>
scripts/export-patches.sh          # writes v<version>..HEAD back to patches/, keeping existing NNNN-name.patch filenames
```

Then commit `patches/` in the repo root. Constraint: **each patch must apply on its own and in any order** — the Docker build uses plain `git am` on a fresh clone, no `--3way`.

Rebasing onto a new upstream release: `scripts/upstream.sh <new>` → resolve conflicts in `upstream/` (`git am --continue`) → `CHATTO_VERSION=<new> scripts/build.sh --check` → `scripts/export-patches.sh <new>` → write `<new>` to `CHATTO_VERSION`.

## Build

```sh
scripts/build.sh            # pnpm install, build frontend, embed it, go build → upstream/cli/bin/chatto
scripts/build.sh --check    # additionally svelte-check (only meaningful after the frontend build)
```

`scripts/lib.sh` wraps `pnpm`: it runs the version pinned in `upstream/mise.toml` via `npx`, under Node 24 through `fnm` if installed. Go version also comes from `upstream/mise.toml`. `.dev/` (gitignored) is the state directory of the external dev-server script.

Known expected failures with the patches applied: upstream's `catalogs.spec.ts` (new i18n strings exist only in en-GB and de-DE). Also failing: two Slow-Mode/Markdown-editor tests in `MessageComposer.svelte.spec.ts` (fail upstream too) and `appearance.page.svelte.spec.ts` (panel count, from 0002).

## Patches (summary — details in README.md)

- `0001-iframe-sidebar-links` — sidebar links flagged via URL fragment `#chatto-embed` open in an iframe at `/chat/<server>/embed/<linkId>`. Frontend only.
- `0002-3d-dice` — dependency-free WebGL dice renderer in `apps/frontend/src/lib/dice/`, lazy-loaded, triggered by dice marker links in bot messages (the room pane also throws the rolls of its threads, e.g. the dice thread).
- `0003-image-annotations` — pins/highlighter on images, each annotation stored as a thread reply with a hidden marker link. Server part: `cli/internal/core/image_annotations.go` + one line in `messages.go`: "quiet" thread replies don't auto-follow the thread — annotation replies, dice bot replies (`#chatto-dice…` markers) and `!…` bot commands, so dice threads cause no notifications unless someone follows them on purpose.
- `0004-nachtgold-scheme` — "Nachtgold" colour scheme: surface tone `navy`, accent `gold`, and a `colorSchemes.nachtgold` preset in `userPreferences.svelte.ts` shown in App Preferences → Appearance. Frontend only.
- `0006-dice-tray` — floating dice bar above the composer: dice d4–d100, ±1, Advantage/Disadvantage (Daggerheart), Wild Die/Damage (SWADE; Wild Die default on for players, off for GMs, detected from `;gm=1` on the viewer's own rolls); **Send** puts the `!r …` into an empty composer and sends it (else posts directly). After any sent `!r`/`!roll` (clicked or typed) the bar shows the dice bot's answer: total with (H)/(F)/(!) or 👍/⬆/👎, coloured dice, modifier, "New roll". For GMs the bar also lists the latest roll of each other player (newest first, max. 5, new rolls flash). The 🎲 composer button toggles the bar per room; the results card above the composer shows only while the bar is hidden (per player for GMs). Code in `apps/frontend/src/lib/diceTray/`. Frontend only.
- `0008-deleted-threads` — upstream fix: the Threads page omits followed threads whose root is deleted and that have no replies left (the room-timeline part is fixed upstream since 0.5.0-beta.21, #3132). Frontend only. Reported upstream 2026-10-26.

0008 is meant to disappear: once upstream fixes the bug, drop the patch (`git am --skip` while rebasing, or `git rebase` it out of `upstream/`), export and update this list and README.md. `export-patches.sh` matches file names by subject, so the remaining patches keep their names. Dropped in 0.5.0-rc.2 because upstream covers them: `0005-room-links` (room autocomplete #3342, entity link pills #3124/#3331) and `0007-tombstone-backfill` (#3343); patch numbers stay as they are.

Fork features communicate through **marker links in message bodies** rather than new server APIs, so unpatched clients just see a plain link:
- Dice: `[🎲](#chatto-dice:<kind>.<sides>.<value>,…;by=<userId>)` — the bot ↔ frontend contract; full spec in README.md. Changing it means changing the dice bot too.
- Annotations: `[⌖](#chatto-annot:v1:<assetId>:pin|hl|mv:…)`, coordinates 0…10000.

## CI / image

`.github/workflows/images.yml` runs on pushes to `main` touching `CHATTO_VERSION` or `patches/` (or `gh workflow run images.yml`). It mirrors upstream's `build-image` job in `.github/workflows/release.yml`: check out the upstream tag → `git am` patches → upstream's `.github/actions/setup` → `mise build-frontend` → `go build` → image from upstream's `docker/Dockerfile.goreleaser`. No Dockerfile of our own. Tags: `ghcr.io/lizzard77/chatto:<version>`, `:<version>-<sha>` and `:latest` (the most recent build on `main`). The image build runs no upstream tests.

Only the steps "Frontend bauen" and "Docker-Kontext" are copied from upstream's `release.yml` (incl. the `go build -tags`); compare them on each update. `Dockerfile.goreleaser` runs `/chatto --version` as a smoke test.

## Conventions

- Talk to the user in German. Comments in scripts are German; README and this file are English.
- Commit as `Phil Reif <pr@sambal.net>`.
- Publishing a new version = commit directly on `main` and `git push origin main`; the push triggers the image build. No feature branches or PRs.
