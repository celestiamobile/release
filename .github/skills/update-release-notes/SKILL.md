---
name: update-release-notes
description: Use when asked to update, regenerate, or refresh the release-notes-*.txt files in this repo (upload-release) for a new Celestia release, by summarizing committed changes from the sibling Celestia repos since the last release.
license: MIT
---

# Update Release Notes

This repo (`upload-release`) holds the release notes shipped alongside Celestia builds:

- `release-notes-resources.txt` — core-only notes (data/engine), used for the resources package.
- `release-notes-android.txt` — AndroidCelestia notes.
- `release-notes-apple.txt` — MobileCelestia (iOS/iPadOS/macOS/visionOS) notes.
- `release-notes-uwp.txt` — CelestiaUWP (Windows) notes.

Each file is a simple numbered list (`1. ...`, `2. ...`, ...). When "updating" the notes, **replace
the full contents** of each file with only the items that are new since the last release — do not
carry over old items that were already part of a shipped release, and do not leave stale entries in
place.

## Sibling repos

Assume these repos are checked out as siblings of `upload-release` (i.e. `../Celestia`, etc.):

- `../Celestia` — core engine/renderer. Shared by all platforms.
- `../CelestiaContent` — shared add-on/data content (not code). Shared by all platforms. Not tagged
  per release.
- `../AndroidCelestia` — Android frontend.
- `../MobileCelestia` — Apple frontend (iOS/iPadOS/macOS/visionOS).
- `../CelestiaUWP` — Windows/UWP frontend.

`release-notes-resources.txt` only reflects `../Celestia` + `../CelestiaContent`. The other three
files reflect `../Celestia` + `../CelestiaContent` + their respective frontend repo.

## Step 1: Fetch latest

For each of the five repos, run `git fetch --all --tags --quiet` so tags/commits are up to date
before comparing. Only use **committed** history for the comparison — ignore any uncommitted or
untracked local changes in the working trees (`git status --short`), they must not appear in the
notes.

## Step 2: Find the last-release baseline commit per repo

- **AndroidCelestia / MobileCelestia**: find the latest release tag, named like `26_8_1_rc1`
  (`MAJOR_MINOR_PATCH_rcN`, underscores). Use `git tag -l | sort -V | tail`. This tag is the
  baseline.
- **CelestiaUWP**: find the latest release tag, named like `26.8.1.0` (dotted version). This tag is
  the baseline.
- **Celestia (core)**: this repo is usually **not** tagged for every frontend release (e.g. it may
  only have `26_8_0_rc1` while the frontends are already at `26_8_1_rc1`). The frontend workflows
  (`.github/workflows/build.yml`) check out `celestiamobile/Celestia` without pinning a ref, so the
  real baseline is whatever commit was `HEAD` of Celestia's default branch at the moment the
  frontend release tags were cut. To find it:
  1. Get the tag creation timestamps of the frontend release tags (`git log -1 --format=%ai <tag>`
     in AndroidCelestia/MobileCelestia/CelestiaUWP — they should all be within minutes of each
     other).
  2. In `../Celestia`, find the last commit on the default branch at or before that timestamp
     (`git log --format="%h %ai %s"` and compare timestamps, accounting for author timezone
     offsets). That commit is the baseline — **do not** just use the latest core tag, it is likely
     stale and will cause already-shipped changes to be re-included.
- **CelestiaContent**: not tagged. Find the pinned reference in each frontend's
  `.github/workflows/build.yml` at its latest release tag, e.g.:
  `CONTENT_COMMIT_HASH: '94ae7673d7dd615acc3dbc483f2a7304099b2ad8'`. All three frontends should
  reference the same content commit hash for the same release — use it as the baseline.

## Step 3: Diff committed changes

For each code repo, run `git log --oneline --reverse <baseline>..HEAD` to list committed changes
since the baseline. For `CelestiaContent`, use `<baseline>..HEAD` for
`release-notes-resources.txt`, but use `<baseline>..<current-content-pin>` for each platform file,
where `<current-content-pin>` is the `CONTENT_COMMIT_HASH` in that frontend's current
`.github/workflows/build.yml`. Read commit bodies (`git log -1 --format=%B <hash>`) and
`git show --stat` for ambiguous ones to understand user-facing impact.

## Step 4: Filter to user-facing changes only

Only keep entries that a user would notice in the app/build, for example: bug fixes affecting
rendering/behavior, new user-facing features or settings, new OS/platform support. Drop:

- Merge commits, CI/build-system tweaks, dependency bumps, version bumps.
- Pure internal refactors with no behavior change (check `git show --stat`/diff if the commit
  message is ambiguous, e.g. "Share renderer settings across apps" moving code between modules with
  no UI change).
- Changes scoped to a platform/frontend not covered by the file being written (e.g. Qt-desktop-only
  UI changes in `../Celestia` don't apply to mobile/UWP notes).
- Doc-only or link-only changes in `../CelestiaContent` (e.g. switching links to https) unless
  explicitly asked to include them.

Consolidate multiple related commits into one concise bullet (e.g. several "iOS 27 support" commits
across Catalyst/UIKit/device-motion become one line). If a change references an upcoming/unreleased
OS version, phrase it generically (e.g. "Add support for upcoming system") rather than naming the
specific unreleased OS version.

## Step 5: Write the files

Always include a first "Data update (<date>)" line. For each platform file, read
`CONTENT_COMMIT_HASH` from that frontend's current `.github/workflows/build.yml`, then use the date
of that commit in `../CelestiaContent`. For `release-notes-resources.txt`, use the date of
`../CelestiaContent` HEAD. Then list the filtered, consolidated changes in the numbered format
`1. ...`, `2. ...`, etc., matching the existing style in the files. Overwrite each file entirely —
don't append to old content.

## Step 6: Sanity check

Diff the four files against each other. Shared core items should read identically across all four
files. Content items and data-update dates in each platform file must match the
`CONTENT_COMMIT_HASH` pinned by that frontend's current workflow; when all three pins equal
`CelestiaContent` HEAD, the resources items must be a subset of every platform file and shared
core/content items should read identically.
