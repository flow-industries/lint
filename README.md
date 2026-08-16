# @flow-industries/lint

Shared [Biome](https://biomejs.dev) configuration and a reusable CI workflow for Flow's TypeScript projects.

This repo is the single source of truth for code-quality rules across Flow's TypeScript repos (`auth`, `site`, `docs`, `ui`, `mcp`, `status`, `dash`, `talk`, `sense`). Each repo carries only a tiny stub that points here — the actual rules and CI steps live here once.

## Biome config

Two presets are published as `@flow-industries/lint`:

- `@flow-industries/lint/biome` — the self-contained core (formatter + recommended lint, no framework domain)
- `@flow-industries/lint/react` — additive: adds only Biome's `react` lint domain on top of the core

### Use it

```sh
bun add -d @flow-industries/lint @biomejs/biome@2.5.8 biome-anti-slop
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
  "lint": "biome check",
  "format": "biome format --write",
  "check": "biome check --write"
}
```

## anti-slop

The [anti-slop](https://github.com/kualta/anti-slop) rules — low-evidence TypeScript patterns like
unjustified type assertions, `unknown` smuggled through contracts, runtime `typeof` narrowing —
run as Biome GritQL plugins via the published `biome-anti-slop` package. No second linter.

Consumers add it as a third preset (Biome resolves the bare specifier from the consumer's
`node_modules`, and the plugin paths inside it resolve against the project root):

```jsonc
{
  "extends": [
    "@flow-industries/lint/biome",
    "@flow-industries/lint/react",
    "biome-anti-slop"
  ]
}
```

Where a rule must stay off (dash and mcp keep `no-shape-in-symbol-names` off — `shape` is the
chart model's domain noun), list the 14 wanted plugins individually under `plugins` instead of
extending the preset; per-rule configuration is not possible for GritQL plugins.

Requires Biome ≥2.5.0, hence the 2.5.8 pin. `npx biome-anti-slop doctor` verifies the rules load.

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
