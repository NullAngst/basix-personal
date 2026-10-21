# basix-personal

My bpm repository for Basix Linux, for anyone on Basix who wants what core doesn't carry: KDE Plasma 6, Wi-Fi, and a growing pile of utilities, each in its own folder.

[Basix-Installer](https://github.com/NullAngst/Basix-Installer) adds this repo on its own when you pick Plasma or Wi-Fi.

## What's here

| Folder | What | README |
|---|---|---|
| `DE/KDE/` | KDE Plasma 6 on Wayland: Qt 6, KDE Frameworks, Plasma, Konsole, Dolphin, Kate, Ark, and the libraries and services they need (elogind, polkit, ...). Versions kept current by `DE/KDE/tools/kde-bump` | [DE/KDE/README.md](DE/KDE/README.md) |
| `Network/wpa_supplicant/` | Wi-Fi, with an s6 service | [DE/KDE/README.md](DE/KDE/README.md#services) |
| `Utilities/` | Single programs, one folder each (kitty and so on) | this file |

## How bpm finds packages

bpm 0.9 looks for a package by its folder name, at most two folders deep into a repository:

```
<repo>/<pkg>/template
<repo>/<group>/<pkg>/template
<repo>/<group>/<subgroup>/<pkg>/template
```

So `Utilities/kitty/template` and `DE/KDE/kwin/template` work. `DE/KDE/kf6/kconfig/template` does not: it's one folder too deep, and bpm says the package doesn't exist. That's why `DE/KDE` has no `kf6/`, `plasma/` or `qt6/` folders.

A TEMPLATE MORE THAN TWO FOLDERS DEEP IS INVISIBLE TO BPM.

Two more rules that come from the same lookup:

- The folder name is the package name, exactly. `Utilities/kitty/` for `pkg_name=kitty`, not `Utilities/Kitty/`.
- Package names must be unique across the whole repo. bpm takes the first folder it finds with that name, so a duplicate shadows the other one without a word.

## Prerequisites

- A Basix system with bpm 0.9 or newer.
- `doas` rights, since installing writes to `/`.

## Using it

1. Put the repo above core in `/etc/bpm/repos.conf`, since bpm takes the first repo that has a package: `nullangst https://github.com/NullAngst/basix-personal.git main`
2. Pull it: `doas bpm pull`
3. Install what you want from it: `doas bpm install kitty`, or whichever package

For Plasma there are a few more steps (use flags, services, greeter). They're in [DE/KDE/README.md](DE/KDE/README.md#installing-plasma-by-hand), or `basix-helper tasks` does them for you.

## Adding a package

Using kitty as the example. It depends on what you're packaging, so adapt the group folder and template to it.

1. Make the folder, named exactly like the package: `mkdir -p Utilities/kitty`
2. Write `Utilities/kitty/template`. Copy a similar template from core (`basix-packages`) as a start, since the build styles and helper functions are the same. Leave the checksum empty: `checksum=""`
3. Put patches in `Utilities/kitty/patches/` and extra files (s6 services, configs) in `Utilities/kitty/files/`, the same as core does. Start each patch with an `Origin:` and a `Why:` line.
4. Commit and push.
5. Fill the checksum: Actions > kde-bump > Run workflow, then merge the pull request it opens. Or run `DE/KDE/tools/kde-bump --fill` locally and commit the result.

   5.5. Or on a Basix box with the repo pulled: `bpm checksum -w kitty`, then copy the changed template back into the repo.

6. Build it once on Basix before you rely on it: `doas bpm pull && doas bpm install kitty`

## Keeping versions current

KDE updates itself. Every Monday the workflow runs `DE/KDE/tools/kde-bump`, which moves Qt, Frameworks, Plasma, Gear and the standalone KDE libraries to their newest stable releases, fills the checksums, regenerates the templates and opens a pull request. Review it and merge it. The details are in [DE/KDE/README.md](DE/KDE/README.md#keeping-versions-current).

Everything else, `Utilities/` and the hand-written templates in `DE/KDE` included, is updated by hand. kde-bump doesn't know where kitty publishes releases. To bump one:

1. Change `version=` in its template, and set `revision=1`.
2. Empty the checksum: `checksum=""`
3. Commit and push.
4. Run the workflow (or wait for Monday) and merge the pull request, which fills the new checksum.
5. Build it on Basix: `doas bpm pull && doas bpm install kitty`

## The workflow

`.github/workflows/kde-bump.yml` does two jobs:

- Every Monday and on demand (Actions > kde-bump > Run workflow): runs `DE/KDE/tools/kde-bump`, which bumps KDE and fills every empty checksum in the repo, commits to a `kde-bump` branch and opens one pull request, or refreshes the open one, with a table of what changed.
- On every push or pull request that touches `DE/KDE`: runs `DE/KDE/tools/kde-gen --check`, which fails if a generated KDE template was edited by hand instead of through `DE/KDE/manifest`.

It uses the workflow's own token and the GitHub REST API, no gh CLI. One repository setting is needed for that token to open pull requests: Settings > Actions > General > Workflow permissions > "Allow GitHub Actions to create and approve pull requests". Without it the run fails at the last step, after pushing the branch, and the log says which setting to turn on.

## Checksums

bpm refuses to build a template with an empty checksum, so an unfilled one stops that package and nothing else. The installer only checks the packages it's about to install, so an unfinished `Utilities/` template doesn't block a Plasma install.

kde-bump downloads each source, checks it against the `.sha256` file the KDE and Qt mirrors publish when there is one, then writes its BLAKE3 sum. Hosts that publish no checksum (GitHub archives, most small projects) get trust on first use, and the pull request table marks those "unchecked". If that matters for a package, compare against its signed release before merging.

## License

BSD-3-Clause, the same as core and Kris's personal repo. See [LICENSE](LICENSE). Patches keep the licenses of the projects they came from.

Now the repo is in `repos.conf`, the checksums are filled in, and `doas bpm install` finds everything in it, KDE or kitty alike. From here, add to it as you see fit.
