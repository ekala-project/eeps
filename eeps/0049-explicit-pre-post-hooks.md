---
EEP: 0049
Title: Automatic pre/post hooks in runPhase
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-10-12
---

# Motivation

Every phase function in stdenv must manually call `runHook preFoo` at the start and `runHook postFoo` at the end. This is pure boilerplate — every phase does it, every setup hook that overrides a phase must repeat it, and forgetting either call silently drops user hooks.

The pattern is universal enough that the framework should handle it. Moving pre/post hook dispatch into `runPhase` eliminates hundreds of redundant `runHook` calls across stdenv, setup hooks, and package definitions while guaranteeing that user-supplied hooks always fire.

# Specification

## Hook name derivation

`runPhase` derives hook names from the phase name by stripping the `Phase` suffix and capitalizing the first letter:

```
buildPhase    → preBuild / postBuild
configurePhase → preConfigure / postConfigure
installCheckPhase → preInstallCheck / postInstallCheck
```

Implemented as:

```bash
local phaseName="${curPhase%Phase}"
local preHook="pre${phaseName^}"
local postHook="post${phaseName^}"
```

## Execution order

`runPhase` calls hooks around the phase body:

```
runHook $preHook
eval "${!curPhase:-$curPhase}"   # the phase function/variable
runHook $postHook
```

## Interaction with skip toggles

The existing `dont*`/`do*` checks in `runPhase` return before any of the above runs. When a phase is skipped, its pre/post hooks are also skipped:

- `dontBuild = true` → `preBuild`, `buildPhase`, and `postBuild` are all skipped.
- `doCheck = false` → `preCheck`, `checkPhase`, and `postCheck` are all skipped.

## Custom phases

Phases inserted via `prePhases`, `preBuildPhases`, `postPhases`, etc. also receive automatic pre/post hooks if they follow the `fooPhase` naming convention.

## Phase functions no longer call runHook

All built-in phase functions (`unpackPhase`, `patchPhase`, `configurePhase`, `buildPhase`, `checkPhase`, `installPhase`, `fixupPhase`, `installCheckPhase`, `distPhase`) and all setup hooks (cmake, meson, ninja, cargo, python, etc.) have their `runHook pre*/post*` calls removed.

Package-level phase overrides in `.nix` files that previously called `runHook` will double-fire the hooks (once from `runPhase`, once from the inline call). Since hooks are idempotent in practice, this is harmless but redundant — the `runHook` calls should be removed from overrides as well.

# Examples

Before (every phase and setup hook):

```bash
buildPhase() {
    runHook preBuild
    make "${flagsArray[@]}"
    runHook postBuild
}
```

After (phase body only):

```bash
buildPhase() {
    make "${flagsArray[@]}"
}
```

# Implementation

Reference implementation: https://github.com/ekala-project/corepkgs/pull/231

The change is in `stdenv/generic/setup.sh`:

1. In `runPhase`, derive `preHook`/`postHook` names from the phase name.
2. Call `runHook "$preHook"` before and `runHook "$postHook"` after the phase body.
3. Remove `runHook pre*/post*` from all built-in phase functions.
4. Remove `runHook pre*/post*` from all setup hooks and package-level phase overrides treewide.
