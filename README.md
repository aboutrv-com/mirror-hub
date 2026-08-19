# mirror-hub

Automated, self-maintaining Git mirrors, powered entirely by GitHub Actions.
No external server, no long-lived credentials, no manual upkeep.

A single daily workflow reads a config file, and for each listed repository
pulls from upstream and pushes an exact copy into a GitHub repo of your own.

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
   - Administration: **Read and write** (to auto-create destination repos)
   - Metadata: **Read-only**

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
