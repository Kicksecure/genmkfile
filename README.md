# Generic Makefile #

Makes packaging scripts simpler. No more need to manually maintain
'make install' targets or distribution specific install files such as
debian/pkg-name.install.

Files in etc/... in root source folder will be installed to /etc/..., files in
usr/... will be installed to /usr/... and so forth. This should make renaming,
moving files around, packaging, etc. very simple. Packaging of most packages
can look very similar.

Provides common make targets such as 'make install'. Very extensible through
file ./make-helper-overrides.bsh or
folder ./make-helper-overrides.d. By using overrides, any make target can be
easily extended using pre or post hooks or replaced.

Contains a minimal Makefile while the heavy lifting is done by a bash script
make-helper.bsh.
## How to install `genmkfile` using apt-get ##

1\. Add [Whonix's Signing Key](https://www.whonix.org/wiki/Whonix_Signing_Key).

```
sudo apt-key --keyring /etc/apt/trusted.gpg.d/whonix.gpg adv --keyserver hkp://ipv4.pool.sks-keyservers.net:80 --recv-keys 916B8D99C38EAF5E8ADC7A2A8D66066A2EEACCDA
```

3\. Add Whonix's APT repository.

```
echo "deb http://deb.whonix.org buster main" | sudo tee /etc/apt/sources.list.d/whonix.list
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

## Payments ##

`genmkfile` requires [payments](https://www.whonix.org/wiki/Payments) to stay alive!
