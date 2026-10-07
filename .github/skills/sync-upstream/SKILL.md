---
name: sync-upstream
description: Use when asked to sync Celestia with upstream (CelestiaProject), pull in the latest CelestiaContent, and propagate the resulting changes across the frontend repos (MobileCelestia/CelestiaCore, AndroidCelestia, CelestiaUWP) — including project-file updates for new/removed source files, Android CURRENT_DATA_VERSION bumps, and AndroidCelestia's versions.txt.
license: MIT
---

# Sync Upstream Celestia

Pulls `CelestiaProject/Celestia` upstream changes into the `celestiamobile/Celestia` fork's
`develop` branch, checks `CelestiaContent` is current, then propagates anything that affects the
frontends.

## Sibling repos

Assume these are checked out as siblings (i.e. `../Celestia`, etc.), same layout as the
`update-release-notes` skill:

- `../Celestia` — core engine/renderer. `origin` = `celestiamobile/Celestia` (the fork frontends
  build against), `upstream` = `CelestiaProject/Celestia` (may need adding with
  `git remote add upstream https://github.com/CelestiaProject/Celestia.git` if missing). Work
  happens on `develop`.
- `../CelestiaContent` — shared data/add-on content. `origin` = `CelestiaProject/CelestiaContent`
  directly (no fork/upstream split). Lives on `master`, not tagged per release.
- `../MobileCelestia` — Apple frontend (SPM app). References `../CelestiaCore` and
  `../CelestiaContent` at CI/build time, not via submodule.
- `../CelestiaCore` — Xcode framework wrapper consumed by MobileCelestia. `origin` =
  `celestiamobile/CelestiaCore`. Not pinned to a specific commit by MobileCelestia's workflow (CI
  checks out its default branch fresh each time).
- `../AndroidCelestia` — Android frontend (Gradle).
- `../CelestiaUWP` — Windows/UWP frontend (MSBuild/vcxproj).

## Step 1: Sync Celestia core

1. In `../Celestia`, check `git status --short` — if there's uncommitted local WIP unrelated to
   this sync, `git stash` it first and `git stash pop` at the end; never fold unrelated WIP into
   the sync commit.
2. `git fetch upstream --quiet` (add the remote first if it doesn't exist).
3. Preview what's incoming: `git log --oneline develop..upstream/master` and
   `git diff develop..upstream/master --stat`. Note any added/removed files under `src/` and any
   changes under `shaders/`, `fonts/`, `locale/`, `images/`, `scripts/` — these drive Step 3/4
   below.
4. `git merge upstream/master -m "Merge remote-tracking branch 'upstream/master' into develop"`.
   Resolve any conflicts normally; re-check the diff-stat above still holds for the merge result.
5. Verify no conflict markers remain (`git diff --check`), confirm `git status --short` is clean
   (modulo the stashed WIP), then `git push origin develop`.
6. Record the merge commit's short hash (`git rev-parse --short=7 HEAD`) — needed for Step 5.

## Step 2: Check CelestiaContent

`cd ../CelestiaContent && git fetch origin --quiet`. Since there's no fork/develop split here,
just compare `master` against `origin/master`. If `origin/master` is ahead, fast-forward
(`git merge --ff-only origin/master`) — this repo is not expected to carry local divergent commits.
Record `git rev-parse origin/master` (full hash) for Step 5.

## Step 3: Diff for frontend project-file impact

For the Celestia merge range (old `develop` HEAD..new `develop` HEAD), check specifically for
added/removed files, since Xcode/vcxproj project files list sources explicitly (no glob):

```
git diff <old>..<new> --diff-filter=AD --name-status -- src
```

If there are added/removed `.cpp`/`.h` files under `src/`:

- **CelestiaCore** (`../CelestiaCore/CelestiaCore.xcodeproj/project.pbxproj`): there's a single
  `PBXGroup` named `Core` with `path = ../Celestia/src` — its nested groups mirror the `src/`
  subdirectory tree (e.g. `celengine`, `celscript`), and file children are plain filenames
  (`path = foo.cpp; sourceTree = "<group>"`) resolved relative to their containing group. New
  files need: a new `PBXFileReference`, an entry in the matching subdirectory group's `children`,
  and (for `.cpp`/`.mm`) an entry in the `PBXSourcesBuildPhase`. Removed files need the reverse.
  There is no automated tool for this — edit `project.pbxproj` by hand carefully, or open the
  project in Xcode and add/remove the files via the UI, then diff the result.
- **CelestiaUWP**: check `*.vcxproj`/`*.vcxproj.filters` for explicit `<ClCompile Include=...>` /
  `<ClInclude Include=...>` entries mirroring `../Celestia/src`; add/remove similarly to the above.
- **AndroidCelestia**: the Android build invokes Celestia's own `CMakeLists.txt` as a subdirectory
  (via the NDK/CMake toolchain) rather than re-listing sources, so added/removed files are usually
  already handled by Celestia's own `CMakeLists.txt` changes merged in Step 1 — normally no
  AndroidCelestia-side file-list edit is needed, but double check if the upstream change touched
  build options/flags (see below).

Also check for compiler/CMake option changes (new `option()`/`target_compile_definitions` etc. in
Celestia's `CMakeLists.txt` files) that might need mirroring into AndroidCelestia's Gradle/CMake
args or CelestiaUWP's project property sheets.

## Step 4: Check for data/asset changes (AndroidCelestia CURRENT_DATA_VERSION)

Still using the Step 3 diff range, check content under `shaders/`, `fonts/`, `locale/`, `images/`,
`scripts/` in `../Celestia`, and any content-affecting change in `../CelestiaContent` since its last
pin. If there's a real data/shader update a running app needs to pick up (not just metadata/doc
changes):

- Bump `CURRENT_DATA_VERSION` in
  `AndroidCelestia/app/src/main/java/space/celestia/mobilecelestia/MainActivity.kt` (it's a string
  like `"182"` — increment it). This forces the app to redo its one-time data setup/copy on next
  launch.

If nothing under those paths changed meaningfully, leave `CURRENT_DATA_VERSION` alone.

## Step 5: Update AndroidCelestia versions.txt

`AndroidCelestia/versions.txt` tracks short (7-char) commit hashes for informational/reproducibility
purposes:

```
Celestia=<short hash>
CelestiaContent=<short hash>
CelestiaLocalization=<short hash>
apple-android-dependencies=<short hash>
```

Update the `Celestia=` line to the new `develop` HEAD short hash from Step 1, and `CelestiaContent=`
to the short hash from Step 2 (only touch the two lines relevant to this sync; leave
`CelestiaLocalization`/`apple-android-dependencies` unless they were also part of this sync).

## Step 6: Check CONTENT_COMMIT_HASH pins

All three frontend workflows (`AndroidCelestia/.github/workflows/build.yml`,
`MobileCelestia/.github/workflows/build.yml`, `CelestiaUWP/.github/workflows/build.yml`) pin
`CONTENT_COMMIT_HASH` to a `CelestiaContent` commit. If Step 2 found CelestiaContent had moved
(i.e. it wasn't already at the hash these pin), update `CONTENT_COMMIT_HASH` in all three workflow
files to the new `CelestiaContent` hash from Step 2 — they should always agree with each other.

## Step 7: Commit and push

Commit each repo's changes separately with a focused message (e.g. "Update versions.txt: Celestia
<hash>, CelestiaContent <hash>"), and push each to its own `develop`. Don't bundle unrelated
pre-existing local WIP into these commits — stash/restore it as in Step 1 if needed.
