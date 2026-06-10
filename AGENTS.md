# Development Rules

## Repo Structure

- **4 packages** under `packages/*`, lockstep versioned (all share same version, currently 0.79.0):
  - `packages/tui` (`@earendil-works/pi-tui`) — terminal UI library (differential rendering)
  - `packages/ai` (`@earendil-works/pi-ai`) — unified multi-provider LLM API
  - `packages/agent` (`@earendil-works/pi-agent-core`) — agent runtime (tool calling, state, transport)
  - `packages/coding-agent` (`@earendil-works/pi-coding-agent`) — interactive coding agent CLI (the `pi` binary)
- **Build order matters**: `tui → ai → agent → coding-agent` (each depends on the previous)
- **Config directory**: `.pi/` at repo root and `~/.pi/` at user home
- **Extra workspace dirs** in root `package.json` (example extensions like gondolin, sandbox, with-deps)

## Toolchain

- **`tsgo`** is the TypeScript compiler (not `tsc`). Wraps TypeScript with strip-only emit. Build with `tsgo -p tsconfig.build.json`.
- **`tsx`** used for running `.ts` files directly (dev, scripts, `pi-test.sh`).
- **`esbuild`** used for browser smoke check (bundling `scripts/browser-smoke-entry.ts` for browser platform).
- **biome** for lint + format: 3-space tab indent, 120 line width. Config at `biome.json`.
  - `noNonNullAssertion` = off, `noExplicitAny` = off, `noEmptyInterface` = off, `useNodejsImportProtocol` = off
- **Erasable TypeScript only** (`erasableSyntaxOnly: true` in tsconfig.base.json). No `enum`, no `namespace`/`module`, no parameter properties, no `import =`/`export =`.
- **Node16 module resolution**, `.ts` extensions in source imports.
- **No relative `.js` imports** in `.ts` files (enforced by `scripts/check-ts-relative-imports.mjs`). Use full `.ts` extension.
- **`npm run check`** runs: `biome check` → pinned-deps check → ts-relative-imports check → shrinkwrap check → `tsgo --noEmit` typecheck → browser-smoke esbuild check.

## Dev Commands

```bash
npm install --ignore-scripts  # hydrate/update deps
npm run build                 # builds all 4 packages in order
npm run check                 # lint + format + typecheck + integrity checks
./test.sh                     # runs all tests (unsets all API keys first)
./pi-test.sh                  # run pi from source (uses tsx)
./pi-test.sh --no-env        # run pi without API keys
```

- Run a single test file from a package root:
  - ai/agent/coding-agent: `node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts`
  - tui: `node --test test/specific.test.ts`
- For `packages/coding-agent/test/suite/`: use `test/suite/harness.ts` + faux provider. No real API keys.
- Issue-specific regressions go in `packages/coding-agent/test/suite/regressions/<issue-number>-<short-slug>.test.ts`.
- Never commit unless asked.

## Coding Conventions

- No `any` unless absolutely necessary.
- No inline imports (`await import()`, `import("pkg").Type`, dynamic type imports). Top-level only.
- Never hardcode key checks (e.g. `matchesKey(keyData, "ctrl+x")`). Add defaults to `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS`.
- `packages/ai/src/models.generated.ts` and `image-models.generated.ts` are generated. Never edit directly; update `packages/ai/scripts/generate-models.ts` or `generate-image-models.ts`, then regenerate.
- Read files in full before wide-ranging changes. Don't rely on search snippets.
- Inline single-line helpers with one call site.
- Check `node_modules` for external API types; don't guess.

## Dependency Security

- Direct external deps pinned to exact versions. Internal workspace deps use `^` ranges.
- Hydrate with `npm install --ignore-scripts`; CI uses `npm ci --ignore-scripts`.
- Lockfile is source of truth. Pre-commit blocks lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`.
- `packages/coding-agent/npm-shrinkwrap.json` generated from root lockfile. Regenerate via `node scripts/generate-coding-agent-shrinkwrap.mjs` (verify with `--check` or `npm run check`).
- New deps with lifecycle scripts require review + explicit allowlist entry in the shrinkwrap generator script.

## Git (Multi-Session Safety)

Multiple pi sessions may run in this cwd simultaneously. Never:
- `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git add -A`, `git add .`, `git commit --no-verify`

When committing:
- Stage only files YOU changed: `git add <path1> <path2>`
- Run `git status` first to verify
- `packages/ai/src/models.generated.ts` may always be included
- Format: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>`
- Never force push. On rebase conflicts, only resolve files you modified; abort and ask otherwise.

## Issues / PRs

- New contributor issues/PRs auto-closed. Maintainers review daily. See `CONTRIBUTING.md`.
- `lgtmi` = future issues stay open; `lgtm` = future issues + PRs stay open.
- When reviewing PRs: inspect via `gh pr view`, `gh pr diff`, `gh api`, `git show` — never switch branches.
- When posting comments: write to temp file, post with `gh issue/pr comment --body-file`. End with AI-generated disclaimer line.
- Issue labels: `pkg:agent`, `pkg:ai`, `pkg:coding-agent`, `pkg:tui`.
- Closing issues via commit: `closes #1, closes #2` (repeat keyword per issue).

## Changelog

- `packages/*/CHANGELOG.md` — one per package.
- New entries go under `## [Unreleased]`, never duplicate subsections.
- Released sections are immutable.
- Internal attribution: `Fixed foo bar ([#123](https://github.com/earendil-works/pi-mono/issues/123))`
- External: `Added X ([#456](https://github.com/earendil-works/pi-mono/pull/456) by [@user](https://github.com/user))`

## Releasing

- **Lockstep versioning**: all packages share one version. `patch` = fixes/additions, `minor` = breaking changes. No major releases.
- Flow: `./scripts/release.mjs <patch|minor|x.y.z>` handles bump → changelog update → artifact regen → check → commit+tag → push.
- Env vars for release: `PI_ALLOW_LOCKFILE_CHANGE=1 npm_config_min_release_age=0 npm run release:patch`
- After tag push, CI publishes npm packages via trusted publishing (no local `npm publish`).
- If CI fails: inspect `publish-npm` job; rerun tag workflow after fix (script is idempotent). Don't rerun release script for same version.

## Testing Interactive Mode with tmux

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux send-keys -t pi-test "your prompt here" Enter
tmux send-keys -t pi-test Escape
tmux kill-session -t pi-test
```

## User Override

If user instructions conflict with any rule here, ask for explicit confirmation before overriding.
