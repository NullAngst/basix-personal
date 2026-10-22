# Packaging

Everything needed to get new software packaged for this repo: a request form I fill in and hand to Claude, the reference it packages from, a commented template skeleton, a linter that checks templates before they go anywhere near a Basix box, and a checker that notices when bpm changes under all of it.

## What's in here

| File | What it's for |
|---|---|
| `REQUEST.md` | The form. Fill it in, paste it into a new chat with Claude |
| `GUIDE.md` | How bpm 0.9 builds a template, checked against the bpm 0.9.3 source and core's templates. Read it when a build breaks |
| `template.skel` | Every template field and function, commented. The starting point for each new template |
| `lint-template` | Static checks for templates: layout, names, checksums, unknown variables, bashisms, missing build tools, dependency names |
| `check-bpm` | Compares upstream bpm with what these files were written for, and reports what changed |
| `bpm-baseline` | The bpm version these files were last checked against. Written by `check-bpm`, never by hand |
| `UPDATE.md` | The update request. Paste it into a new chat with Claude when `check-bpm` reports changes |
| `pkg-bump` | Moves every template that has a `watch` file to its newest upstream release, with checksums |

NEVER ADD A FILE NAMED `template` TO THIS FOLDER. bpm would treat `Packaging/<x>/template` as a package, and the kde-bump workflow fills checksums in every file called `template` it finds. That's why the skeleton is `template.skel`.

## Prerequisites

- A clone of this repo and a clone of core: `git clone https://github.com/kkrruumm/basix-packages`
- bash 4 or newer for `lint-template` and `check-bpm`. Kubuntu has it. On Basix, `doas bpm install bash`.
- `git` and `sha256sum` for `check-bpm`, since it clones bpm and hashes its files.
- `curl`, `git`, `jq` and `b3sum` (or python3 with the `blake3` module) for `pkg-bump`. The workflows install what they need on their own.
- A Basix machine for the real test, since nothing gets built anywhere else.

## Requesting a package

1. Open `REQUEST.md`, copy everything below the line.
2. Fill in the request block. Blank fields are fine, Claude picks and tells you what it picked.
3. Paste it into a new chat with Claude.
4. You get back `basix-<pkg>.tar` with a `README.md` about the package and a `repo/` folder with the files at their real paths.

## Landing a package

1. Unpack it somewhere outside the repo: `mkdir -p ~/tmp/pkg && tar -xf basix-foo.tar -C ~/tmp/pkg`
2. Read `~/tmp/pkg/README.md`, since it lists new dependencies and any use flags you need to turn on.
3. Copy the files into your clone: `cp -r ~/tmp/pkg/repo/. ~/basix-personal/`
4. Lint it: `~/basix-personal/Packaging/lint-template --core ~/basix-packages ~/basix-personal/Utilities/foo`
5. Commit and push.
6. If the checksum came back as `checksum=""`, run the kde-bump workflow (Actions > kde-bump > Run workflow) and merge its pull request, since that's what fills empty checksums.

   6.5. Or on Basix: `doas bpm checksum -w foo`, then copy `/var/db/bpm/repos/nullangst/Utilities/foo/template` back into your clone.

7. On Basix, pull and build: `doas bpm pull && doas bpm build foo`
8. If it fails, `bpm log foo` has the whole log. Paste it to Claude along with the template.
9. Install it: `doas bpm install foo`

## Using lint-template

```sh
Packaging/lint-template                                   # every template in the repo
Packaging/lint-template Utilities/foo                     # one package
Packaging/lint-template --core ~/basix-packages Utilities # also check dependency names
Packaging/lint-template -q                                # errors only
```

E lines are things bpm refuses or silently gets wrong. W lines are worth reading. Exit status is 1 when there's an error, so it works in a workflow too. I use it with `--core` every time, since a typo in a dependency name is the most common way a template breaks.

It doesn't know whether the software configures, compiles on musl or runs. Only `bpm build` on Basix answers that.

## Keeping packages current

A package with a `watch` file next to its template bumps itself. `REQUEST.md` asks whether you want that, and Claude writes the watch file when upstream has a release source it can follow. To add one to an older package by hand:

1. Write `Utilities/foo/watch`, for a GitHub project: `printf 'type=git\nurl=https://github.com/owner/foo\n' > Utilities/foo/watch`. The other types and options are in `GUIDE.md` under Versions.
2. Check it finds the version you have: `Packaging/pkg-bump --dry-run foo` should say "up to date" or show a newer release.
3. Commit and push. From then on the pkg-bump workflow handles it.

`Packaging/pkg-bump` without `--dry-run` does the bump locally, same as the workflow.

## Workflows

Both live in `.github/workflows/` and need nothing besides the repo setting kde-bump already uses: Settings > Actions > General > Workflow permissions > "Allow GitHub Actions to create and approve pull requests".

| Workflow | When | What |
|---|---|---|
| `packaging.yml` | Every push or pull request touching a template, patch, files, watch file or `Packaging/` | Lints every template with dependency names checked against core, shellchecks the scripts. A red run means an E line or a shellcheck finding |
| `packaging.yml` | Wednesdays, or Actions > packaging > Run workflow | Runs `check-bpm`. When bpm changed it opens one issue with the report and a pointer to `UPDATE.md`, and closes it once `check-bpm` passes again |
| `pkg-bump.yml` | Tuesdays, or Actions > pkg-bump > Run workflow | Runs `pkg-bump`, opens one pull request with every bump, the result table and the lint output |

Pull requests opened by a workflow's own token don't start other workflows, so the lint check doesn't run on the pkg-bump or kde-bump pull requests. pkg-bump puts its own lint output in the pull request body for that reason.

## Keeping Packaging current

The guide, the skeleton and the linter describe one bpm version, the one in `bpm-baseline` (0.9.3 right now). When core ships a newer bpm, they can go stale without anything failing. `check-bpm` catches that.

1. Run it: `Packaging/check-bpm`, since it clones bpm and core on its own. Or point it at clones you already have: `Packaging/check-bpm --bpm ~/bpm --core ~/basix-packages`
2. Exit 0 means nothing changed. You're done.
3. Exit 1 means bpm changed, or the Packaging files don't match it. The report lists commits, changed files and functions, changed `bpm.conf` defaults, and every variable, hook, style or helper the linter or the guide is missing.
4. Open `UPDATE.md`, copy everything below the line, paste it into a new chat with Claude.
5. You get back `basix-packaging-update.tar`. Unpack it outside the repo and read its `README.md`, since it says what changed and whether any of your templates had to change.
6. Copy the files in: `cp -r ~/tmp/upd/repo/. ~/basix-personal/`
7. Check it took: `Packaging/check-bpm` should exit 0 now, and `Packaging/lint-template --core ~/basix-packages` should report no errors.
8. Commit and push.

The report also lists commits on bpm's master that core doesn't ship yet. Those are a heads-up. Nothing gets adapted to them until core picks them up.

`--write-baseline` is the last step of an update, after the files match the new bpm. RUNNING IT EARLY HIDES THE CHANGES FROM THE NEXT CHECK.

Now new software goes from a filled-in form to a linted template in the right folder, packages with a watch file follow upstream on their own, and an issue shows up when bpm moves underneath it all. Tinker with the forms and the guide as you see fit.
