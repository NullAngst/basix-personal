# Packaging update request

Copy everything below the line into a new chat with Claude and send it. Run it when the packaging workflow opens its "bpm changed" issue, when `check-bpm` exits with 1 locally, or every so often to be safe. Nothing to fill in.

---

Check whether bpm or core changed in ways my Basix packaging files don't account for, and update them. The files are in `Packaging/` of https://github.com/NullAngst/basix-personal. Follow the instructions below.

## Instructions for Claude

Work through these in order. The point is that `GUIDE.md`, `template.skel`, `lint-template` and `REQUEST.md` describe the bpm that core actually ships, and that the templates in my repo still work with it.

1. Clone https://github.com/NullAngst/basix-personal, https://github.com/kkrruumm/basix-packages (core) and https://github.com/kkrruumm/bpm (full history, with tags).
2. Run `Packaging/check-bpm --bpm <bpm clone> --core <core clone>` and read the whole report. It compares the bpm version core ships (`version=` in core's `sys/bpm/template`) against `Packaging/bpm-baseline`, and checks the Packaging files against that bpm.
3. If it exits 0, also do step 7 and step 8. If those turn up nothing, tell me nothing changed and stop. Don't send a tar for no changes.
4. Read the real diff, not only the report: `git -C <bpm clone> diff <baseline commit> <shipped ref> -- bpm lib etc`. The report says where to look, the diff says what changed. For every change, decide what it means for a template author: new or removed variables, hooks, styles, helpers, changed defaults in `etc/bpm.conf`, changed lookup depth, changed fetch, checksum, extract, patch, sandbox or build root behavior, changed install behavior (configs, accounts, alternatives, hooks).
5. Update the Packaging files to match the shipped version:
   - `GUIDE.md`: every section the change touches, plus the version it says it was checked against (bpm 0.9.3 today). Keep its style: plain statements, tables for lists, no history notes about what bpm used to do unless an old habit now breaks something.
   - `template.skel`: comments, new variables (commented out with an explanation), quoted defaults like BPM_USE and the base package list.
   - `lint-template`: `TMPL_VARS`, `HOOKS`, `STYLES`, the use flag helpers copied into `EVAL_SH` (copy them verbatim from `lib/common.sh`, `pkg-bump` reads the same copy), `BADHOOK` (drop a name once bpm calls it), `HINT`, the style-specific `need` checks for any new style, and the layout depth check if `pkg_find` changed.
   - `REQUEST.md` and `UPDATE.md`: anything that names a bpm version or a step that changed.
   - `pkg-bump`: anything that changed how `version=`, `revision=`, `checksum=` or `dist_files` work.
   - `README.md` in `Packaging/`, and the root `README.md` if the layout rules or install steps changed.
   - `.github/workflows/packaging.yml` and `pkg-bump.yml`: still valid, and every action on its current major version (check each action's releases, a major retired by GitHub breaks the run). Run `actionlint` if you can.
6. If bpm's master has commits the shipped version doesn't, list them in your report and say which would need Packaging changes once core ships them. Don't adapt to them yet.
7. Look at core's templates changed since the baseline date (`date` in `bpm-baseline`): `git -C <core clone> log --since=<date> --name-only -- '*/template'`. Note new conventions worth copying (new flag names, new ways of handling cargo, go, python or services, new global hooks under `files/hooks`). Recheck the "system you're packaging for" list in `GUIDE.md` (musl, LibreSSL, what's missing) against what core and my repo carry now.
8. Run `Packaging/lint-template --core <core clone>` over my whole repo and `Packaging/lint-template --repo <core clone> --core <core clone>` over core. My repo has to come out with no errors. If a bpm change means my existing templates need edits (a renamed variable, a new required field), make them and list each one.
9. When everything matches, run `Packaging/check-bpm --bpm <bpm clone> --core <core clone> --write-baseline`, then run `Packaging/check-bpm` again and make sure it exits 0.
10. Run `shellcheck -s bash` on `lint-template`, `check-bpm` and `pkg-bump` if any script changed.
11. Deliver one tar, `basix-packaging-update.tar`, holding:
    - `README.md`: the bpm version before and after, what changed in bpm that matters for packaging, what you changed in each Packaging file, any templates in my repo you changed and why, the upcoming master commits from step 6, and anything you couldn't verify.
    - `repo/`: only the new or changed files, at their paths from the repo root (`Packaging/...`, `.github/workflows/...`, plus any templates), ready for `cp -r repo/. ~/basix-personal/`. Include the new `Packaging/bpm-baseline`. No diff files.
12. Say plainly that none of the templates were built. A bpm change that alters build behavior is only proven by `bpm build` on Basix.
