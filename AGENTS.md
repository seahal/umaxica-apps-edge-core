# Repository Guidelines

This repository is the destination for the three `core` deployment units
currently living in `umaxica-apps-edge` (`{app,com,org}/core` — TanStack Start
on Vite, deployed to Cloudflare Workers). The units have **not** moved yet: what
exists today is the harness they will land against.

## Setup & commands

pnpm is the ONLY package manager. Never use npm, npx, yarn, or bun.

```sh
pnpm install          # `prepare` installs the lefthook hooks
pnpm run check        # everything: check:static + test
pnpm run check:static # format:check + lint + lint:types + typecheck
pnpm run test         # per-unit suites, then the repository-invariant suite
pnpm run format       # oxfmt, writes
pnpm run lint:fix     # oxlint --type-aware --fix
```

Every root script is a `pnpm -r --if-present` fan-out over the units followed by
the root's own pass. `--if-present` is deliberate: there are zero units today, and
the flag is what lets the same script contract work before and after the
extraction without being edited.

## Project structure

```
app/*  com/*  org/*   deployment units (none yet; see pnpm-workspace.yaml)
test/                 repository invariants — properties of the tree, not of a unit
```

`test/` is for assertions no single unit can make: layout, boundaries, config
parity. It is run by the root `pnpm run test` as `vitest run --dir test`. There
is deliberately no root `vitest.config.ts` — the units own theirs.

`tsconfig.json` at the root covers `test/**/*` only and IS typechecked: `pnpm run
typecheck` ends with `tsc --noEmit`. (This differs from `umaxica-apps-edge`,
where the root `typecheck` only fans out and nothing reads the root config.)

## Per-unit config — do not centralize

When the units arrive they each bring their own `.oxlintrc.json`,
`.oxfmtrc.json`, `tsconfig.json`, `vitest.config.ts` and `knip.jsonc`. Keep it
that way. Never replace a unit's copy with a root `extends` or a shared package:
a unit has to stay valid when it is lifted into its own repository, and a root
`extends` is exactly the coupling that breaks that. The root copies of
`.oxlintrc.json` / `.oxfmtrc.json` govern the root's own files.

Dependency versions are the one thing that IS shared, through the `catalog:` in
`pnpm-workspace.yaml`. The catalog is already populated with every entry the
three units reference, so their manifests need no edit when they move.

## Coding Style & Naming Conventions

Formatter and linter are `oxfmt` and `oxlint`, configured at the root. Run
`pnpm run format` after edits rather than hand-matching the style. Two-space
indentation, single quotes, trailing commas, 100-column print width (80 for
Markdown). `camelCase` for variables and functions, `PascalCase` for exported
types and components, kebab-case for filenames unless a framework says otherwise.

## Testing Guidelines

Vitest. Name a test file after the unit or behaviour it covers
(`router.test.ts`), and cover failure paths and boundary cases, not just the
happy path. Repository-wide invariants go in `test/`; unit behaviour goes in the
unit's own `test/`. Do not claim a coverage threshold until one is configured
and enforced.

## Evidence

Completed tests, validations, verifications, audits, security checks and
performance checks leave a short record in `evidence/` when retaining the result
is useful. Records describe work that was actually performed — never plans,
intentions, or unverified claims. A check that could not be completed is
recorded as such, with the reason and whatever was observed.

- `evidence/` is flat; no subdirectories.
- Only `.md` files.
- `YYYY-MM-DD-<topic>.md`, ISO date, lowercase hyphenated topic.
- No raw logs, screenshots, binaries, archives, dumps, generated reports or
  other large artifacts. Summarize them, and cite the commands, identifiers,
  hashes, measurements and excerpts that carry the result.
- Enforced by `pnpm run test` (`test/evidence-layout.test.ts`), and by the
  `evidence-layout` job in `lefthook.yml` pre-commit.

## Git hooks

Lefthook, installed by the root `prepare` script, so `pnpm install` is all a
fresh clone needs.

- pre-commit: oxfmt (restages) and oxlint on STAGED files, plus the evidence
  layout check. Cost scales with the commit, not the repository.
- pre-push: `format:check`, `lint`, `lint:types`, `typecheck`, `test` — piped
  and cheapest-first, so it stops at the first failure.
- Bypass a single commit or push with `--no-verify`.

## Commit & Pull Request Guidelines

Short, imperative subjects (`Add request router`); one change per commit.
Pull requests should state the problem, summarize the solution, list the
verification actually performed, and call out configuration or dependency
changes. Note any follow-up work or unverified behaviour rather than leaving it
implied.

## Security & Configuration

Never commit secrets or local environment files. The repository ignores `.env`
variants while allowing `.env.example`; document required variables there with
safe placeholder values.

## Design principle

YAGNI: build only what is needed now. The harness here is deliberately narrower
than `umaxica-apps-edge`'s — knip, syncpack, cspell, dependency-cruiser and
size-limit are all absent because there is no code for them to analyse yet. Add
each one when the thing it checks exists, not before.
