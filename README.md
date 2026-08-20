# mirror-hub

[![Mirror sync](https://github.com/aboutrv-com/mirror-hub/actions/workflows/sync.yml/badge.svg)](https://github.com/aboutrv-com/mirror-hub/actions/workflows/sync.yml)
[![Mirrors](https://img.shields.io/badge/mirrors-6-brightgreen)](./mirrors.txt)
[![Sync cadence](https://img.shields.io/badge/sync-daily-blue)](#daily-cadence)
[![Maintained by GitHub Actions](https://img.shields.io/badge/maintained%20by-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](./.github/workflows/sync.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](./LICENSE)

Automated, self-maintaining Git mirrors, powered entirely by GitHub Actions.
No external server, no long-lived credentials, no manual upkeep.

A single daily workflow reads a config file, and for each listed repository
pulls from upstream and pushes an exact copy into a GitHub repo of your own.

> **About this project.** Most of the implementation here was written with the
> help of an LLM. The entire mirroring process is open source and runs
> automatically in GitHub Actions — the config, the workflow, and every step it
> takes are visible in this repo and reproducible by anyone. The goal is to keep
> ongoing hand-maintenance to a minimum and to avoid depending on opaque,
> external mirror mechanisms whose behavior you can't inspect or control, which
> are a recurring source of instability.

## Mirrors

| Mirror | Upstream source |
| --- | --- |
| [`aboutrv-com/dejagnu`](https://github.com/aboutrv-com/dejagnu) | [git.savannah.gnu.org/dejagnu](https://git.savannah.gnu.org/git/dejagnu.git) |
| [`aboutrv-com/binutils-gdb`](https://github.com/aboutrv-com/binutils-gdb) | [sourceware.org/binutils-gdb](https://sourceware.org/git/binutils-gdb.git) |
| [`aboutrv-com/glibc`](https://github.com/aboutrv-com/glibc) | [sourceware.org/glibc](https://sourceware.org/git/glibc.git) |
| [`aboutrv-com/newlib-cygwin`](https://github.com/aboutrv-com/newlib-cygwin) | [sourceware.org/newlib-cygwin](https://sourceware.org/git/newlib-cygwin.git) |
| [`aboutrv-com/musl`](https://github.com/aboutrv-com/musl) | [git.musl-libc.org/musl](https://git.musl-libc.org/git/musl) |
| [`aboutrv-com/gcc`](https://github.com/aboutrv-com/gcc) | [gcc.gnu.org/gcc](https://gcc.gnu.org/git/gcc.git) |

The source of truth is [`mirrors.txt`](./mirrors.txt); this table is a
human-readable snapshot of it.

## How it works

Each line in [`mirrors.txt`](./mirrors.txt) maps a source repo to a destination:

```
<source-git-url>   <owner/name>
```

Once a day (and on demand), [`.github/workflows/sync.yml`](./.github/workflows/sync.yml):

1. Parses `mirrors.txt` into a job matrix — one independent job per mirror
   (`fail-fast: false`, so one broken source never blocks the others).
2. Restores the cached bare clone, then fetches incrementally
   (`git remote update --prune`). Only the very first run does a full
   `git clone --mirror`.
3. Creates the destination repo automatically if it doesn't exist yet.
4. Pushes `refs/heads/*` and `refs/tags/*` with `--force --prune`, so the
   destination becomes an exact copy of upstream.
5. Sets the destination's default branch to match upstream `HEAD`.

Failures are harmless by design: the next daily run retries, and the cached
clone persists, so a transient upstream/GitHub hiccup costs nothing.

## Design notes

**Exact copy, not `push --mirror`.** Some upstreams carry non-standard refs
(`refs/remotes/*`, leftover import refs) that GitHub's receive endpoint rejects.
We mirror only real branches and tags (`refs/heads/*`, `refs/tags/*`) with
`--force --prune` — new refs added, stale refs removed, upstream rewrites
followed — without dragging along junk refs.

**Short-lived credentials.** A GitHub App issues an installation token at
runtime (valid ~1h), so nothing long-lived is ever stored or needs rotating.
The App is installed on the destination owner with permission to create repos
and push. Auth is by App **client-id** + private key, held as repo secrets.
The token is minted *after* the clone/fetch step, so its full ~1h lifetime is
reserved for the push — a slow cold clone of a large repo (gcc, glibc) can't
outlive a token minted up front.

**Malformed historical objects.** Old CVS-era imports (dejagnu, binutils-gdb)
can contain objects GitHub's `fsck` rejects — e.g. an annotated tag whose
tagger line has no date. GitHub rejects the *entire* pack, failing every ref at
once. The workflow detects this, falls back to pushing refs one at a time, and
**salvages** a rejected annotated tag by re-pushing it as a lightweight tag
pointing at the same (healthy) commit. The ref still mirrors and points to the
right revision; only the tag's own message/signature is dropped. A ref that
still can't be pushed is skipped with a warning rather than failing the sync.

**Mirroring workflow files.** Upstreams often keep their own
`.github/workflows/*.yml` on some branches (gcc's `devel/rust/master`,
newlib-cygwin's `main`/`master`/`topic/*`). GitHub refuses to let an App token
push any commit that creates or updates a file under `.github/workflows/`
*unless the App has the **Workflows: write** permission* — and one rejected ref
fails the whole batch push (it's not an fsck error, so the per-ref salvage path
doesn't catch it; the batch just retries and dies). So the App must be granted
**Workflows: Read and write** (see setup). To make sure those mirrored workflow
files never actually *run* in our copy, the sync step disables Actions on every
destination repo (`PUT /repos/{dst}/actions/permissions {enabled:false}`,
idempotent, run every sync). The workflow files exist as inert copies — the
mirror is exact, but it never executes upstream CI or burns our Actions minutes.

**No native "mirror" badge.** GitHub's "mirror of &lt;url&gt;" label under a repo
name is driven by a read-only `mirror_url` field, set only by GitHub's internal
importer — there's no API or settings page to set it on a repo we push to
ourselves. The closest self-serve signal is repo topics, so the sync step tags
each destination with `mirror` + `unofficial-mirror` (`PUT /repos/{dst}/topics`,
idempotent, run every sync), sets the repo's Website field (`homepage`) to the
upstream source URL so the About box links back to the real repo, and sets a
description that names this project as the automated maintainer
(`Unofficial mirror · auto-synced by https://github.com/<owner>/<repo> · not
affiliated with upstream`). The maintainer URL keeps its `https://` scheme
because the About box autolinks full URLs in the description — so it renders as
a clickable link back to this hub, in addition to the upstream Website link.
The homepage and description PATCH is idempotent and runs every sync, so
existing mirrors are backfilled with the current wording. We never write our own files into a
destination — it stays an exact copy of upstream — so the mirror signal lives in
metadata, not in the tree.

**HTTP/1.1 for pushes.** Forcing HTTP/1.1 avoids sporadic HTTP/2 500s that
GitHub returns on large ref-update batches.

**Fallback source + backoff for fetches.** sourceware.org frequently answers
`HTTP 429` when a matrix run cold-clones several of its repos at once. Each
mirror may list an optional fallback URL (3rd column in `mirrors.txt`) — usually
a `git://` URL for the same repo, whose daemon (port 9418) doesn't share the
HTTPS rate limiter. The fetch step retries each URL with exponential backoff
(30s/60s/120s) before falling through to the fallback; if all fail, the mirror
is skipped and retried next run.

**Incremental by default.** The bare clone lives in the Actions cache and is
updated in place; a full clone happens only on a cache miss.

**Observable pushes.** Large pushes (gcc, glibc) used to run silently for
minutes because output was buffered until completion. Now the push streams
live: `git --progress` transfer heartbeat is throttled to a periodic line (not
a per-object flood), the per-ref fallback prints a `N/total` heartbeat every
100 refs and the real rejection reason on failure, and each namespace ends with
a `pushed / salvaged / skipped` summary. The manual run has a **debug** toggle
that enables `GIT_TRACE` (kept off by default; `GIT_CURL_VERBOSE` is never used
because it would leak the auth token into the log).

## Adding a mirror

Add one line to `mirrors.txt` (`<source-url> <owner/name> [fallback-url]`),
commit, and push. The optional 3rd column is a fallback source URL (e.g. a
`git://` URL) used when the primary keeps failing. The destination repo is
created on the next sync. To stop mirroring, delete the line (the destination
repo is left untouched — delete it manually if you want).

## Daily cadence

The schedule is daily (`17 3 * * *`, off-peak UTC). Daily is deliberate:

- Upstreams like binutils-gdb/dejagnu see at most a handful of commits a day;
  a mirror has no minute-level freshness requirement.
- A daily run keeps the Actions cache warm (caches expire after 7 idle days),
  avoiding a slow full re-clone.
- More frequent runs multiply Actions minutes and, for large repos, the risk of
  cache eviction (10 GB/repo limit) triggering full re-clones.

## One-time setup

1. **Create a GitHub App** (owner: the account/org that will host the mirrors).
   Repository permissions:
   - Contents: **Read and write**
   - Administration: **Read and write** (to auto-create destination repos and
     disable Actions on them)
   - Workflows: **Read and write** (upstreams carry `.github/workflows/*.yml` on
     some branches; without this, GitHub rejects those refs and the push fails)
   - Metadata: **Read-only**

   Changing an already-installed App's permissions requires re-approval: the
   owner must approve the new permission in the org/account's installed-Apps
   settings, or the added scope stays inactive.

   For an organization owner, also grant **Organization → Administration:
   Read and write**, otherwise repo auto-creation fails with
   `Resource not accessible by integration (createRepository)`.
2. **Install** the App on that owner, "All repositories".
3. Add two secrets to this hub (Settings → Secrets and variables → Actions):
   - `MIRROR_CLIENT_ID` — the App's Client ID
   - `MIRROR_APP_KEY` — the App's generated private key (full `.pem` contents)
4. Enable Actions, then run **Mirror sync** once manually to verify.

The first clone of a large repo (e.g. binutils-gdb) can take 20–40 min;
subsequent runs are incremental and finish in minutes.

## Notes

- All mirrors are unofficial and not affiliated with their upstreams.
- The workflow only pushes to the destination repos, never to this hub, so the
  hub's own contents are always safe.

## License

This hub's own code — the workflow, config, and docs — is released under the
[MIT License](./LICENSE). The license covers only this repository. Each
mirrored repository retains the license of its upstream project; mirroring does
not relicense anyone's code.
