# Syncing this fork with upstream

This checkout is a fork of [Swofty-Developments/SwoftyLauncher](https://github.com/Swofty-Developments/SwoftyLauncher)
that carries one feature on top of upstream. Everything below keeps that feature
and picks up upstream's releases.

| Remote | Repository |
|---|---|
| `origin` | `https://github.com/olivia-hye/SwoftyLauncher` (the fork) |
| `upstream` | `https://github.com/Swofty-Developments/SwoftyLauncher` |

## What this fork adds

`updates.importer`, a config switch that separates the Simplified Importer's
release check from the launcher's own update check, so either can be turned off
alone. It is one commit (`fcb3054`) touching:

```
packages/core/src/update.rs        UpdateConfig.importer
apps/launcher/src/update.rs        tick_updates(), update_bar()
apps/launcher/src/shell.rs         Shell::importer_check
docs/reference/cli.md              the "updates" key table
.github/workflows/build-windows.yml  manual Windows build
```

`config.json` keeps working for upstream too: `importer` defaults to `true`, so
a config without it behaves exactly like upstream.

## Before you sync

1. Be on `main`, with nothing half-finished:

   ```sh
   git switch main
   git status
   ```

2. Park local changes you do not want to commit or push:

   ```sh
   git stash push -u -m "before upstream sync"
   ```

   `-u` includes untracked files. Anything left in the working tree during a
   rebase will block it.

## Fetch upstream

This clone is **shallow** (`git rev-parse --is-shallow-repository` prints
`true`), which makes rebases onto new upstream commits unreliable: the fetched
tip has no ancestry, so git cannot find a merge base. Fix it once:

```sh
git fetch --unshallow upstream
```

That downloads upstream's full history one time (the shallow `.git` here is
about 28 MB; expect a few hundred MB for the whole history).

If you would rather stay shallow, deepen instead and re-deepen each time git
complains about missing history:

```sh
git fetch --deepen=100 upstream main
```

Then fetch the branch:

```sh
git fetch upstream main
```

## Rebase the fork onto upstream

```sh
git rebase upstream/main
```

- Conflicts will almost certainly be in `apps/launcher/src/update.rs` or
  `packages/core/src/update.rs` if upstream reworks the update code. Re-apply
  the intent, not the old lines: `updates.check` gates launcher releases,
  `updates.importer` gates the importer check, the update bar shows when either
  is on.
- Abort and start over at any point:

  ```sh
  git rebase --abort
  ```

- Prefer `rebase` over `merge` here: it keeps the fork at "upstream plus one
  commit", which is what makes the next sync a fast-forward for everything but
  your own commit.

Confirm only your files differ:

```sh
git diff --stat upstream/main HEAD
```

## Push

The rebase rewrites `main`, so the push is forced. Fetch first, or the lease
check reports `stale info`:

```sh
git fetch origin main
git push --force-with-lease origin main
```

Never force-push `upstream` — it is read-only for you.

If you push through a token URL instead of the `origin` remote, `--force-with-lease`
has no tracking information and is rejected; name the old tip explicitly:

```sh
git push --force-with-lease=main:<old-origin-main-sha> <url> main:main
```

## Rebuild and reinstall

Pushing to `main` runs nothing here: this fork deleted `ci.yml` and `pages.yml`,
so only the manual Windows build is left — **Build Windows binaries** →
**Run workflow** on `main`, or:

```sh
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/olivia-hye/SwoftyLauncher/actions/workflows/build-windows.yml/dispatches \
  -d '{"ref":"main"}'
```

Then, with the launcher closed, replace the two binaries from the
`windows-binaries` artifact:

```
<install>\bin\swofty.exe
<install>\bin\swofty-launcher.exe
```

Leave `loader\`, `dxc\` and `bin\swofty-ui.exe` alone unless upstream changed
them: only `swofty.exe` and `swofty-launcher.exe` are produced here. If the
release notes mention a new loader, install the official release first and then
overlay these two files.

## Restore your local changes

```sh
git stash pop
```

If `config.json` lives in a data folder outside the repo (for example
`W:\SteamLibrary\steamapps\common\Swofty\config.json`), a sync does not touch
it. But upstream can raise `DATA_VERSION` (`packages/core/src/session.rs`),
which migrates the data folder on first start — back the data folder up before
running a newly built launcher.
