# Generic Makefile #

Makes packaging simpler. No more need to manually maintain
'make install' targets or distribution specific install files such as
debian/pkg-name.install.

Files in etc/... in root source folder will be installed to /etc/..., files in
usr/... will be installed to /usr/... and so forth. This should make renaming,
moving files around, packaging, etc. very simple. Packaging of most packages
can look very similar.

Provides common make targets such as 'make install', 'make dist',
'make installsim', 'make installcheck', 'make uninstall',
'make uninstallcheck', 'make distclean'.

Very extensible through
file ./make-helper-overrides.bsh or
folder ./make-helper-overrides.d.
By using overrides, any make target can be easily extended using pre or post
hooks or replaced.
Override files which are executable will be used.
Override files which are not executable will be skipped.

Contains a minimal Makefile while the heavy lifting is done by a bash script
make-helper.bsh.

Building for multiple platforms possible, example:
export make_cross_build_platform_list="i386 amd64"

Can call with lintian (static analysis tool for Debian packages).
By default it will be using lintian if installed while failing open
(non-zero exit code).
lintian can be disabled.
export make_use_lintian=false
Or can be configured to fail closed (non-zero exit code).
export make_use_lintian=true

Can build packages without chroot using debuild (default) or inside chroot
using cowbuilder. To enable cowbuilder, use:
export make_use_cowbuilder=true

Supports signing packages using debsign.
(sign a Debian .changes and .dsc file pair using GPG)
export make_use_debsign=true
## How to install `genmkfile` using apt-get ##

1\. Download [Whonix's Signing Key]().

```
wget https://www.whonix.org/patrick.asc
```

Users can [check Whonix Signing Key](https://www.whonix.org/wiki/Whonix_Signing_Key) for better security.

2\. Add Whonix's signing key.

```
sudo apt-key --keyring /etc/apt/trusted.gpg.d/whonix.gpg add ~/patrick.asc
```

3\. Add Whonix's APT repository.

```
echo "deb https://deb.whonix.org buster main contrib non-free" | sudo tee /etc/apt/sources.list.d/whonix.list
```

4\. Update your package lists.

```
sudo apt-get update
```

5\. Install `genmkfile`.

```
sudo apt-get install genmkfile
```

## How to Build deb Package ##

Replace `apparmor-profile-torbrowser` with the actual name of this package with `genmkfile` and see [instructions](https://www.whonix.org/wiki/Dev/Build_Documentation/apparmor-profile-torbrowser).

## Contact ##

* [Free Forum Support](https://forums.whonix.org)
* [Professional Support](https://www.whonix.org/wiki/Professional_Support)

## Donate ##

`genmkfile` requires [donations](https://www.whonix.org/wiki/Donate) to stay alive!
