# Package request

Copy everything below the line into a new chat with Claude, fill in the request block, and send it. One block per package. Leave a field blank and Claude decides, then says what it decided.

---

Package this for my Basix repo, https://github.com/NullAngst/basix-personal. Follow the instructions after the request.

## Request

- Software:
- Upstream (repo or homepage):
- Version: latest stable
- Group folder: Utilities
- Package name (blank = upstream's name, lowercase):
- Kind (CLI, TUI, GUI, library, daemon):
- Display (wayland, x11, both, n/a): wayland
- Audio (pipewire, pulseaudio, alsa, n/a):
- Features I want on:
- Features I want off:
- s6 service (no, yes):
- Own system user (no, yes, you decide):
- Network during the build allowed (cargo, go): yes
- Prebuilt binaries OK when a source build isn't possible: no
- Man pages and docs: follow the global flags
- Keep it updated automatically (yes, no): yes
- Anything else:

## Instructions for Claude

Work through these in order. Don't skip the checks, they're the only verification this gets before I build it.

1. Clone these: https://github.com/NullAngst/basix-personal, https://github.com/kkrruumm/basix-packages (core) and https://github.com/kkrruumm/bpm. Check out the bpm tag matching `version=` in core's `sys/bpm/template`.
2. Run `Packaging/check-bpm --bpm <bpm clone> --core <core clone>`. If it exits 1, the guide is out of date for the bpm core ships: stop, show me the report, and tell me to run `UPDATE.md` first. If it exits 0, read `Packaging/GUIDE.md` and `Packaging/template.skel`.
3. Check the name isn't taken: no folder with that name 1 to 3 levels deep in core or basix-personal. If it exists, stop and tell me where.
4. Find the latest stable upstream release, not a pre-release. Prefer the release tarball upstream publishes over an auto-generated tag archive, since release tarballs carry generated files like `configure`. Use a git source only when there are no releases, and pin it to a commit.
5. Read upstream's build files (`meson.options` or `meson_options.txt`, `CMakeLists.txt`, `configure --help` or `configure.ac`, `Cargo.toml`, `go.mod`, `pyproject.toml`) and pick the build_style and options from those, not from memory. Turn tests off.
6. Map every dependency to a Basix package name and check each one exists in core or basix-personal. Anything missing gets packaged too, in the same group unless it clearly belongs elsewhere, and so on down. If that chain passes about five new packages, or reaches something Basix can't have (systemd, glibc-only code, an OpenSSL 3-only API), stop and report before writing it all.
7. Check Alpine aports first, then Chimera cports and Void srcpkgs, for patches this package needs on musl, LibreSSL or busybox at this version. Copy what's needed into `patches/` with the `Origin:` and `Why:` lines from the guide.
8. Write the template from `template.skel` in core's style: comments stripped except ones that explain something unusual, core's field order, 4-space indents, continuation lines starting with one space. Reuse core's flag names. Declare every flag the template tests.
9. Checksums: download each source and compute its BLAKE3 (`b3sum`, or `pip install blake3`). If downloads are blocked, write `checksum=""` exactly and say so. Never write a checksum you didn't compute.
10. Auto-update: if I said yes, write a `watch` file next to each new template so the pkg-bump workflow moves it to new upstream releases. The format is in `Packaging/pkg-bump --help` and `GUIDE.md`. Point it at the same place the tarball comes from: `type=git` with the repo URL for GitHub, Codeberg, GitLab and other git hosts, `type=pypi` or `type=crates` for those registries, `type=url` with a `match=` regex for a plain download listing. Tune `tag=`, `filter=`, `ignore=` or `series=` so pre-releases and dev series are skipped. Then run `Packaging/pkg-bump --dry-run <pkg>`: it has to find upstream's newest release and report "up to date" for the version you just packaged. If your sandbox can't reach that host, say so, since the first workflow run is then the first real test. If upstream has no machine-readable release source, or the template can't be bumped by changing `version=` alone (several sources on different version schemes, a multi-line checksum), skip the watch and tell me why. If I said no, write no watch file.
11. Run `Packaging/lint-template --core <core checkout> <package dirs>` until it reports no errors, and explain every warning left.
12. Deliver one tar, `basix-<pkg>.tar`, holding:
    - `README.md`: what was packaged and at which version, the build_style, the flags and their defaults, every new dependency package, any lines I need in `/etc/bpm/package.use` or `BPM_USE` (wayland, pulseaudio and so on), services and accounts, whether it auto-updates (and what the watch follows) or why not, the build command, and what you couldn't verify.
    - `repo/`: only the new or changed files, at their paths from the repo root, ready for `cp -r repo/. ~/basix-personal/`. No diff files of my repo. If a new group folder appears, update the table in the root `README.md` and include it. If `.github/workflows/pkg-bump.yml` or `Packaging/pkg-bump` is missing from my repo and a watch file was written, include them too.
13. Say plainly that none of this has been built. It's unbuilt until `bpm build` passes on Basix.
