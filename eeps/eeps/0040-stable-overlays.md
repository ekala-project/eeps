---
EEP: 0040
Title: Stable releases via frozen overlays
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Release
Created: 2026-07-02
---

# Motivation

As `corepkgs` adopts a rolling-release model with multi-versioned packages in `pkgs-many/`,
users need a mechanism to opt into version stability. Production deployments,
CI pipelines, and reproducible environments all benefit from a guarantee that
major versions of core toolchains (Node.js, Go, Python, etc.) remain constant across updates.

This proposal introduces a freeze-release workflow that generates Nix overlays 
pinning each `pkgs-many/` package to its current default variant. The overlay 
acts as a stable snapshot: users apply it and receive only patch-level changes
while the rest of `corepkgs` continues to roll forward. This approach:

- Decouples stability guarantees from the main package tree
- Requires no changes to individual package definitions
- Composes naturally with existing Nix overlay infrastructure
- Provides traceable metadata (release name, commit, timestamp) for auditability

# Detailed Specification

## Generated overlay format

The freeze-release tooling produces a single Nix file containing a standard overlay:

```nix
# Frozen Release: stable-2026.Q2
# Generated: 2026-05-18T21:33:25Z
# From commit: 874b560

final: prev: {
  abseil-cpp = prev.abseil-cpp.v202508;
  go = prev.go.v1_25;
  nodejs = prev.nodejs.v22;
  # ...
}
```

Each entry pins the top-level package attribute to the variant that was the default at generation time. Packages without a parseable `defaultSelector` or missing `variants.nix` are omitted.

## How it works

1. **Enumeration** -- scan all directories in `pkgs-many/`
2. **Detection** -- read each package's `default.nix` and extract the `defaultSelector` pattern (e.g. `p: p.v22`) via regex
3. **Validation** -- confirm the extracted variant exists in `variants.nix`
4. **Generation** -- emit a sorted overlay file with release metadata in comments

## Tooling

Two scripts implement the workflow:

**`scripts/freeze-release.sh`** -- shell wrapper for common usage:

```bash
# Default release name derived from current year/month
./scripts/freeze-release.sh

# Custom release name and output path
./scripts/freeze-release.sh "stable-2026.Q2" "./overlays/stable-2026-q2.nix"
```

The wrapper resolves the git commit, generates a timestamp, creates the output directory, and invokes the Nix script.

**`scripts/freeze-release.nix`** -- core implementation accepting these arguments:

- `releaseName` -- identifier embedded in the overlay header
- `outputPath` -- target file path
- `corePkgsPath` -- root of the `corepkgs` checkout
- `gitCommit`? -- short hash for traceability
- `timestamp`? -- ISO 8601 generation time

# Example Usage

## Applying a frozen overlay

```nix
# Most likely adopt a config option
import corepkgs {
  # Avoids conflating different releases
  config.release = "YY.MM";
}
```

```bash
# Ad-hoc build against a frozen release
nix-build -E '(import ./. { overlays = [ (import ./overlays/stable-2026-q2.nix) ]; }).nodejs'
```

## Recommended release cadence

- `stable-YY.MM` -- 6 month releases for tighter tracking (preferred, similar to Nixpkgs)
- `stable-YYYY.Q#` -- quarterly releases for production baselines
- `lts-vX.Y` -- long-term support snapshots

# Unresolved Questions

- Release cadence could be quicker or longer than 6 months with this model
- The `defaultSelector` regex is a heuristic; non-standard patterns will be silently skipped
- For packages which get promoted from a "normal" package to a `pkgs-many/` package, how to handle this?

# Future Work

- Integration with CI to auto-generate overlays on a schedule
- Validation tooling to diff two frozen overlays and summarize version changes
- Support for partial freezes? (e.g. freeze only a subset of `pkgs-many/`)

# Reference Implementation

[corepkgs@e9347f9](https://github.com/ekala-project/corepkgs/commit/e9347f929e191d74483e043af408331e40c2a6b2)
