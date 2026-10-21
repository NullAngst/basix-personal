# KDE Plasma for Basix

The KDE part of basix-personal: Plasma 6 on Wayland (Qt 6.12, KDE Frameworks 6.31, Plasma 6.8, a few Gear apps) and everything it needs that core doesn't have, for anyone on Basix who wants a full desktop next to core's Sway.

Paths in this README are from the repository root. The repo-wide rules (layout depth, checksums, the workflow) are in the [main README](../../README.md).

## Status

Nothing here has been built on a Basix machine yet. The environment I wrote it in had no Basix system, no VM and no access to download.kde.org or download.qt.io. What has been checked:

- Every template loads with the real bpm 0.9 (`bpm query` on all 186 of them, with the use flags the installer sets).
- The full dependency walk for `kde-plasma-desktop`, `kde-plasma-extras`, `kde-apps-base`, `wpa_supplicant`, `tuigreet` and `flatpak` finds every package in this repo or core, and has no build cycles. That's 342 packages, 179 of them from here.
- Every patch applies (`patch --dry-run`) to the tagged upstream sources.
- `DE/KDE/tools/kde-gen --check` matches every committed template, and `DE/KDE/tools/kde-bump` ran end to end against a mock of the KDE and Qt mirrors, in this layout too.
- `kde-gen` and `kde-bump` are ShellCheck clean as POSIX sh.

The acceptance tests are still open: KWin finding the GPU and input through libudev-zero, an HTTPS fetch through KIO against LibreSSL, and a Plasma session starting from greetd.

### Tested combinations

| Date | Plasma | Qt | Frameworks | GPU | Machine | Result | Tested by |
|---|---|---|---|---|---|---|---|
| 2026-10-21 | 6.8.1 | 6.12.0 | 6.31.0 | - | none | Not built. Templates parse, dependency walk and patch dry-runs pass | builder |
| | | | | | | | |
| | | | | | | | |
| | | | | | | | |

### Known missing

- Network applet. plasma-nm needs NetworkManager, which Basix doesn't have. Use wpa_supplicant and dhcpcd.
- Bluetooth. bluedevil needs BlueZ.
- Printing. print-manager needs CUPS.
- Battery and power profiles. Without UPower, powerdevil can't show a battery or switch profiles.
- Auto-mounting removable drives in Dolphin. That's UDisks2.
- Intel microcode. Not packaged anywhere on Basix yet.
- The Energy page in Info Center, since Qt Graphs isn't packaged (see [Patches](#patches)).
- OCR in Spectacle until you add Tesseract language data, for example `eng.traineddata` in `/usr/share/tessdata`.

## Prerequisites

- A Basix system with bpm 0.9 or newer and a working `BPM_BUILDROOT`, which the stock `bpm.conf` has.
- Disk: 60 GiB free or more for Plasma. Qt and KWin build trees are big.
- RAM: at least 2 GiB per `BPM_JOBS` job, plus swap. Qt and KWin are large C++ builds.
- Time: roughly 8 to 14 hours of building.
- For `DE/KDE/tools/kde-bump`: `curl`, `b3sum`, `tar`, `xz`, `gzip`, and `git` for the GitHub watch.

RUN DE/KDE/tools/kde-bump (OR THE WORKFLOW) BEFORE THE FIRST INSTALL.

## Installing Plasma by hand

`basix-helper tasks` (Desktop) do all of this for you. By hand:

1. Put the repo above core in `/etc/bpm/repos.conf`, since bpm takes the first repo that has a package: `nullangst https://github.com/NullAngst/basix-personal.git main`
2. Pull it: `doas bpm pull`
3. Copy the KDE use flags into `/etc/bpm/package.use`: `grep -vE '^(#|$)' /var/db/bpm/repos/nullangst/DE/KDE/package.use | doas tee -a /etc/bpm/package.use`
4. In `/etc/bpm/bpm.conf`, take `-wayland` and `-pulseaudio` out of `BPM_USE`, since Plasma needs both. Use nano, vi, or whichever editor you prefer.
5. Build and install it: `doas bpm install kde-plasma-desktop kde-apps-base tuigreet`

   5.5. Optional extras (System Monitor, wallpapers, Aurorae, the Flatpak settings page, ksshaskpass, browser integration): `doas bpm install kde-plasma-extras`

6. Let the greeter account write tuigreet's cache, since `--remember` keeps the last user name there: `doas chown greeter:greeter /var/cache/tuigreet`
7. Point greetd at tuigreet and Plasma. In `/etc/greetd/config.toml`, under `[default_session]`, set: `command = "tuigreet --time --remember --asterisks --cmd 'dbus-run-session startplasma-wayland'"`
8. Turn off `run-user-dir` if it's enabled, since elogind manages `/run/user` now: `doas s6 disable run-user-dir`
9. Enable the session services: `doas s6 enable seatd elogind dbus polkit greetd`
10. Apply the service set: `doas s6 apply`
11. Reboot: `doas reboot`

PipeWire has to start inside the session. The installer drops `basix-pipewire-launcher` into `/etc/xdg/autostart`. By hand, add an autostart entry in System Settings that runs `pipewire`, `wireplumber` and `pipewire-pulse`, or whichever way you start them.

Now tuigreet shows up on tty 7, you log in, and Plasma starts on Wayland.

## Layout

```
DE/KDE/<pkg>/template   every KDE template, one folder per package, no category level
DE/KDE/manifest         input to kde-gen: versions, dependencies, cmake options, checksums
DE/KDE/package.use      use flags the installer and basix-helper copy into /etc/bpm/package.use
DE/KDE/patches/<pkg>/   patches for generated templates (kde-gen copies them in)
DE/KDE/tools/           kde-gen, kde-bump
Network/wpa_supplicant/ Wi-Fi, used with or without Plasma
```

Why no `kf6/`, `plasma/` and so on any more? Because bpm 0.9 only looks two folders deep for a package (`<repo>/*/*/<pkg>/template`). `DE/KDE/kf6/kconfig/template` is three deep, so bpm would never find it. See the main README.

The Qt modules, frameworks, Plasma and Gear packages, plus qca, qcoro, polkit-qt6, pulseaudio-qt, kquickimageeditor and cxx-rust-cssparser, are written by `DE/KDE/tools/kde-gen` from `DE/KDE/manifest`. Those templates start with a "Generated by DE/KDE/tools/kde-gen" line. Don't edit them, since the next run overwrites them, and the workflow's check job fails on a push that edits one. Everything else in `DE/KDE` (elogind, polkit, the X11 libraries, the meta packages, tuigreet, ...) is written by hand.

All 64 frameworks of the 6.31 release are here except the ones Plasma can't use on Basix: kapidox, kdav, kcontacts, kpeople (it needs kcontacts), kdnssd, modemmanager-qt, networkmanager-qt, bluez-qt, oxygen-icons and syndication. kcalendarcore, kmime, kplotting and ktexttemplate are packaged but nothing in the meta packages pulls them in, so they cost no build time unless you install them.

## Services

elogind, polkit and wpa_supplicant ship s6 sources the way core's greetd does: with the `services` use flag on (the default), `files/sources/<name>` and `<name>-log` go to `/usr/share/s6-rc/sources`. Logs land in `/var/log/<name>`.

wpa_supplicant reads `/etc/wpa_supplicant/s6.conf` for the interface. Leave `WPA_INTERFACE` empty and it picks the first wireless one. The network list goes in `/etc/wpa_supplicant/wpa_supplicant.conf`, for example from `wpa_passphrase "MySSID" | doas tee -a /etc/wpa_supplicant/wpa_supplicant.conf`, then delete the commented `#psk=` line it adds, since that one is the passphrase in plain text.

polkit runs as the `polkitd` system account and authenticates through Basix's `base-auth` PAM stack.

## Keeping versions current

I don't want versions sitting stale, so the versions live in one place and a tool moves them.

- `DE/KDE/manifest` has one version per set (`frameworks`, `plasma`, `gear`, `qt`) and one per standalone package, plus the checksums, licenses and per-package overrides.
- `DE/KDE/tools/kde-gen` writes every generated template from it. `kde-gen --check` fails if a committed template differs from a fresh run.
- `DE/KDE/tools/kde-bump` finds the newest stable release of each set (same major version) and each standalone package, downloads the tarballs, writes the BLAKE3 checksums and the licenses from each tarball's `LICENSES/` folder, resets revisions, updates the meta package versions and reruns `kde-gen`. It also fills empty checksums in hand-written templates anywhere in the repo, not only in `DE/KDE`.

To run it:

1. See what would change: `DE/KDE/tools/kde-bump --dry-run`
2. Run it: `DE/KDE/tools/kde-bump`

   2.5. To hold one set or package at a version for this run: `DE/KDE/tools/kde-bump --set plasma=6.8.1`

3. Review the table it prints and the diff: `git diff`
4. Build at least `qt6-base`, `kwin` and `plasma-workspace` on a Basix box before you push. A minor release (Plasma 6.8 to 6.9, Qt 6.12 to 6.13) can need new dependencies or refreshed patches, and `kde-bump` can't know that.

`DE/KDE/tools/kde-bump --fill` only fills empty checksums at the current versions. That's the one to run first on a fresh clone.

If a tarball of a set fails to download, the whole set stays at its old version, since half a set won't build. The table and the exit code (1) say so.

### The weekly workflow

`.github/workflows/kde-bump.yml` runs `kde-bump` every Monday and on demand, commits to a `kde-bump` branch and opens one pull request (or refreshes the open one) with the result table. On every push or pull request that touches `DE/KDE`, it also runs `kde-gen --check`. The main README has the one repository setting it needs.

## Checksums

bpm checks a BLAKE3 sum for every source file. KDE and Qt publish SHA-256, not BLAKE3. So `kde-bump` downloads each tarball, checks it against the `.sha256` file the KDE and Qt mirrors publish next to it, and only then writes its BLAKE3 sum. The table marks those as "sha256-checked".

GitHub archives and a few other hosts publish no checksum. For those the BLAKE3 sum is trust on first use, marked "unchecked". If that matters to you, compare against the project's signed release or tag before merging.

Already filled in when this was written, from GitHub release assets I could download and check: duktape, icu, libb2, libical, libsndfile, leptonica, python-cairo, boost-headers, eudev and noto-fonts. attr and libxkbcommon reuse the sums Kris and core already carry for the same files.

## The four gaps, and what was decided

1. logind. KWin's DRM backend only talks to logind (or ConsoleKit), so seatd alone won't do. elogind 255.24 is packaged with Alpine's musl patches, with s6 services, and core's `base-session` PAM file already loads `pam_elogind.so` as optional. seatd stays enabled, since core's greetd service depends on it.
2. udev. Everything builds against core's libudev-zero first. If KWin finds no GPU or input, `DE/KDE/eudev` is ready: `doas bpm install eudev`, then make its library the active one with `doas bpm a eudev /usr/lib/libudev.so.1`. mdevd stays the device manager either way. This is the first acceptance test, and it hasn't run.
3. TLS. Qt targets OpenSSL, and LibreSSL isn't a supported backend upstream. qt6-base builds against LibreSSL with OpenBSD ports' five TLS plugin patches. The acceptance test is an HTTPS fetch through KIO (`kioclient cat https://kde.org`). If that fails, don't ship it: stop and report.
4. gettext. KDE translations are off by default (`-nls`), so `po/` folders are removed before configure and gettext-tiny only has to satisfy ki18n's build. If it doesn't, `DE/KDE/gettext` is GNU gettext 1.0, not wired into anything yet. Installing it next to gettext-tiny makes their shared files bpm alternatives.

## Patches

Each patch starts with an Origin line and a Why line. Generated templates take theirs from `DE/KDE/patches/<pkg>/`.

| Package | Patch | Origin | Why |
|---|---|---|---|
| qt6-base | `0001-qt6-qtbase-lfs64.patch` | Alpine | musl 1.2.4 dropped the `*64` LFS aliases |
| qt6-base | `0002-qt6-qtbase-dns-resolver.patch` | Alpine | musl has no `res_nquery` family |
| qt6-base | `03-fix-wayland-deadlock.patch` | Alpine | Wayland event dispatch deadlock |
| qt6-base | `03-symlinks.patch` | Alpine | relative tool symlinks instead of DESTDIR paths |
| qt6-base | `10-libressl-tls.patch` | OpenBSD ports, 5 files joined, `-p0` to `-p1` | build and run the OpenSSL backend on LibreSSL |
| qt6-multimedia | `select.patch` | Alpine | musl needs `sys/select.h` |
| qca | `libressl.patch` | OpenBSD ports, `-p0` to `-p1` | LibreSSL paths in the ossl plugin |
| kinfocenter | `optional-qtgraphs.patch` | written here | Qt Graphs isn't packaged; build without the Energy page |
| elogind | `0001` to `0010` | Alpine | musl fixes for 255.24 |
| duktape | `no-libduktaped.patch` | Alpine | skip the duplicate debug library |
| lmdb | `lmdb-make.patch` | Alpine | versioned shared library, tools linked to it |
| giflib | three CVE patches | Alpine | security fixes |
| attr | `basename.patch` | kkrruumm/basix-personal | musl has no GNU `basename()` |
| pulseaudio | `link-libintl.patch` | Alpine | only link `-lintl` when needed |
| wpa_supplicant | three patches | Alpine | nl80211 and SAE fixes for 2.11 |

## Where this differs from the plan

- union is in the Plasma set. The plan had it out as a theming preview, but plasma-workspace 6.8 lists Union as REQUIRED, so it can't be left out. Union pulls in cxx-rust-cssparser, which is Rust.
- Three templates fetch Rust crates at build time (`allow_network=yes`): cxx-rust-cssparser, kdeplasma-addons (it requires Corrosion) and tuigreet. Everything else builds offline.
- Extra dependencies the CMake files require that weren't in the plan: union, cxx-rust-cssparser, corrosion, qca, qcoro, qtkeychain, qt6-speech, qt6-location, highway, kquickimageeditor, tesseract and leptonica (Spectacle links them unconditionally), libsecret, libcanberra, pulseaudio (client library only), libsndfile, attr, boost-headers, iso-codes, perl-uri, sassc, libsass, python-cairo, socat, the xcb-util libraries and five X11 client libraries for XWayland support.
- libxkbcommon is shadowed here. Core's template turns X11 off and can't build xkbregistry (no libxml2 in its build deps), and Qt and KWin need both. Drop my copy once core covers it.
- kpeople is out, since it needs kcontacts, which the plan already leaves out.
- KDE handbooks (`doc`) default to off for every KDE template through `DE/KDE/package.use`, since bpm's global `BPM_USE` turns `doc` on. Man pages stay on.

## If core gets Qt or KDE later

This repo sits above core in `repos.conf`, so a package with the same name here wins. If Kris adds Qt 6 to core under other names (say `qt6-qtbase`):

1. In `DE/KDE/manifest`, rename every `qt6-<module>` in the `depends` and `make` lines to his names.
2. Delete the `pkg qt6-*` blocks.
3. Regenerate: `DE/KDE/tools/kde-gen`
4. Delete the Qt folders: `git rm -r DE/KDE/qt6-*`

The same goes for the hand-written dependencies (elogind, polkit, icu and so on): delete the folder once core has the package. If core takes the same name, my copy shadows it until you do.

## License

BSD-3-Clause, see [LICENSE](../../LICENSE). Patches keep the licenses of the projects they came from.

Now the repo is pulled, the checksums are filled in, and `doas bpm install kde-plasma-desktop` builds a Plasma 6 Wayland session that starts from greetd.
