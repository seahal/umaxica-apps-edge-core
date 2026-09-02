# Toolchain harness bootstrap

## What was being verified

That the harness added to this repository actually runs end to end on the current
tree: dependency install, formatter, linter, type-aware linter, typechecker, the
repository-invariant test suite, and the git hooks. And that the evidence layout
check still behaves correctly after being moved out of a standalone script and
into that suite.

## Why

This repository was a project shell with no package manifest, so the evidence
check added earlier today had to be a bare-Node script and nothing ran it
automatically (see `2026-09-02-evidence-policy-rollout.md`). The harness closes
that gap and gives the three `core` units arriving from `umaxica-apps-edge`
something to land against, so the extraction is a directory move rather than a
per-manifest rewrite.

## Context

- Repository: `umaxica-apps-edge-core`
- Revision at time of check: `5a875c8` (main)
- Host: Linux, node v24.20.0, pnpm 12.2.1 (workspace pins pnpm 12.0.0 via `devEngines`)
- Date: 2026-09-02

## Decisions

- **Three-unit workspace**, mirroring `umaxica-apps-edge`: `packages: app/* com/* org/*`.
  Globs, not literal paths - pnpm treats a literal path matching no directory as a
  configuration error, and the units do not exist yet.
- **Core scope only**: oxfmt, oxlint (plain and type-aware), tsc, vitest,
  lefthook, GitHub Actions. knip, syncpack, cspell, dependency-cruiser and
  size-limit are deliberately absent - there is no code for them to analyse, and
  the repository states YAGNI as a binding principle.
- The `catalog:` block is copied whole from `umaxica-apps-edge` (31 entries,
  the union of what the three `core` manifests reference). It is not trimmed to
  what is currently importable, because the consumers are the units and they are
  not here yet; a missing key fails the install rather than falling back.

## What was added

`.npmrc`, `package.json`, `pnpm-workspace.yaml`, `tsconfig.json`,
`.oxfmtrc.json`, `.oxlintrc.json`, `lefthook.yml`,
`.github/workflows/integration.yaml` (five jobs: format, lint, lint-types,
typecheck, test), and `test/evidence-layout.test.ts`.

`scripts/check-evidence.mjs` was deleted; its rule set now lives in the test file
unchanged.

`AGENTS.md` was rewritten: its previous text described a repository with "no
package manifest, task runner, or supported build command", which stopped being
true.

## Commands run and what was observed

| Command                                                   | Observed                                                                                                                                            |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm install`                                            | 57 packages; `prepare` ran `lefthook install` → `sync hooks: pre-push, pre-commit`. The `app/*`/`com/*`/`org/*` globs matched nothing without error |
| `pnpm run format:check`                                   | **initially FAILED** on 3 files; passes after `pnpm exec oxfmt .`                                                                                   |
| `pnpm run lint`                                           | **initially FAILED**: `require-unicode-regexp` at `test/evidence-layout.test.ts:39`; passes after adding the `u` flag                               |
| `pnpm run lint:types`                                     | PASS                                                                                                                                                |
| `pnpm run typecheck`                                      | PASS (fan-out, then `tsc --noEmit` over `test/**/*`)                                                                                                |
| `pnpm run test`                                           | Test Files 1 passed (1); Tests 3 passed (3)                                                                                                         |
| `pnpm run check`                                          | PASS end to end                                                                                                                                     |
| `pnpm exec lefthook run pre-commit` (with a staged `.ts`) | all three jobs ran and passed: `format`, `lint`, `evidence-layout`                                                                                  |
| `pnpm exec vitest run --dir test` (violation fixture)     | 3 failed as expected - 1 subdirectory, 2 non-`.md`, 7 misnamed; fixture removed afterwards                                                          |
| `python3 -c "import yaml; yaml.safe_load(...)"`           | `lefthook.yml` and `integration.yaml` both parse                                                                                                    |

## The four real defects the harness caught

All four were mine, in files added earlier today, and none was visible before a
linter and a typechecker existed here. Two were found by this repository's gates
and two by re-running the other repositories' full `check` afterwards - which I
had not done when the files were first distributed.

1. **Formatting.** `README.md`, the prior evidence record and the test file all
   failed `oxfmt --check`. The README change is a single added trailing newline
   (`insertFinalNewline: true`); its content is unchanged.
2. **`require-unicode-regexp`.** The filename pattern lacked the `u` flag.
   `umaxica-apps-edge` carried the same defect - missed there because only Vitest
   and oxfmt had been run against its copy, never oxlint.
3. **`unicorn/no-array-sort`** (found by `portal`'s oxlint). `entries.sort()`
   mutates the array it reads; changed to `toSorted()`.
4. **TS7006 / TS2591, implicit `any` and missing Node types** (found by
   `umaxica-apps-cdn`'s `vp check`). `let entries;` was untyped. Fixed as
   `let entries: Dirent[]` in the TypeScript copies - but `umaxica-apps-cdn`
   turned out to have no `tsconfig.json` and no `@types/node` at all (it is a
   static-asset repository with only `jsconfig.json`), so a `.ts` test could
   never typecheck there. Its copy was rewritten as `test/evidence-layout.test.mjs`,
   plain JavaScript, which Vitest collects identically.

Changes 2-4 were applied to all copies across the workspace, and the negative
fixture was re-run after each: still exactly 10 violations reported, so none of
them changed behaviour.

## Assessment

PASS. Every gate runs and passes on the current tree, the hooks are installed and
fire, and the evidence check behaves identically in both directions after the
move into the test suite.

## Limitations

- **Not verified: CI.** `.github/workflows/integration.yaml` has never executed on
  GitHub Actions. What was verified is that it is valid YAML and that every
  command it invokes passes locally. Action versions (`actions/checkout@v5`,
  `pnpm/action-setup@v4`, `actions/setup-node@v5`) are unverified against the
  runner.
- **Not verified: `--frozen-lockfile`.** Install was run as
  `--no-frozen-lockfile` to generate the first `pnpm-lock.yaml`. CI uses the
  frozen form, which has not been exercised.
- **Not verified: pre-push.** `lefthook run pre-push` reported every command as
  "no matching push files" - there is nothing to push in this clone. The same
  commands were run directly and pass; the hook's behaviour on a real push is
  unverified.
- **Zero units.** Every `pnpm -r` fan-out currently reports "Scope: 0 of 1
  workspace projects". The fan-out half of each script is therefore untested;
  only the root half has run.
- The three `core` units have not moved. Whether their manifests install cleanly
  against this catalog is the next thing to check, and it is not checked here.

Nothing was committed; the change is left in the working tree.
