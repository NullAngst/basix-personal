# Packaging for Basix: the reference

How bpm 0.9 turns a template into a package, checked line by line against the bpm source (kkrruumm/bpm 0.9.3, the version core ships) and the templates in core. This is what Claude reads before writing a template, and what you read when one breaks.

When this file and the bpm source disagree, the source wins. bpm has no docs of its own, so read `lib/common.sh`, `lib/build.sh` and `lib/style/*.sh` when in doubt.

## Where packages go in this repo

- `<Group>/<pkg>/template` or `<Group>/<Sub>/<pkg>/template`. bpm looks through at most two group folders. A package folder nested deeper is invisible.
- The folder name is the package name. `pkg_name` has to match it or bpm refuses the template.
- Names are unique across every repo in `repos.conf`. The first repo listed wins, and inside a repo the shallowest path wins. A duplicate shadows the other one silently.
- Putting a package here with the same name as a core package replaces core's for anyone with this repo above core. Do that on purpose only.
- `Packaging/` holds no packages. NEVER PUT A FILE NAMED `template` IN `Packaging/`. bpm would see `Packaging/<x>/template` as a package, and kde-bump scans the whole repo for files named `template` to fill checksums.

## How bpm reads a template

The template is sourced by `/bin/sh` with `set -eu`. On Basix that shell is busybox ash, so:

- POSIX sh only. No `[[ ]]`, arrays, `function`, `${var//a/b}`, `${var:0:3}`, `<<<`, `source`, `echo -e`.
- `set -u`: reading an unset variable is fatal. Write `${foo:-}`.
- `set -e`: the build dies on any failing command, including the last command of a phase function. `use doc && bdoc README` as the last line fails the build when doc is off. End with `:` or write an `if`.
- The template is sourced many times per run: dependency walks, use flag checks, the fetch job, the build stage. The top level only assigns variables and defines functions. Commands inside `$( )` at the top level are fine when they're pure, like the `use_if` helpers.
- bpm presets `pkg_name` (to the folder name), `revision=1` and `build_style=gnu-configure` before sourcing, then fails if `version` is empty or `pkg_name` doesn't match.

## Variables bpm understands

The full list from `TMPL_VARS` in `lib/common.sh`. Anything else is ignored by bpm. Prefix your own helper variables with `_` (`_commit`, `_tag`), the way core does.

| Variable | What it does |
|---|---|
| `pkg_name` | Package name, must equal the folder name |
| `version` | Upstream version. Archive is `<pkg>@<version>-<revision>.tar.<comp>` |
| `revision` | Default 1. Bump for template-only changes, reset to 1 on a version bump |
| `build_style` | Default `gnu-configure`. See the styles table |
| `short_desc`, `home_page`, `license` | Shown by `bpm query`. License is SPDX |
| `maintainer`, `conflicts`, `provides` | Accepted and cleared between templates, but nothing in bpm 0.9 acts on them |
| `dist_files` | Sources, see below |
| `checksum` | BLAKE3 per dist_files entry, see below |
| `wrk_src` | Source dir name inside the build dir. Default `$pkg_name-$version` |
| `create_wrk_src` | `yes` extracts into a new `$wrk_src` dir, for tarballs with no top dir |
| `skip_extract` | Saved distfile names to copy instead of extract |
| `build_wrk_src` | Subdir of wrk_src to run the phases from, created if missing |
| `patch_args` | Args to `patch` for `patches/`, default `-p1` |
| `depends` | Runtime deps, recorded in the package, installed first |
| `make_depends` | Build deps, the only thing besides base inside the build root |
| `use_depends` | `pkg:flag` pairs that must be enabled on those packages, bpm dies if not |
| `use_flags` | Declared flags and their defaults |
| `allow_network` | `yes` lets the build reach the network |
| `no_strip` | `yes` skips stripping |
| `keep_libtool` | `yes` keeps `.la` files |
| `register_shell` | Paths added to `/etc/shells` |
| `system_accounts`, `system_groups` | Accounts and groups made at install, with `<name>_uid`, `_gid`, `_descr`, `_homedir`, `_shell`, `_groups` |
| `config_files` | Extra paths outside `/etc` treated as config |
| `configure_script`, `configure_args` | configure step |
| `make_cmd` | Default `make` |
| `make_build_args`, `make_build_target` | build step |
| `make_check_args`, `make_check_target` | check step, target default `check` |
| `make_install_args`, `make_install_target` | install step, target default `install` |
| `meson_args` | meson only, before configure_args |
| `cmake_build_type` | Default `Release` |
| `qmake_cmd` | Default `qmake` |
| `zig_build_args` | Replaces `$ZIGFLAGS` when set |
| `cargo_args`, `cargo_install_path` | cargo only |
| `go_package` (or `go_import_path`), `go_build_tags`, `go_ldflags`, `go_mod_mode` | go only |
| `gem_spec`, `stackage` | gemspec and haskell-stack only |

Not bpm variables, even though other docs use them: `use_default`, `host_make_depends`, `hostmakedepends`, `makedepends`, `distfiles`, `homepage`, `bpm_sccache`. FrankenBasix's `TEMPLATES.md` documents several of these for its fork. Against bpm 0.9 they do nothing.

## Sources and checksums

`dist_files` entries, space separated:

- `https://host/foo-1.0.tar.xz`: saved under the last path segment, query string dropped.
- `https://host/archive/v${version}.tar.gz>foo-${version}.tar.gz`: saved under the name after `>`. Do this for GitHub, Codeberg and sourcehut tag archives, the way core does. A bare `v1.0.tar.gz` says nothing about what's in it, and two sources of one package with the same saved name overwrite each other in `$BPM_CACHE/sources/<pkg>/`.
- `git+https://host/foo.git#<tag or commit>`: cloned with submodules, checked out detached. Checksum is `SKIP`. Pin to a commit hash when integrity matters, since a tag can move.
- `file:///abs/path.tar.gz`: copied from the local disk.

Extraction: `.tar`, `.tar.*`, `.tgz`, `.txz`, `.tbz2` through tar, `.zip` through `unzip` (core has no unzip package, so that leans on busybox's applet), anything else copied into the build dir. Extra archives can be unpacked by hand in a phase with `bextract <saved name> [subdir]`.

Set `wrk_src` whenever the top dir of the archive isn't `$pkg_name-$version`. bpm 0.9.3 has code meant to detect the dir on its own, but it runs before extraction, so it never fires. With a wrong `wrk_src`, bpm quietly `cd`s into the parent build dir instead, then patching or configure fails with an error that doesn't mention wrk_src. Typical cases: Codeberg archives (`foo/`, no version), tags with a `v` (`foo-v1.0/` from sourcehut), upstream names that differ from the package name.

`checksum`: one BLAKE3 hash per dist_files entry, in the same order, `SKIP` in the slot of a `git+` entry (bpm still counts that slot). bpm compares the first 64 hex characters, so stray characters after the hash pass unnoticed. Rules:

- Keep `checksum="..."` on one line. `bpm checksum -w` replaces only the line starting with `checksum=`, and kde-bump fills only the exact text `checksum=""`. A multi-line checksum like core's readline breaks both.
- An empty checksum makes bpm refuse to build that package. Fill it with `doas bpm checksum -w <pkg>` on Basix or run the kde-bump workflow, which downloads the sources and writes the sums.
- `SKIP` on a tarball turns verification off. Don't.

Sources are fetched outside the sandbox before any build starts, up to `BPM_FETCH_JOBS` packages in parallel, into `$BPM_CACHE/sources/<pkg>/`.

## What a build does, in order

1. Fetch (`do_fetch` if the template defines it) and verify checksums. Outside the sandbox.
2. Enter the sandbox: new mount, pid, ipc, uts namespaces, plus a network namespace unless `allow_network=yes`. Unprivileged users get a user namespace and run as root inside it.
3. With `BPM_BUILDROOT` set (stock bpm.conf sets it): assemble a throwaway root from `BPM_BASEPKGS` plus the runtime closure of `make_depends`, chroot into it with a scrubbed environment.
4. Extract: `do_extract`, else the style's, else bpm's own.
5. `cd $build_dir/$wrk_src` (or `$build_dir` without a word if that's missing), then patch: `do_patch`, else apply `patches/*.patch` and `*.diff` sorted together, with `patch_args`.
6. `cd` into `build_wrk_src` if set.
7. configure, build, check (only with `BPM_CHECK=1`), install. Each runs `pre_<phase>`, then `do_<phase>` or the style's `style_<phase>`, then `post_<phase>`.
8. Delete `.la` files (unless `keep_libtool=yes`), strip executables (unless `no_strip=yes` or `BPM_STRIP=0`).
9. Fold dirs that are symlinks on the live system (like `/usr/sbin` to `/usr/bin`, `/lib` to `/usr/lib`) into their targets.
10. Write the db entry (template, version, use flags, depends, accounts, shells, BLAKE3 of every file, manifest) and the archive into `$BPM_CACHE/bin/`.

The only hooks are the ones in step 7 plus `do_fetch`, `do_extract`, `do_patch`. `pre_patch`, `post_extract`, `pre_fetch` and friends are never called.

Inside the phases:

| Name | Value |
|---|---|
| `$DESTDIR` | Staging root. Everything here becomes the package |
| `$FILESDIR` | `<pkg>/files` |
| `$PATCHESDIR` | `<pkg>/patches` |
| `$SOURCEDIR` | Downloaded dist_files |
| `$build_dir` | Dir holding `$wrk_src` |
| `$make_cmd`, `$make_jobs` | `make`, `-j$BPM_JOBS`. `MAKEFLAGS` is set to the same |
| `$CC`, `$CXX` | `cc`, `c++` unless set |
| `$CFLAGS`, `$CXXFLAGS`, `$LDFLAGS`, `$RUSTFLAGS`, `$GOFLAGS` | Hardened defaults from bpm.conf (`-O2`, `_FORTIFY_SOURCE=3`, relro, now, `-fstack-protector-strong`, PIE for Go) |
| `HOME` | `/root` inside the build root |
| `LC_ALL` | `C` |

## Build styles

What each style runs, from `lib/style/`. "needs" is what make_depends has to carry beyond base.

| Style | configure | build | install | needs |
|---|---|---|---|---|
| `gnu-configure` | `./configure --prefix=/usr --bindir=/usr/bin --sbindir=/usr/bin --libdir=/usr/lib --libexecdir=/usr/libexec --sysconfdir=/etc --localstatedir=/var` (+ `--runstatedir=/run` when supported) `$configure_args` | `$make_cmd $make_jobs` | `$make_cmd DESTDIR=... install` | nothing for a release tarball. `autoconf automake libtool` if you have to run `autoreconf` |
| `configure` | `./configure --prefix=/usr $configure_args` | make | make install | for hand-written configure scripts |
| `gnu-makefile` | none | `make PREFIX=/usr` | `make DESTDIR=... PREFIX=/usr MANPREFIX=/usr/share/man install` | nothing |
| `cmake` | `cmake -S . -B build` with `/usr`, `lib`, `/etc`, `/var`, Release, the flags | `cmake --build build --parallel` | `cmake --install build` | `cmake`. Add `samurai` when configure_args has `-G Ninja` |
| `meson` | `meson setup build --prefix=/usr --libdir=lib --buildtype=plain --wrap-mode=nodownload ...` | `ninja -C build` | `ninja -C build install` | `meson` (pulls python and samurai) |
| `cargo` | none | `cargo build --release --locked` | `cargo install --path . --root $DESTDIR/usr --locked` | `rust`, `allow_network=yes`, a `Cargo.lock` in the tarball |
| `go` | none | `go build -o .gobin/ $go_package` with GOPATH in the build dir | every binary in `.gobin` to `/usr/bin` | `go`, `allow_network=yes` unless `vendor/` ships |
| `python3-pep517` | none | `python3 -m build --wheel --no-isolation` | `python3 -m installer --destdir=... --prefix=/usr` | `python python-build python-installer` plus the backend (`python-setuptools python-wheel`, `python-flit-core`, ...) |
| `python3-module` | none | `python3 setup.py build` | `setup.py install --root=...` | `python python-setuptools` |
| `perl-module` / `perl-ModuleBuild` | `perl Makefile.PL` / `perl Build.PL` | make / `./Build` | vendor dirs, `.packlist` removed | `perl` |
| `qmake` | `$qmake_cmd PREFIX=/usr ...` | make | `make INSTALL_ROOT=... install` | Qt |
| `scons`, `waf`, `waf3` | waf: `./waf configure` | `scons` / `./waf build` | with DESTDIR | `scons` / `python` |
| `zig-build` | none | none | `zig build install --prefix $DESTDIR/usr $ZIGFLAGS` | `zig` |
| `ruby-module`, `gemspec`, `R-cran`, `haskell-stack` | see `lib/style/` | | | ruby, R, stack. Rare on Basix |
| `fetch` | dist_files copied into the build dir, never extracted | none | your `do_install`, required | |
| `meta` | none | none | creates `/usr/share/doc/$pkg_name` | `depends` is the point. No dist_files or checksum needed |
| `custom` | yours | yours | yours | whatever you call |

Style notes that bite:

- meson: `--wrap-mode=nodownload` means subprojects aren't fetched. A project that relies on wraps for deps has to get those deps as packages, or ship them in the tarball.
- meson: `--buildtype=plain` so bpm's CFLAGS are the only optimization flags. Don't add `-Dbuildtype=release`.
- cmake: default generator is Unix Makefiles. KDE templates add `-G Ninja` and `samurai`.
- cargo: `--locked` fails without `Cargo.lock`. Core sets `export CARGO_HOME="$PWD/.cargo"` in `pre_build` to keep the registry in the build tree. When you only want some binaries, override `do_install` with `bbin target/release/<bin>` the way core's greetd does.
- go: core sets `go_ldflags="-s -w ..."` and `no_strip=yes`.
- custom: no style at all, so a phase you don't define does nothing. Without `do_install` the package is empty.

## Install helpers

All paths are relative to `$DESTDIR`. Sources are relative to the current dir (wrk_src).

| Helper | Does |
|---|---|
| `binstall <file> <mode> <dir> [newname]` | Copy one file, any mode, any dir |
| `bbin <file> [newname]` | 0755 to `/usr/bin` |
| `blib <file> [newname]` | 0644 to `/usr/lib` |
| `blibexec <file> [newname]` | 0755 to `/usr/libexec/$pkg_name` |
| `binclude <file> [newname]` | 0644 to `/usr/include` |
| `bpkgconfig <file> [newname]` | 0644 to `/usr/lib/pkgconfig` |
| `bconf <file> [newname]` | 0644 to `/etc` |
| `blicense <file> [newname]` | 0644 to `/usr/share/licenses/$pkg_name` |
| `bdoc <file> [subdir]` | 0644 to `/usr/share/doc/$pkg_name[/subdir]`. Second arg is a dir, not a name |
| `bdata <file> [subdir]` | 0644 to `/usr/share/$pkg_name[/subdir]` |
| `bman <file.N> [newname]` | 0644 to `/usr/share/man/manN`, section from the extension |
| `bcompletion <file> <bash\|zsh\|fish> [name]` | Into the right completion dir, named for the shell |
| `bmkdir [-m mode] <dir>...` | Make dirs, default 0755 |
| `bcopy <src>... <dir>` | Recursive copy, globs ok |
| `bmove <src> <dest>` | Move inside DESTDIR |
| `bln <target> <linkname>` | Symlink, target written exactly as given |
| `brm <path>...` | Delete inside DESTDIR |
| `bsed <expr> <file>...` | Portable in-place sed |
| `bextract <distfile> [subdir]` | Unpack an extra distfile into the build dir |
| `die`, `msg`, `sub`, `warn` | Error out, or print status lines |

## Use flags

- `use_flags="man -doc wayland"`: `man` and `wayland` default on, `doc` off.
- Resolution, later wins: template `use_flags`, then `BPM_USE` from bpm.conf, then matching lines in `/etc/bpm/package.use` (shell glob on the package name).
- The stock bpm.conf has `BPM_USE="zlib ssl man doc -x11 -wayland -pulseaudio -bluetooth -nls"`. That beats the template default. A template with `wayland` on still builds without it until the user turns it on. The KDE README has users remove `-wayland` and `-pulseaudio` from BPM_USE for Plasma.
- `use` reads global flags even when the template doesn't declare them, but only declared flags are recorded in the package. Declare every flag you test, or flipping it won't trigger a rebuild.
- A changed flag set rebuilds the package on the next `bpm install` or `bpm update`.

Helpers, all usable at the top level inside `$( )` and inside functions:

| Call | On | Off |
|---|---|---|
| `use x` | exit 0 | exit 1 |
| `usev x [val]` | `val` (default `x`) | nothing |
| `use_if x a [b]` | `a` | `b` or nothing |
| `use_enable x [name]` | `--enable-name` | `--disable-name` |
| `use_with x [name]` | `--with-name` | `--without-name` |
| `use_bool x [name]` | `-Dname=true` | `-Dname=false` |
| `use_cmake x [name]` | `-Dname=ON` | `-Dname=OFF` |

For meson features (`enabled`/`disabled`), write `$(use_if x -Dfoo=enabled -Dfoo=disabled)`, as core does. `use_bool` gives true/false, which is for boolean options only. Pass a multi-word value with quotes: `$(use_if man "docbook-xsl libxslt")`.

Flags core uses a lot, so reuse these names instead of inventing new ones: `man`, `doc`, `nls`, `static`, `x11`, `wayland`, `pulseaudio`, `bluetooth`, `services`, `completions`, `desktop`, `ssl`, `zlib`.

## Dependencies

- `depends`: runtime. Recorded in the package. bpm installs them before the package and walks them recursively.
- `make_depends`: build time. With a build root, the root is `BPM_BASEPKGS` + `make_depends` + their runtime deps, nothing else. A library foo links against goes in both lists. A missing one is a build failure, which is the point.
- Without a build root (`BPM_BUILDROOT=` empty), make_depends must already be installed on the host or bpm stops and lists them.
- Base, from the stock bpm.conf: `baselayout musl busybox make binutils gcc linux-headers fortify-headers pigz xz bzip2 certs pkgconf b3sum zstd`. No bash, no perl, no python, no cmake.
- Runtime deps of make_depends come along. `meson` brings `python` and `samurai`, so a meson template doesn't list samurai.
- Continuation lines start with one space, like core:

```sh
make_depends="fcft fontconfig freetype meson pkgconf
 scdoc wayland wayland-protocols"
```

## The system you're packaging for

Basix is not glibc and not systemd. Check each of these against the software before writing anything:

- musl libc. glibc-only code (`execinfo.h`, `res_nquery`, `*64` LFS aliases, `strerror_r` GNU form, `error.h`, `fts.h`, `obstack`) needs patches. Prebuilt glibc binaries, AppImages and most vendor tarballs won't run at all. For those, use Flatpak instead of a template.
- LibreSSL, package `libressl`. There is no `openssl` package. Projects that need OpenSSL 3-only APIs need patches or a different TLS backend.
- busybox userland, no coreutils package. busybox covers the common options, but build scripts that lean on rarer GNU coreutils, sed or grep options can fail. bash isn't in the build root either: a `#!/bin/bash` build script needs `bash` in make_depends.
- `libudev-zero` (core) or `eudev` (this repo), no systemd, no logind except `elogind` from this repo. s6 and s6-rc for services.
- `gettext-tiny` in core, full `gettext` in this repo. Translations are behind the `nls` flag.
- `samurai` is the ninja. There is no package named `ninja`.
- Missing at the time of writing (check again, these change): nodejs, gtk4, NetworkManager, BlueZ, CUPS, UPower, UDisks2, sdl2/sdl3, an `openssl` package.

Where to find musl fixes, in order: Alpine aports, Chimera Linux cports, Void Linux srcpkgs (musl builds), KISS community. Copy the patch, don't reinvent it.

## Patches

- In `<pkg>/patches/`, named `NN-what.patch`, applied in sorted order with `patch -p1` (or `patch_args`). No whitespace in names.
- First two lines, before the diff (patch skips leading text): where it came from, and why.

```
Origin: Alpine aports community/foo/fix-musl.patch
Why: musl has no execinfo.h
```

- `do_patch` replaces patching entirely. Call `apply_patches` inside it to keep `patches/` working, like core's readline.

## Services, accounts, hooks

s6-rc service, the way core's greetd and this repo's wpa_supplicant do it:

```
<pkg>/files/sources/foo/type               longrun
<pkg>/files/sources/foo/run                #!/bin/execlineb -P, then fdmove -c 2 1, then the daemon in the foreground
<pkg>/files/sources/foo/producer-for       foo-log
<pkg>/files/sources/foo/dependencies.d/base   empty file
<pkg>/files/sources/foo-log/type           longrun
<pkg>/files/sources/foo-log/run            #!/bin/execlineb -P, then s6-log -d3 -- n3 s1000000 T /var/log/foo
<pkg>/files/sources/foo-log/consumer-for   foo
<pkg>/files/sources/foo-log/notification-fd   3
<pkg>/files/sources/foo-log/dependencies.d/base   empty file
```

Installed behind `use_flags="services"` with the `bcopy` and `chmod` lines from `template.skel`. Definitions live under `/usr/share` so bpm owns them. baseinit's hook syncs the s6-rc repository after every install and removal. Which services run at boot is repository state under `/var/lib/s6-rc` that the user controls, so the template never enables anything.

Accounts: `system_accounts` and `system_groups` in the template, created at install time before the files land. See greetd in core.

Global caches: bpm has no per-package install scripts, only global hooks in `/etc/bpm/hooks/{pre,post}-{install,remove}/`. Existing ones: ldconfig (bpm), glib schemas and gio modules (glib), MIME database (shared-mime-info, this repo), s6-rc (baseinit). There's no icon cache or desktop database hook, so packages don't ship `icon-theme.cache`. If a package installs `gschemas.compiled` or `giomodule.cache`, `brm` them in post_install, the glib hook rebuilds them.

## Versions

Every template pins its version and checksum. bpm has no "latest": it rebuilds when `version-revision` or the flag set changes, so a moving source would never update. That's a tradeoff built into bpm, and no template setting changes it. What keeps things current is automation that edits the pins:

- KDE: kde-bump moves `DE/KDE/manifest` weekly and regenerates those templates.
- Everything hand-written: a `watch` file next to the template opts it into `Packaging/pkg-bump`, which the pkg-bump workflow runs weekly. It finds upstream's newest stable release, sets `version=`, resets `revision=1`, downloads every source, writes the BLAKE3 sums and opens one pull request. A package whose download fails keeps its old version.
- No watch file: bump by hand. Change `version`, set `revision=1`, empty the checksum, push, let the kde-bump workflow fill it.

The watch file, one `key=value` per line, `#` comments:

| Key | Meaning |
|---|---|
| `type=git` | Newest tag of a git repo, any host. Needs `url=` (the repo, `git ls-remote` reads it) |
| `type=pypi` | Newest release on PyPI. Needs `name=`. Downloads are checked against PyPI's sha256 |
| `type=crates` | Newest release on crates.io. Needs `name=` |
| `type=url` | Newest match on a download page or directory listing. Needs `url=` and `match=` |
| `tag=` | git only. Regex for the whole tag, group 1 is the version. Default `^v?([0-9]+(\.[0-9]+)+)$`, which skips rc, beta and anything else with letters |
| `match=` | url only. Regex, group 1 is the version |
| `filter=` | pypi and crates. Versions to accept, default `^[0-9]+(\.[0-9]+)+$` |
| `ignore=` | Versions to skip, for a dev series like GNOME's odd minors |
| `series=` | Only versions starting with this, `6.` to stay on 6.x |

Regexes are POSIX extended and can't contain `#`. Example for a GitHub project tagged `v1.2.3`:

```
type=git
url=https://github.com/owner/foo
```

pkg-bump only changes `version=`, `revision=` and `checksum=`. It can't handle a template whose sources follow different version schemes, or a multi-line checksum. Those stay manual. It also can't know that a new release needs a new dependency, option or patch, so every bump pull request is a "build it on Basix before merging" job.

A `git+` source on a branch name would drift without bpm noticing. Pin it to a tag or commit.

## Testing on Basix

1. Pull the repo: `doas bpm pull`
2. Check it loads and the flags resolve: `bpm query foo`
3. Fill the checksum if empty: `doas bpm checksum -w foo`, then copy the template from `/var/db/bpm/repos/nullangst/...` back into your clone
4. Build it: `doas bpm build foo`, or without doas if your user can write `/var/cache/bpm`
5. If it fails, read the whole log: `bpm log foo`. `doas bpm -v build foo` shows it live.
6. Install it: `doas bpm install foo`
7. See what it installed: `bpm files foo`
8. Rebuild without bumping anything: `doas env BPM_FORCE=1 bpm build foo`

## Things Claude can't check from a sandbox

No Basix machine or rootfs is available in Claude's sandbox. Template syntax, variable names, dependency names, checksum shape and repo layout can be checked there (`lint-template` does that). Whether it configures, compiles on musl and runs can only be checked by `bpm build` on Basix. Every template is unbuilt until you build it.
