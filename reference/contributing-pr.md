# Contributing to SUMMA via Pull Request

SUMMA is a community model with two active upstreams. This page covers
the choice between them, the standard fork-and-PR workflow, the coding
conventions enforced by reviewers, and what reviewers will look for.

---

## Step 0: Pick the right upstream

| If your contribution is | Submit to |
|-------------------------|-----------|
| Bug fix in stable physics, documentation, build system, parameter table | `NCAR/summa` |
| New model decision option, refactor, build improvements | Either, depending on where the related code already lives. Check both forks first. |
| Anything touching the SUNDIALS solver path, IDA, KINSOL, or the new numerics from Clark et al. 2021 | `CH-Earth/summa` |
| Multilingual / reorganized source tree work | `CH-Earth/summa` |

When in doubt, look at the most recent commits in both forks. Whichever
fork is actively merging in your area is the right target.

The two repos diverged enough that you cannot blindly cherry-pick
between them. Plan for one fork as your target; mirror the change
manually if the other fork would also benefit.

---

## Step 1: Fork and clone

On the GitHub web UI, click **Fork** on the upstream repo of choice.
Then on your local machine:

```bash
git clone https://github.com/<you>/summa.git
cd summa
git remote -v
# origin   https://github.com/<you>/summa.git (fetch)
# origin   https://github.com/<you>/summa.git (push)
```

Add the upstream as a tracked remote so you can pull in changes:

```bash
git remote add upstream https://github.com/NCAR/summa.git
# or
git remote add upstream https://github.com/CH-Earth/summa.git
```

---

## Step 2: Branch off `develop`

All new work branches off `develop`, never `master`.

```bash
git fetch upstream
git checkout -b feature/<short-name> upstream/develop
```

Naming conventions (from `docs/development/SUMMA_git_workflow.md`):

| Branch type | Naming |
|-------------|--------|
| Feature | `feature/<feature_name>` |
| Documentation | `docs/<doc_change_name>` |
| Hotfix (admin only) | `hotfix/<hotfix_name>` |
| Release (admin only) | `release/<release_name>` |
| Project-specific support (rare) | `support/SUMMA.<base>.<feature>` |

Keep the branch focused on one logical change. If you have a runoff
scheme and a soil temperature scheme, that is two feature branches and
two PRs. Reviewers will ask you to split a multi-feature PR.

---

## Step 3: Make changes following the coding conventions

The conventions are documented in
`docs/development/SUMMA_coding_conventions.md`. The points reviewers
notice:

### Variable names

- Self-describing names. `canopyEvap` not `ce`.
- camelBack: lowercase first letter, capital at each word boundary.
- One declaration per line, with units in a comment:

  ```fortran
  real(rk),intent(out) :: canopyEvap   ! canopy evaporation (kg m-2 s-1)
  ```

- Prefix local instantiations of global variables with their dimension
  family: `scalarAlbedo`, `mLayerDepth`, `iLayerHydCond`. The prefixes
  are:
  - `scalar`: a single value
  - `mLayer`: at the mid-point of a layer
  - `iLayer`: at the interface between two layers
- Always `implicit none`.

### Numbers, constants, and parameters

- Never hard-code physical constants in equations. Use the symbols in
  `build/source/dshare/multiconst.f90`. If you need a new constant, add
  it there.
- Model parameters live in the parameter files and are read in. During
  development you may declare them as `real(rk),parameter` at the top of
  a routine, but before the PR is merged they must be promoted to real
  inputs.

### Comments

- Comment-block header before every subroutine and function. Format:

  ```fortran
   ! ************************************************************************************************
   ! public subroutine init_metad: initialize metadata structures
   ! ************************************************************************************************
  ```

  The "public" / "private" / "internal" scope is part of the header.
- For routines that implement a non-trivial method, add a "Method"
  section in the header explaining the equations and assumptions. The
  `groundwatr` subroutine in `engine/groundwatr.f90` is the canonical
  example.
- Comment liberally inline. Never commit code with large commented-out
  blocks; use git history for that.

### Multiple statements on a line

Forbidden except:

- Error handling:
  ```fortran
  if(err>0)then; message=trim(message)//trim(cmessage); return; endif
  ```
- Long `case` statements:
  ```fortran
  case('scalarCosZenith'); get_ixmvar = iLookMVAR%scalarCosZenith   ! cosine of solar zenith angle (0-1)
  ```

### Indenting and whitespace

- One space per nesting level (subroutine inside module, `if` body,
  `do` body).
- Two blank lines before a subroutine header comment block; none
  between the block and the routine.
- Strip trailing whitespace. Most editors can do this on save.

### Module organization

- Every subroutine lives in a module, even if the module contains only
  one routine. This avoids the explicit-interface burden.
- `private` everything; `public` only what other modules call.
- `USE` other modules with `only:`:
  ```fortran
  USE vegliqflux_module, only: vegliqflux
  ```
- Use `intent(in)`, `intent(out)`, `intent(inout)` on every dummy
  argument.

### Licensing header

Every new source file starts with the GPLv3 header. Copy it verbatim
from `header.license` in the repo root. The text:

```fortran
! SUMMA - Structure for Unifying Multiple Modeling Alternatives
! Copyright (C) 2014-<year>  NCAR/RAL
!
! This program is free software: you can redistribute it and/or modify
! it under the terms of the GNU General Public License as published by
! the Free Software Foundation, either version 3 of the License, or
! (at your option) any later version.
!
! This program is distributed in the hope that it will be useful,
! but WITHOUT ANY WARRANTY; without even the implied warranty of
! MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
! GNU General Public License for more details.
!
! You should have received a copy of the GNU General Public License
! along with this program.  If not, see <http://www.gnu.org/licenses/>.
```

---

## Step 4: Commit and push

Atomic commits with one logical change each.

```bash
make clean                # do not commit object files or executables
git status
git add build/source/engine/snowAlbedo.f90 build/source/engine/mDecisions.f90
git commit
```

Commit messages follow the standard Git convention: a 50-character
imperative subject, a blank line, and a wrapped body that explains the
why.

```
Add Roesch-style snow albedo decay option

Adds 'roesch' as a new option for the alb_method decision.
Implements the Roesch et al. (2001) wind-and-temperature-dependent
decay function. Default behavior unchanged when alb_method is set
to conDecay or varDecay.
```

Push to your fork:

```bash
git push -u origin feature/<short-name>
```

---

## Step 5: Update `docs/whats-new.md`

Add a bullet under the `## Pre-release` section near the top of
`docs/whats-new.md`. This is required for the PR to be merged.

```
## Pre-release
### Major changes
- Add Roesch-style snow albedo decay option

### Minor changes
- 
```

Use `### Major changes` for new physics or new decisions; `### Minor
changes` for bug fixes, refactors, and small additions.

---

## Step 6: Test

Reviewers will ask whether you ran the test cases. The minimum:

```bash
cd case_study/reynolds
../../bin/summa.exe -m fileManager_reynoldsConstantDecayRate.txt
../../bin/summa.exe -m fileManager_reynoldsVariableDecayRate.txt
```

Both should run to completion. Check that `output/` contains the new
NetCDF files and that no NaN warnings appeared on stderr.

For physics changes, run the full test-cases archive (Reynolds,
Wrightwood, Senatorial, Aspen) and diff against the verification output
shipped with the archive. See `case-studies.md` for how to obtain and
run it.

For new decision options, demonstrate that:

1. The default behavior is bit-identical when the new option is not
   selected.
2. The new option produces a physically sensible result on at least
   one test case.

---

## Step 7: Open the pull request

On your fork's GitHub page, click **Compare & pull request** after
pushing. Set:

- **base repository**: `NCAR/summa` (or `CH-Earth/summa`)
- **base branch**: `develop`
- **head repository**: `<you>/summa`
- **compare branch**: `feature/<short-name>`

The PR title should be a short imperative summary. The body should
include:

- What changed and why.
- Which test cases were run, and the results (pass / numeric diff /
  expected behavior change).
- A link to the reference paper for any new physics scheme.
- A note on backward compatibility: confirm default behavior is
  unchanged when the new option is not selected.

The PR template (added in v3.1.0) prompts for these. Fill it in.

---

## Step 8: Iterate during review

The reviewer (a SUMMA admin or a domain expert) will:

- Check that the coding conventions are followed.
- Look for physics correctness, including the cited reference.
- Confirm test cases still pass.
- Suggest edits.

Push more commits to your branch in response. Each push automatically
attaches to the open PR.

When approved, an admin merges into `develop`. During the next release
cycle, `develop` merges into `master` for the official version release.
Your change appears in the `## Version X.Y.Z` block of `whats-new.md`
when that happens.

---

## Common contribution pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| PR includes object files or executables | Forgot `make clean` before committing | `git rm` the offending files; add a `.gitignore` entry if needed |
| Reviewer asks why your `develop` branch lags upstream | You did not rebase | `git fetch upstream && git rebase upstream/develop` and force-push to your branch |
| PR diff includes formatting churn | Editor reformatted unrelated files | `git checkout -- <unrelated_file>` to discard before re-staging |
| New physics added with no namelist guard | Default behavior changed | Wrap the new code in a `select case` on a new decision option, default off |
| Pull request mixes a bug fix and a new feature | Branch is not focused | Split into two branches and two PRs |
| Test cases produce different output | New code path is hit when it should not be | Confirm the new code is gated behind a decision; default option must reproduce prior output bit-identical |
| Reviewer asks for a reference for a new scheme | Physics added with no citation | Cite a peer-reviewed paper in the source comment block and the PR body |

---

## License and contributor expectations

SUMMA is GPLv3 (see `COPYING`). Every new source file must carry the
GPLv3 header from `header.license`. By contributing you agree to
license your changes under GPLv3.

The maintainers expect:

- A peer-reviewed reference for any new physics option.
- Backward compatibility: existing decision combinations reproduce
  prior output bit-identical.
- Documentation update in `docs/whats-new.md` and, if a new decision is
  added, in `docs/configuration/SUMMA_model_decisions.md`.
- Tests for any non-trivial change.

---

## Where to next

- Understand where your change lives in the source tree:
  `architecture.md`
- Decide which decision your option extends: `model-decisions.md`
- Verify your change against the reference experiments:
  `case-studies.md`
- Diagnose failures introduced by the change: `debugging.md`
