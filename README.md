# lemonade-packaging

Builds Arch Linux packages with `makepkg`, indexes them into a pacman repository with
`repo-add`, and publishes the result from the `gh-pages` branch so GitHub Pages serves it
at `https://<owner>.github.io/lemonade-packaging/arch/`.

Everything runs in one workflow: [`.github/workflows/build-and-publish.yml`](.github/workflows/build-and-publish.yml).

## What your users run

```ini
# /etc/pacman.conf
[lemonade]
Server = https://<owner>.github.io/lemonade-packaging/arch/$arch
SigLevel = Never
```

```console
$ sudo pacman -Sy && sudo pacman -S <pkgname>
```

`$arch` is substituted by pacman, so one `Server` line covers every architecture you
publish (the workflow currently publishes `x86_64` only — see *Extending*).

## Repository layout you maintain

```
.github/workflows/build-and-publish.yml   # the pipeline
PKGBUILD                                  # the one and only, at the repository root
lemonade-sysusers.patch                   # sources the PKGBUILD applies in prepare()
lemonade-web-app.patch
```

There is exactly **one** `PKGBUILD` and it lives at the repository root: the workflow builds
that file and nothing else — no discovery step and no per-subdirectory filter. It is a split
`PKGBUILD` (`pkgbase=lemonade` → `lemonade-server` + `lemonade-desktop`), so that single
build still publishes several archives. `.git/` and the scratch directories — `src/` (the
checkout), `out/` (the built archives) and `site/` (the `gh-pages` clone) — are never
committed. Adding a second package means adding it to this PKGBUILD's `pkgbase`, not
creating another file.

## Published layout (the `gh-pages` branch)

```
.nojekyll                                  # stops Jekyll from mangling the tree
arch/index.html                            # human landing page + the pacman.conf snippet
arch/x86_64/lemonade.db          -> lemonade.db.tar.gz   # symlink created by repo-add
arch/x86_64/lemonade.db.tar.gz             # the package database pacman downloads
arch/x86_64/lemonade.files.tar.gz          # file listings for pacman -F
arch/x86_64/foo-1.2-3-x86_64.pkg.tar.zst   # the packages themselves
```

`REPO_NAME` (default `lemonade`) is the name of the database, i.e. the string inside the
`[...]` header of your users' `pacman.conf`. `PAGES_DIR` (default `arch`) is the directory
below the Pages root.

## Triggers

| Event | Behaviour |
| --- | --- |
| `push` to `main` touching `PKGBUILD`, `*.patch` or the workflow | build + publish |
| `workflow_dispatch` | build + publish; `rebuild_db` drops the database first, `lemonade_version` pins `DATE`/`COMMITS`, `force_bump` re-checksums without a version change |
| `pull_request` | the full pipeline runs (so a broken PKGBUILD is caught) but nothing is pushed |
| `schedule` (`30 16 * * 3`, Wednesdays 16:30 UTC) | follow upstream's release branch: bump `DATE` and `COMMITS`, rebuild, republish |

Keep the schedule *after* upstream's release cut — upstream branches `release-v<year>.<week>`
off `main` at 16:00 UTC on the Wednesday of the cut
([release.md](release.md), "Release Cadence and Channels"), and a run that fires before the
new branch exists resolves last week's release instead. `0 15 * * 4` (Thursday 15:00 UTC) is
the slot with a day of slack around the cut and is kept commented out in the workflow.

GitHub disables a schedule on a branch after 60 days without a commit; push anything or
re-enable it in Settings → Actions. The weekly bump commit is itself a commit to `main`, so
in practice the schedule keeps itself alive.

## Tracking the upstream release branch

Upstream ships date-based versions — `year.week.number`
([release.md](release.md), "Versioning") — so a new release appears every week and each one
needs its own build here. The root PKGBUILD carries two variables and derives the rest:

```bash
DATE=2026.39                                  # the release-v<DATE> branch to build
COMMITS=1                                     # commits added to it since it was cut
pkgver=${DATE}.${COMMITS}rc                   # -> 2026.39.1rc
```

`COMMITS` is upstream's `number`: commits added to the release branch after it was branched
from `main`, and `0` immediately after a cut. Keeping the pair here — rather than a full
version string — means one pair of values decides both the branch that gets cloned and the
version its packages are named.

Those lines are rewritten by the `Bump DATE and COMMITS…` step, which runs on the schedule
and on `workflow_dispatch` (never on `push`/`pull_request` — those build exactly what is
committed):

1. `git ls-remote --heads https://github.com/lemonade-sdk/lemonade.git refs/heads/release-v*`
   — `DATE` becomes the newest release branch, which is the release upstream is currently
   working on (a branch that ends up untagged is a skipped stable release, not a deleted
   branch, so the newest one is always the live one).
2. `GET /repos/lemonade-sdk/lemonade/compare/main...release-v<DATE>` → `total_commits`, taken
   as `COMMITS`. The call is unauthenticated because `GITHUB_TOKEN` is scoped to *this*
   repository and cannot read upstream's (two requests a week stay far under the 60 req/h
   limit).
3. If the pair already matches the PKGBUILD the step stops here — no rewrite, no re-checksum,
   no commit — unless `force_bump` is set. Otherwise both lines are rewritten and `updpkgsums`
   regenerates the `b2sums` array, because the `git+…#branch=${LEMONADE_RELEASE_BRANCH}` entry
   in it is hashed content: the moment the branch moves the checksum is wrong and `makepkg`
   refuses to build. The clone lands in `SRCDEST` (`$RUNNER_TEMP/srcdest`), which the build
   step reuses, so upstream is downloaded once per run.
4. Once the packages have been built and published, `Push the version bump` commits just
   `PKGBUILD` to the branch the workflow was running on. If the branch moved while the build
   was running it rebases the single commit and retries once; a genuine conflict aborts with
   an annotation and leaves the remote untouched.

Nothing is committed unless a build of the new version succeeded, so `main` never points at a
version this repository cannot build — a failed week is simply retried by the next run, which
resolves the newest release branch again (the same one if nothing was cut since).

The bump and the build have to share a run: a push made with `GITHUB_TOKEN` does **not**
trigger another workflow run, so a bump-only commit would otherwise sit un-built until some
other change landed. To take a release out of rotation, run the workflow by hand with
`lemonade_version` set to `year.week.number` (or `year.week`, which means `.0`);
`force_bump` rewrites and re-checksums the PKGBUILD without changing the version.

## Order of operations in one run

1. **Toolchain** — `pacman -Sy archlinux-keyring && pacman -Su base-devel git sudo` inside
   `ghcr.io/archlinux/archlinux:latest`.
2. **Build user** — `makepkg` refuses to run as root and that image defaults to root, so a
   `build` user is created with a `NOPASSWD` sudoers line (needed by `makepkg -s` to install
   makedepends), and the checkout is `chown`ed to it.
3. **Checkout** — plain `git` (`fetch --depth 1` + `checkout --detach`) instead of
   `actions/checkout`. JavaScript actions need a Node.js runtime inside the container and
   Arch images do not ship one. For `pull_request`, `refs/pull/N/merge` is fetched so the
   tested tree matches GitHub's merge preview.
4. **Check** — `src/PKGBUILD` exists. There is exactly one PKGBUILD, at the repository root,
   so nothing is discovered and no `package` filter is honoured; a missing file means the
   checkout is wrong and fails immediately instead of confusing `makepkg` minutes later.
5. **Build** — `makepkg -sc --noconfirm` on that PKGBUILD with `PKGEXT=.pkg.tar.zst` and
   `SOURCE_DATE_EPOCH` set to the commit time (reproducible archives, no needless churn).
   The split `pkgbase` still yields one archive per `pkgname`. A failing build fails the run
   and withholds the pending version bump.
6. **Restore** — clone `gh-pages` if it exists (`install -d site` otherwise) and
   `touch site/.nojekyll`. The database refers to package files by name, so the files
   already published have to be on disk before it is rewritten.
7. **Create / update the database** — `repo-add -p -R "$REPO_NAME.db.tar.gz" *.pkg.tar.*`.
   `repo-add` creates the `.db`/`.files` pair when it is missing and updates it in place
   otherwise. `-p` refuses to replace a newer published version; `-R` deletes archives that
   the new one supersedes. The `.db.old`/`.files.old` backups `repo-add` leaves behind are
   deleted. If the entries turn out identical to what is already published, the published
   bytes are restored — `repo-add` re-stamps its output with the current time, and without
   that step every scheduled run would create a commit.
8. **Verify** — extract every `%FILENAME%` from the database and fail if a listed archive is
   missing; delete archives no entry points at (e.g. a build `repo-add -p` refused), so the
   served tree and the database can never disagree.
9. **Index** — render `arch/index.html` with the `pacman.conf` snippet and links to every
   archive. Deliberately byte-stable so a no-op run stays a no-op.
10. **Publish** — `git add`, commit, `git push origin HEAD:refs/heads/gh-pages` — skipped
    for pull requests, skipped when the tree is byte-identical.
11. **Report** — a run-summary note with counts and the `Server` line.

## One-time GitHub setup

1. **Settings → Pages → Source: “Deploy from a branch”**, branch `gh-pages`, folder
   `/ (root)`. The workflow writes the `arch/` directory itself; it does not create the
   branch until the first successful run, so enable Pages *after* the first run, or create
   an empty `gh-pages` branch first — either order works.
2. **Settings → Actions → General → Workflow permissions: “Read and write”**. `GITHUB_TOKEN`
   cannot be granted more than this cap, and the publish push needs `contents: write`.
3. Optional: protect `gh-pages` from force-pushes. The workflow only ever fast-forwards
   (it clones the tip and adds one commit).

## Concurrent runs

`concurrency.group: arch-repo-publish` with `cancel-in-progress: false` serialises runs,
because each one rewrites the same branch. A run that starts while another is publishing
would otherwise be rejected by the remote as a non-fast-forward.

## Extending

* **Package signing** — add `--sign` to `makepkg` (needs a key in the runner) and `-s -k
  <keyid>` to `repo-add`, then ship the key and use `SigLevel = Required` for users. Store
  the private key in an Actions secret and import it in a step before the build.
* **Another architecture** — the job hardcodes `ARCH: x86_64` and `runs-on: ubuntu-latest`.
  Duplicate the job onto `ubuntu-24.04-arm` (native arm64 runners are free for public
  repos) with `ARCH: aarch64`; both jobs write into `arch/<arch>/`, so they touch disjoint
  paths. Give each job its own `concurrency` group if you do.
* **A second repository/database** — change `REPO_NAME`; users add a second `[repo]` block
  pointing at the same `Server` path.
* **`actions/deploy-pages` instead of pushing a branch** — the build job must then hand the
  tree to a second job on a runner that has Node.js, because `actions/upload-pages-artifact`
  and `actions/deploy-pages` are JavaScript actions and cannot execute inside the Arch
  container. The current single-job git-push design avoids that split; the upside of
  switching is that Pages keeps its history in GitHub's artifact store instead of a branch.
* **Cache** — wrap the build with `actions/cache` over `~/.cache/makepkg` (or
  `BUILDDIR`) if rebuilds get expensive; it is a JS action, so it would need the same
  two-job split, or `--skipinteg`-style local caching.

## Gotchas

* `SOURCE_DATE_EPOCH` makes archives reproducible, but `repo-add` does not honour it for its
  own timestamps — that is what the “unchanged” restore in step 7 is for.
* GitHub Pages serves `https://<owner>.github.io/<repo>/arch/x86_64/lemonade.db` because
  `repo-add` leaves `lemonade.db` as a symlink to `lemonade.db.tar.gz`; git stores it as a
  symlink and Pages serves it as a file. Both URLs work.
* The repository is unsigned (`SigLevel = Never`). pacman warns; that is expected until
  signing is wired up.
* `out/` and `site/` are scratch directories created inside the workspace by the job and
  are git-ignored. `site/` is its own git repository (a clone of `gh-pages`, or a fresh
  `git init -b gh-pages`), so the `git add -A` in the publish step can only ever stage
  files under it — nothing from your source tree can leak onto Pages.
