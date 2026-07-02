---
name: genmkfile
description: "Use when building Kicksecure/derivative-maker Debian packages with genmkfile (targets like 'genmkfile deb-pkg', cowbuilder chroot builds, the output/dist folder as a .deb cache, dist/debdist/debdsc tarballs, deb-install, lintian, debsign). Covers the build sequence, env-var knobs, cowbuilder base creation, and override hooks."
license: MIT
---

# genmkfile: building derivative-maker / Kicksecure Debian packages

`genmkfile` is a generic Makefile generator used by Kicksecure / Whonix /
derivative-maker packages. It replaces hand-written `debian/*.install` and
`make install` targets: files laid out as `usr/...`, `etc/...` in the package
root install to `/usr/...`, `/etc/...`. The engine is a big bash script
(`/usr/share/genmkfile/make-helper-one.bsh`); a thin `/usr/bin/genmkfile` wrapper
locates `GENMKFILE_PATH` (`./packages/kicksecure/genmkfile/usr/share/genmkfile`
in a dm checkout, else `/usr/share/genmkfile`). Run it from a package's root
(the dir containing `debian/`).

## Build sequence (cowbuilder)

`deb-pkg` with cowbuilder needs the source tarballs built FIRST, in order:

```
genmkfile dist       # -> upstream tarball  (excludes .git and ./debian)
genmkfile debdist     # -> debian tarball
genmkfile debdsc      # -> .dsc
genmkfile deb-pkg     # -> builds the .deb(s)
```

`genmkfile distclean` removes generated tarballs/build cruft. A plain
`genmkfile deb-pkg` with no cowbuilder uses `debuild` in-place (needs build-deps
installed; `sudo genmkfile deb-all-dep` installs them).

## cowbuilder mode (chroot build)

**In a derivative-maker checkout, do NOT hand-roll the cowbuilder invocation.**
Use dm's own build steps, which set up the base, the approx mirror, the pbuilder
config, and the TMPDIR removal canonically (and call the dm-submodule genmkfile,
not the installed one):

```
make_cross_build_platform_list=amd64 \
  ./build-steps.d/*_cowbuilder-setup       --allow-untagged true --allow-uncommitted true --flavor source --target root
make_cross_build_platform_list=amd64 \
  ./build-steps.d/*_create-debian-packages --allow-untagged true --allow-uncommitted true --flavor source --target root
```

dm handles the libpam-tmpdir/pbuilder breakage via `help-steps/variables`
(`chroot_env_vars_var_remove_list="TEMP TEMPDIR TMP TMPDIR"`), the mirror via the
approx apt-cacher on `127.0.0.1:9977` (`APPROX_PROXY_ENABLE=yes`), and the pbuilder
config auto-generated to `~/derivative-binary/pbuilder.conf`. dm's `2100` step only
iterates dm's own package set + does `reprepro-add`. The raw knobs below are for
understanding / building a standalone package OUTSIDE a dm checkout.

### Building standalone packages WITH dm's cowbuilder env (verified recipe)

For packages that aren't in dm's package set (so dm's `2100` won't iterate them)
but should still build with dm's machinery, reuse dm's base + config instead of
hand-rolling. Run dm's setup ONCE, then build each package via genmkfile pointed
at the dm pbuilder config:

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
genmkfile dist && genmkfile debdist && genmkfile debdsc && genmkfile deb-pkg
```

Gotchas learned: dm's `1300` fails standalone WITHOUT `1200` first (debootstrap).
The TMPDIR break is fixed by dm's base (in-chroot `mkdir $TMPDIR`), NOT by passing
the config alone -- a config-only build still fails `dpkg-deb: failed to make
temporary file`. If a package's source tree has an empty dir, `genmkfile dist`
errors "Empty directory found!"; fix it with a `./make-helper-overrides.bsh`
(executable) defining `make_dist_hook_pre` to drop a placeholder -- don't move
the package off genmkfile. A small wrapper that loops over the package dirs keeps
a multi-package build DRY.

### Raw genmkfile cowbuilder knobs (standalone, outside dm)

Enable with env vars (the README knobs):

```
make_use_cowbuilder=true \
make_cowbuilder_dist_folder=<dist-folder> \
dist_build_pbuilder_config_file=~/derivative-binary/pbuilder.conf \
make_cross_build_platform_list=amd64 \
genmkfile deb-pkg
```

- `make_use_cowbuilder=true` -- build inside a cowbuilder chroot.
- `make_cowbuilder_dist_folder=<dir>` -- REQUIRED with cowbuilder; the **output /
  dist folder** where the built `.deb` / `.changes` / `.dsc` land (`--buildresult`).
  This folder is the natural **package cache**. Must already exist.
- `make_cross_build_platform_list='amd64 arm64'` -- one build per arch.
- `make_cowbuilder_distribution` -- defaults to `$dist_build_apt_stable_release`
  or `lsb_release -sc` (e.g. `trixie`).
- `make_cowbuilder_mirror` -- apt mirror for the chroot; derivative-maker sets it
  to its apt-cacher (`http://127.0.0.1:9977/debian`); falls back to
  `https://deb.debian.org/debian`. (In dm, `buildconfig.d` exports it.)
- `dist_build_pbuilder_config_file` -- `--configfile` for cowbuilder/pbuilder.
- `make_use_lintian=true|false`, `make_use_debsign=true` + `make_debsign_opts`
  (debsign with `sq verify` against `$DEBEMAIL`).

genmkfile passes to cowbuilder: `--build <dsc> --basepath /var/cache/pbuilder/base.cow_<arch> --buildplace /var/cache/pbuilder/cow.cow_<arch> --buildresult <dist_folder> --debbuildopts=-sa`. **It does NOT create the base** -- create it once first.

### Create the cowbuilder base (once per arch)

cowbuilder `--build` needs `--basepath` to already exist. Create it:

```
sudo cowbuilder --create \
  --basepath /var/cache/pbuilder/base.cow_amd64 \
  --distribution trixie \
  --architecture amd64 \
  --mirror https://deb.debian.org/debian \
  [--configfile ~/derivative-binary/pbuilder.conf]
```

Update it later with `--update --basepath ...`. Note: build-deps (e.g.
`debhelper`) must be reachable from `--mirror`; the package's *runtime* deps are
NOT needed at build time. So `Architecture: all` packages whose only build-dep is
debhelper build against a plain `deb.debian.org` base even if runtime deps
(signal-cli, etc.) live elsewhere. Caveat: `libpam-tmpdir` breaks cowbuilder
(sets TMPDIR) -- keep it out of the build host.

## Other useful targets

`install` / `uninstall` (to a live system or `DESTDIR`), `installsim` /
`uninstallcheck` (dry runs), `deb-install` (build-deps via dpkg), `deb-icup`
(install just-built current package), `deb-remove` / `deb-purge`, `lintian`,
`deb-chl-bumpup-*` (changelog bump), `git-tag-sign/-verify`. List all:
`genmkfile help`.

## Extending: overrides (no forking the engine)

Drop an executable `./make-helper-overrides.bsh` or files in
`./make-helper-overrides.d/` to add pre/post hooks or replace any target.
Non-executable override files are skipped. Prefer this over editing the engine.

## Gotchas

- Run from the package root; tarballs/build outputs are written to `../`
  (`dpkg-source` can't target another dir), but cowbuilder results go to
  `--buildresult` (the dist folder).
- `deb-pkg` (cowbuilder) hard-fails if `dist`/`debdist`/`debdsc` weren't run --
  the error tells you which tarball is missing.
- The dm apt-cacher (`127.0.0.1:9977`) is only up during a dm build session;
  when it's down, set `make_cowbuilder_mirror=https://deb.debian.org/debian`.

### cowbuilder `--execute`: passing variables into a chroot script

`cowbuilder --execute -- <script>` runs `<script>` inside the chroot with the
*caller's* environment, as far as sudo lets it through. derivative-maker's
`$SUDO_TO_ROOT` already appends `--preserve-env=$env_vars_keep_list`
(`help-steps/variables`), and `cowbuilder` forwards that environment into the
`--execute` script (verified: a `--preserve-env`'d, exported variable is visible
inside the chroot).

- `--preserve-env=<list>` only carries **exported** variables. A set-but-unexported
  shell variable named in the list is silently dropped. So to hand a chroot script
  its inputs, `export` them and rely on `--preserve-env` -- there is no need to
  serialize them to a file and `source` it (which would be runtime-generated code).
  Several `env_vars_keep_list` entries are set but not exported, which is exactly
  why the old vbox chroot scripts ferried them through a generated `declare -p`
  env-file; exporting them is the fix.

### cowdancer COW + the hardlink-bypass trap

The buildplace is a fresh per-invocation `cp -al <base>` into `cow.<PID>` (hardlinks
into the base), torn down afterwards unless `--save-after-login`. cowdancer
(`libcowdancer` via `LD_PRELOAD` + `COWDANCER_ILISTFILE`) breaks a file's hardlink
before a write so the base is preserved. But `sudo`'s `env_reset` strips both, so
anything run via `sudo` inside the chroot runs **without** cowdancer -- its
writes/unlinks hit the hardlinked-into-base inode directly and **corrupt the shared
base**. Harmless for a one-shot `--execute` (the buildplace is discarded), but do
NOT reuse a base across runs if a non-cowdancer step mutated files in it (e.g.
`VBoxManage unregistervm --delete` deleting a VM whose disk/registry are hardlinked
into the base). This is also why a root command run via `sudo` inside the chroot can
fail with `cp: ... Cannot allocate memory`: `libcowdancer` stays `LD_PRELOAD`'ed
while `COWDANCER_ILISTFILE` is gone, leaving the COW layer half-initialized -- run
such commands directly (without sudo) so the inherited cowdancer env stays intact.
