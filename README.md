# @flow-industries/lint

Shared [Biome](https://biomejs.dev) and [Oxlint](https://oxc.rs) configuration and a reusable CI workflow for Flow's TypeScript projects.

This repo is the single source of truth for code-quality rules across Flow's TypeScript repos (`auth`, `site`, `docs`, `ui`, `mcp`, `status`, `dash`, `talk`, `sense`). Each repo carries only a tiny stub that points here — the actual rules and CI steps live here once.

## Biome config

Two presets are published as `@flow-industries/lint`:

- `@flow-industries/lint/biome` — the self-contained core (formatter + recommended lint, no framework domain)
- `@flow-industries/lint/react` — additive: adds only Biome's `react` lint domain on top of the core

### Use it

```sh
bun add -d @flow-industries/lint @biomejs/biome@2.4.12
```

Add a `biome.json` to the repo root. React projects extend **both** presets (Biome merges them left to right):

```jsonc
{
  "extends": [
    "@flow-industries/lint/biome",
    "@flow-industries/lint/react"
  ],
  "files": { "includes": ["**", "!dist"] }
}
```

A non-React project extends just the core:

```jsonc
{ "extends": ["@flow-industries/lint/biome"], "files": { "includes": ["**", "!dist"] } }
```

The `react` preset is additive on purpose — it carries only the domain, never its own copy of the formatter/core rules. (A relative `extends` *inside* a published package does not resolve from a consumer's `node_modules`, so the core can't be pulled in transitively; listing both presets in the consumer is the reliable pattern.) Per-repo ignores (generated dirs, vendored code) go in the local stub via `files.includes` — Biome merges these arrays additively with the shared presets. Keep `@biomejs/biome` pinned to the exact version above so every repo lints with an identical rule set.

Recommended `package.json` scripts:

```jsonc
{
  "lint": "biome check && oxlint",
  "format": "biome format --write",
  "check": "biome check --write"
}
```

## Oxlint config (anti-slop)

Biome owns formatting and the general lint set. Oxlint runs one extra thing on top: **anti-slop**, a
vendored copy of [dmmulroy/anti-slop](https://github.com/dmmulroy/anti-slop) — rules that reject
low-evidence TypeScript (unjustified type assertions, `unknown` in signatures, runtime `typeof`
narrowing, widening away known types). The plugin source lives in `oxlint/anti-slop/` here and is
published as a bundled ESM file, so every repo lints against one copy instead of vendoring its own.

The two linters do not overlap: `oxlint.base.json` turns **every** built-in Oxlint category off, so
Oxlint reports anti-slop findings and nothing else.

### Use it

```sh
bun add -d @flow-industries/lint oxlint@1.78.0 @oxlint/plugins@1.78.0
```

Add an `.oxlintrc.json` to the repo root:

```jsonc
{
  "extends": ["./node_modules/@flow-industries/lint/oxlint"],
  "jsPlugins": [
    { "name": "anti-slop", "specifier": "@flow-industries/lint/anti-slop" }
  ],
  "ignorePatterns": [".claude/**", "dist/**"]
}
```

Unlike Biome, Oxlint's `extends` **does** resolve into a consumer's `node_modules`, so the rule list
is inherited rather than copied. Two things must stay local, though:

- **`jsPlugins`** — a plugin declared in the extended config is not inherited, so each repo repeats
  this one line. The bare package specifier resolves through the package's `exports`.
- **`ignorePatterns`** — Oxlint roots ignore globs at the directory holding the config that declares
  them, and rejects `..`, so patterns set inside `node_modules` could never match consumer files.

Keep `oxlint` and `@oxlint/plugins` pinned to the same exact version as each other; they ship in
lockstep and a mismatch fails plugin loading.

### Node requirement

Oxlint is a Rust binary but loads JS plugins through Node, so **Node must be on `PATH`** wherever
`bun run lint` runs — Bun alone is not enough. The published plugin is pre-bundled to plain ESM, so
any Node 18+ works; a `.ts` plugin would additionally need Node ≥22.18 for type stripping, and Node
refuses to strip types under `node_modules` at all. That is why this package ships `dist/*.mjs`.

## Reusable CI workflow

`.github/workflows/ts-check.yml` is a `workflow_call` workflow that sets up Bun, installs with a frozen lockfile, then runs `bun run lint` and a typecheck. Call it from a repo:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  check:
    uses: flow-industries/lint/.github/workflows/ts-check.yml@v1
    with:
      runner: flow-arc            # ubuntu-latest for public repos
      typecheck-cmd: "bun run typecheck"
```

Inputs:

| Input | Default | Notes |
|---|---|---|
| `runner` | `ubuntu-latest` | Use a self-hosted runner label (e.g. `flow-arc`) for private repos; public repos stay on `ubuntu-latest`. |
| `install-cmd` | `bun install --frozen-lockfile` | Override to add flags such as `--ignore-scripts` when a transitive dep's native build breaks on the runner. |
| `typecheck-cmd` | `bun run typecheck` | Override for repos whose typecheck needs codegen first. |
| `run-build` | `false` | Set `true` to also run `bun run build`. Skip for react-router apps — building under Bun hits the `react-dom/server.bun.js` `renderToPipeableStream` gap; let the docker job build under Node instead. |
| `node-version` | `22` | Node used by `actions/setup-node`. Oxlint loads the anti-slop plugin through Node, and the `flow-arc` runner image ships no Node of its own. |
