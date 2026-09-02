# evidence/ convention rollout

> **Followed up the same day by `2026-09-02-harness-bootstrap.md`.** The migration
> described under "Why a script and not a test" below has now been carried out:
> the check lives in `test/evidence-layout.test.ts` and `scripts/check-evidence.mjs`
> is gone. The "Limitations" section below was accurate when written and is kept
> unedited as the record of that state; it no longer describes the repository.

## What was being verified

That the newly added `evidence/` layout check works in this repository - that it passes on the
current tree, that it actually fails when the layout is wrong, and that it is a no-op before the
directory exists.

## Why

`evidence/` was adopted across this workspace as a flat, Git-tracked record of verification that
was actually performed. This repository was created to receive the `core` deployment units
currently living in `umaxica-apps-edge` (`{app,com,org}/core`, TanStack Start on Vite), so it gets
the convention now rather than after the extraction, when the layout would already be set.

This file is also the first record under the convention here, which makes it its own first test
case: the check has to accept this file's own name.

## Context

- Repository: `umaxica-apps-edge-core`
- Revision at time of check: `5a875c8` (main)
- Host: Linux, node v24.20.0
- Date: 2026-09-02

## Repository state at time of check

A project shell, not yet an application. The tree contains `README.md`, `LICENSE`, `.gitignore`,
`AGENTS.md` and `CLAUDE.md` (an `@AGENTS.md` shim) and nothing else - no package manifest, no
lockfile, no source, no test runner, no CI workflow. History is a single `Initial commit`.

## What was added

- `scripts/check-evidence.mjs` - the same rule set used across this workspace, in bare Node with no
  dependencies.
- An `Evidence` section in `AGENTS.md`, including the migration note reproduced below.

## Why a script and not a test

Every other repository in this workspace that already owns a test runner carries this check as a
test file, wired to nothing because the existing suite collects it. This repository has no manifest
and no runner, so a Vitest file would install nothing, run nothing and enforce nothing. A bare-Node
script is the only form that can be executed here today, which is why it is the form that could
actually be verified below.

That is explicitly temporary. The `core` units bring Vitest, `vitest.config.ts` and a `test/`
directory with them (each already runs `test: vitest run`). When they land, the check should move
to `test/evidence-layout.test.ts` - matching the repository-invariant suite `umaxica-apps-edge`
keeps in `test/` - and `scripts/check-evidence.mjs` should be deleted. `evidence/` is a
repository-level concern, so it belongs at the repository root even if this becomes a workspace of
several units.

## Rules enforced

1. `evidence/` contains no subdirectories.
2. Every direct child is a regular file ending in `.md`.
3. Every filename matches
   `^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])-[a-z0-9]+(-[a-z0-9]+)*\.md$`.

A missing `evidence/` directory is deliberately not a failure, so the check is a no-op until the
directory exists.

## Commands run and what was observed

| Command                                                             | Observed                                           |
| ------------------------------------------------------------------- | -------------------------------------------------- |
| `node scripts/check-evidence.mjs` (no `evidence/`)                  | exit 0, no output                                  |
| `node scripts/check-evidence.mjs` (fixture with every failure mode) | exit 1, reported exactly 10 violations - see below |
| `node scripts/check-evidence.mjs` (two valid records only)          | `evidence/ layout OK (2 records)`, exit 0          |
| `git check-ignore -v evidence/... scripts/...`                      | neither path is ignored; both are trackable        |

The negative fixture contained a subdirectory (`2026-q3/`), `notes.txt`, `report.pdf`,
`2026-9-2-x.md`, `2026-13-01-x.md`, `2026-09-32-x.md`, `Sep-02-2026-x.md`, `2026-09-02-Topic.md`,
`2026-09-02-.md`, `2026-09-02-a--b.md`, and one valid record. All ten bad entries were reported and
the valid one was not. The fixture was removed afterwards.

## Assessment

PASS. The check behaves correctly in all three directions - clean, dirty, and absent. There was no
pre-existing check in this repository that adding it could break.

## Limitations

- **Nothing runs this automatically.** There is no CI workflow, no git hook and no aggregate command
  in this repository, so enforcement today is the documented manual command only. This is a real
  gap, recorded rather than hidden; it closes when the `core` extraction brings a test runner and
  the check moves into the suite.
- The extraction itself is not started and its shape is undecided. Whether this repository ends up
  as one unit or a workspace of three does not change the rules above, but it does decide where the
  check finally lives.
- `.gitignore` already ignores `*.log`, which incidentally makes one prohibited artifact type hard
  to commit. That is a coincidence, not enforcement, and the check does not rely on it.

Nothing was committed; the change is left in the working tree. This record covers the layout check
only - whether any given evidence record is honest is not mechanically checkable and remains a
review question.
