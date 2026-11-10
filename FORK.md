# This fork: keeping it, building it

`necrustulum/horizon` is a fork with a few changes upstream does not take (offline play, the
telemetry/partner switches, Prism/ElyPrism preparation). Two workflows here keep it usable without
touching upstream's own files:

- `.github/workflows/fork-build.yml` — builds the launcher for Linux x86-64, Windows x64 and macOS
  arm64 on GitHub's runners, keeps them as artifacts, and publishes a `fork-*` tag as a prerelease.
- `.github/workflows/fork-sync.yml` — every night, fast-forwards `main` from
  `horizon-rust/horizon`.

## Branches

| Branch | What it is | Who changes it |
|---|---|---|
| `main` | a copy of `horizon-rust/horizon`'s `main`, no fork commits | `fork-sync` (or `gh repo sync`) |
| `custom` | `main` plus this fork's commits | you (and the agent), by merging `main` in |

Builds run on `custom` and on `fork-*` tags, never on `main`: a sync alone does not start a build.

### One-time setup

If `main` currently holds the fork's commits (it did before this file), move them to `custom` and
reset `main` to upstream:

```bash
git remote add upstream https://github.com/horizon-rust/horizon.git
git fetch upstream
git branch custom main                 # the fork's commits, kept
git reset --hard upstream/main         # main becomes a mirror
git push --force-with-lease origin main
git push -u origin custom
gh workflow run fork-sync              # make sure the nightly sync works
```

`--force-with-lease` on your own fork's `main` is safe here: the commits are on `custom` first.

## Sync

Automatic: `fork-sync` every night, or `gh workflow run fork-sync` by hand. It uses GitHub's
"merge upstream" (a fast-forward), so it fails if someone committed on `main`; that is the signal to
move those commits to `custom`.

Manual, in `custom`:

```bash
git fetch upstream
git switch custom
git merge upstream/main      # or: git rebase upstream/main
git push
```

`merge` is the easy habit: conflicts appear once per sync, in the files this fork edits:
`launcher/src/cli.rs`, `launcher/src/backend.rs`, `launcher/src/sources.rs`,
`crates/horizon-prepare/src/artifacts.rs`, `launcher/lang/*.json` and `docs/*`.

## Builds and releases

- Push to `custom` → the launcher for the three systems, as run artifacts (14 days).
- Tag `fork-YYYY.MM.DD` → the same builds as a prerelease with `SHA256SUMS`:

```bash
git switch custom && git push
git tag fork-2026.11.10 && git push origin fork-2026.11.10
```

macOS builds use `gpui_platform/runtime_shaders` (GitHub's macOS runners may lack the full Metal
toolchain): the launcher compiles its shaders at start, which works everywhere and is a little
slower on the first frame.

## What this fork does not build

- **The game.** `game-build.yml`/`nightly-build.yml` need the maintainer's own Minecraft jars and
  exporters on a self-hosted Mac, and the release signing key. This launcher installs the official
  Nightly instead (its checksums are signed with the key the launcher already carries).
- **Signed launcher updates.** No `HORIZON_LAUNCHER_TAG` is given to the build, so
  `self_update_allowed()` is false: the launcher keeps itself, and a newer fork build is a download
  from this repository's releases. Install one by unpacking it over the old binary.
- **The Windows setup/installer and the Linux AppImage.** The archives are the plain binaries;
  upstream's `launcher.yml` still has the setup exe and AppImage for an official release.

## Making a change

Do it on `custom`, keep it to a few commits with a clear message, and let the build check it. When
upstream changes the same lines, merge `main` in and resolve there — the merge commit is what makes
the next sync a fast-forward for `main` again.
