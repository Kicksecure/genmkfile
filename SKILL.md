---
name: genmkfile
description: "Use when building Kicksecure/derivative-maker Debian packages with genmkfile (targets like 'genmkfile deb-pkg', cowbuilder chroot builds, the output/dist folder as a .deb cache, dist/debdist/debdsc tarballs, deb-install, lintian, debsign). Covers the build sequence, env-var knobs, cowbuilder base creation, and override hooks."
license: MIT
---

# genmkfile: building derivative-maker / Kicksecure Debian packages

- `genmkfile`: generic Makefile generator for Kicksecure / Whonix / derivative-maker packages.
- Replaces hand-written `debian/*.install` and `make install` targets.
- Files laid out as `usr/...`, `etc/...` in the package root install to `/usr/...`, `/etc/...`.
- Engine: big bash script `/usr/share/genmkfile/make-helper-one.bsh`.
- Thin `/usr/bin/genmkfile` wrapper locates `GENMKFILE_PATH`, relative to the WRAPPER's own location (not your CWD): `./packages/kicksecure/genmkfile/usr/share/genmkfile` in a dm checkout, else `./usr/share/genmkfile`, else `/usr/share/genmkfile`.
- Run it from a package's root (the dir containing `debian/`).

## Build sequence (cowbuilder)

- `genmkfile deb-pkg` is the whole build. `make-helper.bsh` intercepts it and runs, in
  order: `deb-cleanup`, `dist` (upstream tarball; excludes `.git` and `./debian`),
  `debdist` (debian tarball), `debdsc` (`.dsc`), then `deb-pkg-build` once per entry in
  `make_cross_build_platform_list`.
- So do NOT run `dist`/`debdist`/`debdsc` yourself first: `deb-pkg` starts with
  `deb-cleanup`, which DELETES those artifacts, then regenerates them. Running them by
  hand is wasted work, not a prerequisite.
- The "does not exist. Did you run genmkfile dist ..." errors come from
  `deb-pkg-build`, which is the target to use when you deliberately want the build step
  alone.
- `genmkfile distclean` is currently the SAME as `clean`: it runs the Makefile's `clean`
  target and nothing else. It does not remove tarballs, and it does not remove debhelper
  residue -- that is `deb-clean` / `deb-cleanup`.
- Plain `genmkfile deb-pkg` with no cowbuilder uses `debuild` in-place (needs build-deps installed; `sudo genmkfile deb-all-dep` installs them).

### Prefer cowbuilder

- Build in a cowbuilder chroot (see below). The chroot declares its own build-deps, so
  the `.deb` does not depend on what happens to be installed on the build host, and the
  build is reproducible on another machine.
- If the base is missing, create it (`Create the cowbuilder base`, below) rather than
  building without it.

### In-place build (no chroot) -- what it does

Reference for the behaviour of `genmkfile deb-pkg` when `make_use_cowbuilder` is unset.

- The build runs `debuild` against the HOST's installed packages, so the result reflects
  host state; build-deps must already be present (`sudo genmkfile deb-all-dep`).
- Artifacts land in `../`, not in a dist folder under the package root.
- It writes debhelper residue into the source tree (`debian/.debhelper`,
  `debian/*.substvars`, `debian/files`, `debian/debhelper-build-stamp`, and a
  `debian/<package-name>` staging dir). `genmkfile dist` then rejects the tree with
  "Empty directory found!". `genmkfile deb-cleanup` clears it; `genmkfile distclean`
  does not (it only runs the Makefile's `clean`).
- cowbuilder does not hit the residue problem -- it builds from the source tarball
  inside the chroot.
- Lintian is fatal by default on ANY captured output (even an info note -- default opts add
  `--pedantic --info --display-info`), and fires AFTER the `.deb` already exists; see the
  `make_use_lintian` knob below.

## cowbuilder mode (chroot build)

- **In a derivative-maker checkout, do NOT hand-roll the cowbuilder invocation.**
- Use dm's own build steps: they set up the base, the approx mirror, the pbuilder config, and the TMPDIR removal canonically (and call the dm-submodule genmkfile, not the installed one):

```
make_cross_build_platform_list=amd64 \
  ./build-steps.d/*_cowbuilder-setup       --allow-untagged true --allow-uncommitted true --flavor source --target root
make_cross_build_platform_list=amd64 \
  ./build-steps.d/*_create-debian-packages --allow-untagged true --allow-uncommitted true --flavor source --target root
```

- dm handles libpam-tmpdir/pbuilder breakage via `help-steps/variables` (`chroot_env_vars_var_remove_list="TEMP TEMPDIR TMP TMPDIR"`).
- dm sets the mirror via the approx apt-cacher on `127.0.0.1:9977` (`APPROX_PROXY_ENABLE=yes`).
- dm auto-generates the pbuilder config to `~/derivative-binary/pbuilder.conf`.
- dm's `2100` step only iterates dm's own package set + does `reprepro-add`.
- Raw knobs below are for understanding / building a standalone package OUTSIDE a dm checkout.

### Building standalone packages WITH dm's cowbuilder env

- For packages NOT in dm's package set (so dm's `2100` won't iterate them) but that should still build with dm's machinery: reuse dm's base + config instead of hand-rolling.
- Run dm's setup ONCE, then build each package via genmkfile pointed at the dm pbuilder config:

```
# once: dm build machine + base (1200 brings up the approx 9977 mirror; 1300
# creates the base whose in-chroot `mkdir $TMPDIR` fixes the libpam-tmpdir break)
cd ~/derivative-maker
./build-steps.d/1200_prepare-build-machine       --allow-untagged true --allow-uncommitted true --flavor source --target root
make_cross_build_platform_list=amd64 ./build-steps.d/1300_cowbuilder-setup --allow-untagged true --allow-uncommitted true --flavor source --target root

# per package (in its root): the dm config is the ONLY extra knob -- no
# COWBUILDER_PREFIX, no TMPDIR=/tmp override
export make_use_cowbuilder=true make_use_lintian=false
export make_cowbuilder_dist_folder=<dist-folder>   # where built .debs land
export dist_build_pbuilder_config_file=~/derivative-binary/pbuilder.conf
genmkfile deb-pkg   # runs deb-cleanup, dist, debdist, debdsc, deb-pkg-build itself
```

Gotchas learned:
- dm's `1300` fails standalone WITHOUT `1200` first (debootstrap).
- The TMPDIR break is handled by genmkfile's own build-time `env --unset` (see "Build facts"), independent of dm's base or any config -- do not add a TMPDIR workaround on top.
- `genmkfile dist` errors "Empty directory found!" for TWO different reasons -- check which:
  - A genuinely empty dir in the source tree: fix with an executable `./make-helper-overrides.bsh` defining `make_dist_hook_pre` to drop a placeholder -- don't move the package off genmkfile.
  - Build residue from a previous in-place `debuild`: `debian/.debhelper/generated/_source/home` is left behind empty. Clear it with `genmkfile deb-cleanup` (runs `debian/rules clean` plus the debhelper-file removal). `genmkfile distclean` does NOT remove it -- it only runs the Makefile's `clean` target. Note `deb-pkg` already runs `deb-cleanup` first, so this usually self-corrects on the next build.
- A small wrapper that loops over the package dirs keeps a multi-package build DRY.

### Raw genmkfile cowbuilder knobs (standalone, outside dm)

- Enable with env vars (the README knobs):

```
make_use_cowbuilder=true \
make_cowbuilder_dist_folder=<dist-folder> \
dist_build_pbuilder_config_file=~/derivative-binary/pbuilder.conf \
make_cross_build_platform_list=amd64 \
genmkfile deb-pkg
```

- `make_use_cowbuilder=true` -- build inside a cowbuilder chroot.
- `make_cowbuilder_dist_folder=<dir>` -- REQUIRED with cowbuilder; the **output / dist folder** where the built `.deb` / `.changes` / `.dsc` land (`--buildresult`). This folder is the natural **package cache**. Must already exist.
- `make_cross_build_platform_list='amd64 arm64'` -- one build per arch.
- `make_cowbuilder_distribution` -- defaults to `$dist_build_apt_stable_release` or `lsb_release -sc` (e.g. `trixie`).
- `make_cowbuilder_mirror` -- apt mirror for the chroot; derivative-maker sets it to its apt-cacher (`http://127.0.0.1:9977/debian`); falls back to `https://deb.debian.org/debian`. (In dm, `buildconfig.d` exports it.)
- `dist_build_pbuilder_config_file` -- `--configfile` for cowbuilder/pbuilder.
- `make_use_lintian=true|false` -- **fails closed on a WARNING, not just an error.** Any lintian output at all (the default opts add `--pedantic --info --display-info`) reaches `make_lintian_on_warning`, which exits non-zero unless `make_lintian_fail_open_versus_closed=open`. It runs AFTER the `.deb` is already built, so the package exists in the dist folder even though `deb-pkg` "failed" -- check for the artifact before rebuilding. Unset means autodetect: lintian runs if installed. Use `make_use_lintian=false` for a build you just want the `.deb` from.
- `make_use_debsign=true` + `make_debsign_opts` (debsign with `sq verify` against `$DEBEMAIL`).
- genmkfile passes to cowbuilder: `--build <dsc> --basepath /var/cache/pbuilder/base.cow_<arch> --buildplace /var/cache/pbuilder/cow.cow_<arch> --buildresult <tmpdir> --debbuildopts=-sa`, where `<tmpdir>` is a fresh temp dir under `genmkfile_temp_dir`; only on success does genmkfile `mv` the artifacts into `<dist_folder>` (a killed/failed build leaves nothing in the dist folder). **It does NOT create the base** -- create it once first.

### Create the cowbuilder base (once per arch)

- cowbuilder `--build` needs `--basepath` to already exist. Create it:

```
sudo cowbuilder --create \
  --basepath /var/cache/pbuilder/base.cow_amd64 \
  --distribution trixie \
  --architecture amd64 \
  --extrapackages aptitude \
  --mirror https://deb.debian.org/debian \
  [--configfile ~/derivative-binary/pbuilder.conf]
```

- Update it later with `--update --basepath ...`.
- **`--extrapackages aptitude` is REQUIRED.** pbuilder's DEFAULT satisfydepends is the shipped
  `/usr/lib/pbuilder/pbuilder-satisfydepends` -> `-aptitude` symlink, and aptitude is NOT a
  pbuilder dependency, so a fresh debootstrap base lacks it: without the flag a `deb-pkg` (with no
  config that swaps in the native-apt resolver) dies `env: 'aptitude': No such file` ->
  `E: pbuilder-satisfydepends failed`. A build passing dm's `pbuilder.conf`
  (`PBUILDERSATISFYDEPENDSCMD=.../pbuilder-satisfydepends-apt`) avoids it; a RAW `genmkfile
  deb-pkg` with no `dist_build_pbuilder_config_file` falls to the aptitude default. (The safe tool
  `ensure-cowbuilder-base`, which boot-durable-deb uses, already passes the flag.)
- Build-deps (e.g. `debhelper`) must be reachable from `--mirror`; the package's *runtime* deps are NOT needed at build time.
- So `Architecture: all` packages whose only build-dep is debhelper build against a plain `deb.debian.org` base even if runtime deps (signal-cli, etc.) live elsewhere.
- `libpam-tmpdir` (which sets `TMPDIR=/tmp/user/0`) is handled by genmkfile's build-time
  `env --unset` (see "Build facts") for `cowbuilder --build`; the `--create` path under `sudo`
  is NOT covered and needs the same unset itself. It can otherwise stay installed on the build host.

## Other useful targets

- `install` / `uninstall` (to a live system or `DESTDIR`).
- `installsim` / `uninstallcheck` (dry runs).
- `deb-install` (install the ALREADY-BUILT .deb from the dist folder, then `installcheck`). Build-deps are `deb-build-dep` / `deb-all-dep`.
- `deb-icup` (install just-built current package).
- `deb-remove` / `deb-purge`.
- `deb-clean` (delete temporary debhelper files) / `deb-cleanup` (runs the Makefile `clean` AND `debian/rules clean` AND `deb-clean`, then deletes that package's artifacts from `$DISTDIR`). `deb-cleanup` is what clears the residue an in-place build leaves; `distclean` does not.
  - CAUTION: those deletions are `${DISTDIR}/<package>_*`, and under cowbuilder `$DISTDIR` IS `make_cowbuilder_dist_folder` -- the .deb cache. Since `deb-pkg` runs `deb-cleanup` first, every `deb-pkg` wipes that package's cached `.deb`/`.dsc`/`.changes`/`.buildinfo` before rebuilding, so a FAILED build leaves no cached artifact for it.
  - `deb-cleanup` executes `debian/rules clean` from the working directory, i.e. it runs code from the package tree; it is not a pure file-removal step.
- `lintian`.
- `deb-chl-bumpup-*` (changelog bump).
- `git-tag-sign/-verify`.
- List all: `genmkfile help`.

## Man pages

- `genmkfile manpages` renders every `man/*.ronn` to `auto-generated-man-pages/<name>`
  with a deterministic date and `--manual`/`--organization` = the source package name.
- COMMIT `auto-generated-man-pages/` to git; run `genmkfile manpages` (and recommit) only
  when a `.ronn` changes. Do NOT gitignore them, generate at build time, or Build-Depend on
  `ronn`.
- `debian/rules` just installs them: `override_dh_installman: dh_installman
  $(CURDIR)/auto-generated-man-pages/*` (this package's own `debian/rules` is the model).

## Extending: overrides (no forking the engine)

- Drop an executable `./make-helper-overrides.bsh` or files in `./make-helper-overrides.d/` (also read from `./debian/`) to add pre/post hooks or replace any target. They are SOURCED into the engine shell -- arbitrary code from the package tree.
- Non-executable override files are skipped.
- Prefer this over editing the engine.

## Gotchas

- Run from the package root. Tarballs/build outputs go to `$DISTDIR`: `..` by default, but reassigned to `make_cowbuilder_dist_folder` whenever `make_use_cowbuilder=true` (which is then also `--buildresult`). genmkfile hand-rolls `tar` precisely BECAUSE `dpkg-source` can only target `../`.
- `deb-pkg` does NOT need `dist`/`debdist`/`debdsc` run first -- it runs them itself. The "did you run genmkfile dist" error belongs to `deb-pkg-build`, which is the build step on its own.
- The dm apt-cacher (`127.0.0.1:9977`) is only up during a dm build session; when it's down, set `make_cowbuilder_mirror=https://deb.debian.org/debian`.
- `Could not open file /var/cache/apt/archives/partial/<pkg>.deb - No such file` then
  `pbuilder-satisfydepends failed` is NOT the shared aptcache: pbuilder hardlinks `*.deb` in from
  `$APTCACHE` (`APTCACHEHARDLINK=yes`, `BINDMOUNTS=""`) and never bind-mounts it, so the chroot's
  `partial/` comes from the base cow. The real cause is a WEDGED or CONCURRENT cowbuilder build
  sharing one `cow.cow_<arch>` buildplace (this machine runs parallel sessions) -- kill the wedged
  tree by PID (it loops `Retrying to unmount dev/pts in 5s`), then serialize or give each build a
  `make_cow_suffix`. NOT a genmkfile bug; do NOT mkdir the shared aptcache.

### cowbuilder `--execute`: passing variables into a chroot script

- `cowbuilder --execute -- <script>` runs `<script>` inside the chroot with the *caller's* environment, as far as sudo lets it through.
- derivative-maker's `$SUDO_TO_ROOT` already appends `--preserve-env=$env_vars_keep_list` (`help-steps/variables`), and `cowbuilder` forwards that environment into the `--execute` script.
- `--preserve-env=<list>` only carries **exported** variables. A set-but-unexported shell variable named in the list is silently dropped.
- So to hand a chroot script its inputs, `export` them and rely on `--preserve-env` -- no need to serialize them to a file and `source` it (which would be runtime-generated code).

### cowdancer COW + the hardlink-bypass trap

- The buildplace is a fresh per-invocation `cp -al <base>` into `cow.<PID>` (hardlinks into the base), torn down afterwards unless `--save-after-login`.
- cowdancer (`libcowdancer` via `LD_PRELOAD` + `COWDANCER_ILISTFILE`) breaks a file's hardlink before a write so the base is preserved.
- But `sudo`'s `env_reset` strips both, so anything run via `sudo` inside the chroot runs **without** cowdancer -- its writes/unlinks hit the hardlinked-into-base inode directly and **corrupt the shared base**.
- Harmless for a one-shot `--execute` (the buildplace is discarded), but do NOT reuse a base across runs if a non-cowdancer step mutated files in it (e.g. `VBoxManage unregistervm --delete` deleting a VM whose disk/registry are hardlinked into the base).
- This is also why a root command run via `sudo` inside the chroot can fail with `cp: ... Cannot allocate memory`: `libcowdancer` stays `LD_PRELOAD`'ed while `COWDANCER_ILISTFILE` is gone, leaving the COW layer half-initialized -- run such commands directly (without sudo) so the inherited cowdancer env stays intact.

## Build facts (host-generator, TMPDIR, changelog)

- **genmkfile is a HOST-SIDE generator, NOT a Build-Depends.** It writes `debian/*.install`
  before the source package is packed; `debian/rules` is a plain `dh $@`, so nothing in the
  build chroot invokes it -- Build-Depends is debhelper-only. A stray `genmkfile` build-dep is a
  common reason a package can't resolve deps on a plain deb.debian.org base (genmkfile lives only
  in the kicksecure repo); any other off-mirror Build-Depends does the same. Drop it; the build
  needs no local-repo/OTHERMIRROR/D-hook.
- **libpam-tmpdir TMPDIR break -- FIXED at the root in genmkfile.** libpam-tmpdir (Kicksecure
  default) gives sudo's root session `TMPDIR=/tmp/user/0`, absent in the chroot ->
  `dpkg-deb: failed to make temporary file` -> `pbuilder-satisfydepends failed`. genmkfile now
  runs `env --unset=TMPDIR --unset=TMP --unset=TEMPDIR --unset=TEMP` immediately before every
  `cowbuilder --build` (`make-helper-one.bsh`), so NO per-build `COWBUILDER_PREFIX` or
  `--configfile` is needed to survive it, and libpam-tmpdir can stay installed. Caveat: the
  `cowbuilder --create` path is NOT covered -- a caller that creates a base under sudo still
  needs the same unset itself (see the private-ai-config skill's `ensure-cowbuilder-base`).
- **cowbuilder essentials:** `make_use_cowbuilder=true` AND `make_cowbuilder_dist_folder` are
  BOTH required, or the cowbuilder-guard refuses it as an in-place build. Base
  `/var/cache/pbuilder/base.cow_<arch>` (`cow.cow_<arch>` is scratch); `/var` resets at boot,
  recreate ~4min (`cowbuilder --create --basepath ... --distribution trixie --extrapackages
  aptitude --mirror https://deb.debian.org/debian`). `make_use_lintian=false` skips pre-existing findings.
  `genmkfile dist` tars the WORKING tree -- confirm `git status` clean first.
- **NEVER hand-edit `debian/changelog`** in genmkfile packages -- it is AUTO-GENERATED and
  gate-enforced (`check_changelog_no_manual`): a commit touching it HARD-FAILS unless it is a
  genmkfile auto-bump (exact subject `bumped changelog version` + a changelog-family-only diff)
  or carries `Changelog-manual-ok: <reason>` (prefer the bump). Workflow: (1) commit code only;
  (2) `genmkfile deb-chl-bumpup-major` (needs `export DEBFULLNAME DEBEMAIL` first -- the
  Bash-tool shell is non-interactive; it bumps upstream 0.1->0.2->...->1.0 and resets the
  revision to `-1`, the Kicksecure convention, NOT a `-N` bump); (3) commit the changelog with
  that exact subject.
