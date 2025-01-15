---
Title: Rename dependency inputs to be more intuitive
Author: jonringer
Discussions-To: https://github.com/NixOS/nixpkgs/issues/28327
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-9-15
---

# Summary

The names `nativeBuildInputs` and `buildInputs` are often unintuitive to newcomers
and don't fit into the more normalized naming of `deps<Host><Target>` established
in [nixpkgs#26805](https://github.com/NixOS/nixpkgs/pull/26805). This proposal
tries to adopt more ergonomic naming of derivation dependencies based on learnings
from when nixpkgs#26805 was implemented in 2017. This is to lower the "oddity budget"
of using nixpkgs.

## Detailed Implementation

| Current Names    | Legacy Names | Proposed Names | Simple Description | Example    |
| --------         | -------      | ----------     | --------------     | ---------  |
| depsBuildBuild   |    | bootstrapDeps  | Dependency is a compiler, which produces things for buildPlatform. | `autotools` checking compiler compatibility may build and run built programs, these cannot target Host or Target platforms |
| depsBuildHost | nativeBuildInputs   |  commands  | Dependency is a tool which runs on the build platform, but produces code which runs on host | `cmake`, `meson`, `ninja` |
| depsBuildTarget  |    |  **removed**    | Dependency is a compiler, but produces code not for host, but for target platform. This is bad. | **none** |
| depsHostHost     |    |  **removed**    | Dependency runs on host platform, and can be used only during build. Currently depsBuildBuild is used instead. | **none** |
| depsHostTarget |  buildInputs  |  libraries    | Dependency is the same as this derivation. Runs on host, produces code for target platform | C/C++ libraries |
| depsTargetTarget |    |  **removed** (potentially)    | Dependency needs to be used/injected unaltered at runtime by package we're building. However, `hostTarget` should be able to be used in most siutations. | gobject-inspection |

Currently these are implemented as a "doubly linked list" where you can go forward and backward
between `stdenv`s which target different build/host/target triplets. Also, a spliced package
set is needed to pass a spliced drv which can provide the correct variant. E.g. `drv.__spliced.deps<host><target>`

```
# pkgs/stdenv/booter.nix

# depsHostTarget  is assumed to be `thisStage` in this context, thus omitted from adjacentPackages
adjacentPackages = if args.selfBuild or true then null else rec {
  pkgsBuildBuild = prevStage.buildPackages;
  pkgsBuildHost = prevStage;
  pkgsBuildTarget =
    if args.stdenv.targetPlatform == args.stdenv.hostPlatform
    then pkgsBuildHost
    else assert args.stdenv.hostPlatform == args.stdenv.buildPlatform; thisStage;
  pkgsHostHost =
    if args.stdenv.hostPlatform == args.stdenv.targetPlatform
    then thisStage
    else assert args.stdenv.buildPlatform == args.stdenv.hostPlatform; pkgsBuildHost;
  pkgsTargetTarget = nextStage;
};
```

`depsBuiltTarget` and `depsHostHost` were previously aliased to "whatever made the most
sense", however, have not come to be used in practice in nixpkgs. The only mentions
are of stdenv mkDerivation helper's which attempt to support spliced package sets
out of completeness.

## Example cross-compilation use case

The most common use case for this would be `pkgsCross.aarch64-multiplatform` in which case the build, host, target platforms from an x86_64 machine would be:

| Platform | System |
| -------- | ------ |
| Build    | "x86_64-linux" |
| Host     | "aarch64-linux" |
| Target   | "aarch64-linux" |

In the vast majority of cases, just `commands` and `libraries` need to be specified:

```nix
{ stdenv, cmake, openssl }:

stdenv.mkDerivation {
  ...

  # CMake is used at buildtime, to produce code for the host platform
  commands = [ cmake ];

  # openssl is used during the build for the host platform, producing code for the target platform
  libraries = [ openssl ];

  ...
}
```

## Migration plan

Migration plan consists of supporting "both" paradigms while the poly-repo fork
is still translating work from upstream nixpkgs. Eventually, existing dependency
names should be formally deprecated with a warning. It is currently unknown if
the existing input labels will be unsupported in the future.

## Future work

- Determine appropriate date for formally deprecating existing input names
- Determine if existing names should be supported for backwards compatibility with existing nix expressions

## Relevant topics

- https://github.com/NixOS/nixpkgs/issues/28327#issuecomment-879815573

## Adjacent issues

- Package splicing pain in general, how to improve the implementation:
  - https://discourse.nixos.org/t/frustrations-about-splicing/49607
  - https://github.com/NixOS/nixpkgs/issues/204303

- Propagated vs not propagated inputs:
  - https://discourse.nixos.org/t/the-papercut-thread-post-your-small-annoyances-confusions-here/3100/24

