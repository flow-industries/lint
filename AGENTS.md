# AGENTS.md

This file provides guidance to agents working with code in this repository.

`@flow-industries/lint` publishes the shared Biome preset (`biome.base.json`, `biome.react.json`)
and the reusable `ts-check` workflow that every TypeScript repo in the Flow fleet runs in CI.
Consumers extend **both** presets and pin Biome to an exact version.

## Publishing

The package publishes to npm **automatically on a version bump**. Change `version` in
`package.json`, open a PR, merge it — the push to `main` runs `.github/workflows/publish.yml`,
which publishes that exact version through npm trusted publishing (GitHub OIDC; there is no npm
token stored anywhere). There is no build: `files` ships the two preset files directly.

- **Never run `npm publish` by hand.** The workflow is the only publisher. A manual publish from a
  stale checkout is how you get `You cannot publish over the previously published versions`.
- **The version is the trigger, not the file.** Merging any other `package.json` change is a no-op:
  the workflow checks the registry first and skips a version that already exists.
- **Do not rename `publish.yml`.** npm authenticates the run against that exact filename, so it can
  only change if the trusted publisher on npmjs.com changes with it.

## A rule change here reddens the whole fleet at once

Every TypeScript repo extends this preset, so tightening a lint rule fails CI everywhere the moment
consumers pick it up — not in this repo, where there is almost nothing to lint. Before publishing a
rule change, check what it does to the repos that consume it (`auth`, `voice`, `talk`, `site`,
`docs`, `ui`, `status`), and prefer landing the fix in those repos first.
