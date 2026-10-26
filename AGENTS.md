# AGENTS.md

## Project overview

Slidev turns a Markdown file (`slides.md`) into a Vite + Vue slide deck with a dev server, presenter mode, and PDF/PPTX/PNG export. This is a pnpm + turbo monorepo; `CONTRIBUTING.md` is the human guide, this file is the agent-facing version.

## Setup

- Node >= 22.12 and **pnpm 12.15.0 on PATH** (`npm i -g pnpm@12.15.0`). `corepack pnpm` is not enough: turbo looks up the `pnpm` binary and `pnpm build` fails with "Unable to find package manager binary".
- **Windows: use LF line endings**, as CI does. With `core.autocrlf=true`, lint reports 100+ `Insert ␍` errors and snapshot tests rewrite files. Fix with `git config core.autocrlf false`, then re-checkout.
- `pnpm install` rewrites tracked files (`skills-lock.json`, `packages/vscode/schema/*.json`). Don't commit those diffs unless the change is about them.

## Architecture

`@slidev/cli` (`packages/slidev/node/`, entry `cli.ts`) parses `slides.md` with `@slidev/parser`, generates virtual modules (`node/virtual/`) and Vite plugins (`node/vite/`), and the `@slidev/client` Vue app renders them. All packages share `@slidev/types`. Markdown syntax extensions live in `node/syntax/`; export (PDF/PPTX) in `node/commands/`.

- `packages/client/`: shipped as raw `.vue`/`.ts` source, **no build step**.
- `packages/vscode/`: VS Code extension and language server.
- `demo/starter/`: dev playground deck, and the source of create-app's template.
- `test/`: cross-package tests and fixtures.

## Building & running

```bash
pnpm install
pnpm build          # turbo, all packages -> dist/ (~25s)
pnpm typecheck      # vue-tsc --noEmit
pnpm lint           # eslint . --cache ; formatting is ESLint, there is no separate prettier step
pnpm verify         # build -> typecheck -> lint -> test (run before calling a change done)
pnpm demo:dev       # long-running dev server for demo/starter; don't block on it
```

## Testing

```bash
pnpm test                      # vitest over packages/* + test/ (~11s)
pnpm exec vitest run <path>    # single test file
```

- **Build before testing.** Workspace packages resolve to their `dist/`, and `test/api-snapshots.test.ts` refuses to run against a stale build of `types`/`parser`. After editing `packages/{types,parser,slidev}`, rerun `pnpm build`.
- Tests sit next to the code as `*.test.ts` (e.g. `node/resolver.test.ts`); cross-package tests go in `test/` with fixtures in `test/fixtures/`.
- Changing the public API of `@slidev/types` or `@slidev/parser` changes `test/__snapshots__/tsnapi/`. Update with `pnpm exec vitest run test/api-snapshots.test.ts -u` and review the diff; it is the API changelog.

## Code style & commits

- Style is `@antfu/eslint-config`: no semicolons, single quotes, 2 spaces. Fix with `pnpm lint:fix`; the pre-commit hook runs `eslint --fix` on staged files.
- Dependency versions live in named **catalogs** in `pnpm-workspace.yaml` (`dev`, `prod`, `frontend`, `types`, ...). Add deps as `"pkg": "catalog:<name>"`, never a literal version. Exception: `packages/vscode/package.json` uses literal versions on purpose.
- Commits use Conventional Commits with package scopes: `fix(cli): ...`, `feat(export): ...`, `test(e2e): ...`.

## PR instructions

- PRs target `main` and are squash-merged, so the **PR title becomes the commit subject**: write it as a Conventional Commit (`fix(parser): ...`). There is no PR template.
- Run `pnpm verify` first. CI runs tests on Ubuntu, Windows, and macOS plus a Cypress e2e job, and `autofix.ci` may push lint fixes onto the branch, so pull before pushing again.
- Docs changes go in `docs/` here, not in the `slidevjs/docs` mirror. A change to deck syntax or options usually needs a matching `docs/` update.
- Do not push or open PRs against `slidevjs/slidev` unless asked; work on the fork (`origin`).

## Do not touch

- **Generated files:** `skills/slidev/**` (generated from `docs/`, see `skills/GENERATION.md`; copied into the published CLI by `scripts/publish.mjs`), `packages/vscode/schema/*.json` (written by `scripts/schema.ts` on install), `packages/create-app/template/{slides.md,pages,snippets}` (copied from `demo/starter/`; edit that instead).
- `.claude/` and `.agents/` are gitignored and filled by `skills experimental_sync` at install. They are third-party contributor skills, not project docs, and not the user-facing `skills/` folder.
- `plans/` holds maintainer planning docs. Ignore unless asked.
- Never run `pnpm release`, `ci:publish`, or the vscode `publish` script.
