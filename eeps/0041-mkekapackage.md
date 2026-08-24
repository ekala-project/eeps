---
EEP: 0041
Title: mkEkaPackage -- scope-based dependency declaration
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-08-18
Requires: 0038
---

# Motivation

[EEP 0038](https://github.com/ekala-project/eeps/pull/38) proposed replacing
`buildInputs` and `nativeBuildInputs` with `libraries` and `commands` as
attrsets, and moving toward explicit package scope usage. This proposal builds
on that foundation by defining a concrete `mkEkaPackage` function that takes
scope-based dependency declaration to its logical conclusion: dependency
functions receive the correct package scope directly, eliminating the need
for spliced packages and `callPackage`-injected dependency arguments entirely.

Current issues with `mkDerivation`:
- Every dependency must be declared twice: once in the `callPackage` function
  arguments, and again in `nativeBuildInputs` or `buildInputs`
- Overriding individual dependencies requires list manipulation (filtering by
  value equality, mapping, concatenating) rather than targeting by name
- Spliced packages exist solely to work around `callPackage` injecting a single
  version of each package. The splice machinery in `splice.nix` is complex and
  a source of subtle cross-compilation bugs
- `nativeBuildInputs` and `buildInputs` are legacy names that don't communicate
  their cross-compilation semantics

# Detailed implementation

## `mkEkaPackage` as a scope member

Unlike `stdenv.mkDerivation`, `mkEkaPackage` is not attached to `stdenv`. It
is a member of the package scope itself. This is a deliberate break from the
`stdenv.mkDerivation` paradigm.

`mkEkaPackage` is an attrset with `__functor`, making it both callable and
introspectable. It carries references to `stdenv` and the package scopes so
that users can discover build details:
- `mkEkaPackage.stdenv.cc` -- the compiler
- `mkEkaPackage.stdenv.hostPlatform` -- platform information
- `mkEkaPackage.scopes.buildHost` -- the build-time package scope

## Automatic construction from scope machinery

`mkEkaPackage` is constructed automatically by the existing scope creation
functions. Both `makeScope` and `makeScopeWithSplicing'` already produce a
`self` fixpoint with `callPackage`, `newScope`, and `overrideScope`. The
scopes information needed by `mkEkaPackage` is already present in these
functions.

### Top-level scope

The top-level package set is assembled via overlays in `stage.nix`. The
`splice` overlay (`stdenv/splice.nix`) already defines `newScope`,
`callPackage`, and `makeScopeWithSplicing'`, and has access to all six
platform scopes via `pkgs.pkgsBuildHost`, `pkgs.pkgsHostTarget`, etc.

`mkEkaPackage` is added to the `splice` overlay alongside these:

```nix
# stdenv/splice.nix — added to the returned attrset
mkEkaPackage = {
  inherit (pkgs) stdenv;
  scopes = {
    buildBuild   = pkgs.pkgsBuildBuild;
    buildHost    = pkgs.pkgsBuildHost;
    buildTarget  = pkgs.pkgsBuildTarget;
    hostHost     = pkgs.pkgsHostHost;
    hostTarget   = pkgs.pkgsHostTarget;
    targetTarget = pkgs.pkgsTargetTarget;
  };

  __functor = self: fnOrAttrs:
    (import ./generic/make-eka-package.nix {
      inherit lib config;
      inherit (self) stdenv scopes;
    }).mkEkaPackage fnOrAttrs;
};
```

Because the `splice` overlay receives `pkgs` (the `self` of the top-level
fixpoint), `mkEkaPackage` automatically has access to the correct scopes.

### Sub-scopes via `makeScopeWithSplicing'`

Language ecosystems (python, perl, lua, xorg, llvm, etc.) create their package
scopes via `makeScopeWithSplicing'`. This function already receives
`otherSplices` which contains all six platform variants of the sub-scope, and
it already has access to `splicePackages` and `newScope` from the top-level.

`mkEkaPackage` can be constructed automatically inside `makeScopeWithSplicing'`
by adding it to the `self` fixpoint:

```nix
# In lib.makeScopeWithSplicing' — the self fixpoint gains mkEkaPackage
self = f self // {
  newScope = scope: newScope (spliced // scope);
  callPackage = newScope spliced;
  overrideScope = g: makeScopeWithSplicing' { ... } { f = extends g f; };
  packages = f;

  # New: mkEkaPackage constructed from the scope's own splices
  mkEkaPackage = {
    stdenv = self.stdenv or otherSplices.selfHostTarget.stdenv;
    scopes = {
      buildBuild   = otherSplices.selfBuildBuild;
      buildHost    = otherSplices.selfBuildHost;
      buildTarget  = otherSplices.selfBuildTarget;
      hostHost     = otherSplices.selfHostHost;
      hostTarget   = self;
      targetTarget = otherSplices.selfTargetTarget;
    };
    __functor = self: fnOrAttrs: /* ... */;
  };
};
```

This means every scope created with `makeScopeWithSplicing'` automatically
gets a correctly-configured `mkEkaPackage`. No manual wiring is needed per
ecosystem. A python package can use `mkEkaPackage` from its own scope, and the
`commands` function will receive `python3Packages.pkgsBuildHost` (which
contains python-specific build tools) rather than the top-level
`pkgsBuildHost`.

For `makeScope` (used by simpler ecosystems like R and Rust that don't need
cross-compilation splicing), `mkEkaPackage` is constructed with `self` as the
only scope (since there are no `otherSplices`):

```nix
# In lib.makeScope
self = f self // {
  newScope = scope: newScope (self // scope);
  callPackage = self.newScope { };
  overrideScope = g: makeScope newScope (extends g f);
  packages = f;

  mkEkaPackage = {
    stdenv = self.stdenv;
    scopes = {
      buildHost  = self;
      hostTarget = self;
      # remaining scopes fall back to self in non-cross contexts
    };
    __functor = self: fnOrAttrs: /* ... */;
  };
};
```

## Dependency declaration

`commands` and `libraries` are functions that receive the appropriate package
scope and return a named attrset of dependencies. The scope is the un-spliced
package set for the correct platform offset, so no `__spliced` extraction is
needed.

| Attribute              | Replaces                       | Scope received    |
|------------------------|--------------------------------|-------------------|
| `commands`             | `nativeBuildInputs`            | `pkgsBuildHost`   |
| `libraries`            | `buildInputs`                  | `pkgsHostTarget`  |
| `propagatedCommands`   | `propagatedNativeBuildInputs`  | `pkgsBuildHost`   |
| `propagatedLibraries`  | `propagatedBuildInputs`        | `pkgsHostTarget`  |
| `depsBuildBuild`       | `depsBuildBuild`               | `pkgsBuildBuild`  |
| `depsBuildTarget`      | `depsBuildTarget`              | `pkgsBuildTarget` |
| `depsHostHost`         | `depsHostHost`                 | `pkgsHostHost`    |
| `depsTargetTarget`     | `depsTargetTarget`             | `pkgsTargetTarget`|

When `mkEkaPackage` constructs the derivation, each attrset is flattened to a
list via `builtins.attrValues`, with `null` values filtered out and `getDev`
applied to each derivation. The flattened lists are passed to the underlying
`derivation` call (or `builtins.strictDerivation` when available).

## Flattening

Each scope-receiving function is called with its scope, producing an attrset.
The flattening step:

1. Filters out `null` values (enabling conditional deps)
2. Validates that remaining values are derivations, paths, or strings
3. Applies `getDev` to derivation values (matching current behavior)
4. Returns `builtins.attrValues` of the result

Attrsets are ordered by their keys, so the flattened list is deterministic and
reproducible regardless of how the user wrote the expression.

## overrideAttrs composition

Since `commands` and `libraries` are attribute values on the derivation args,
`overrideAttrs` composes with them naturally:

```nix
# Adding a dependency
pkg.overrideAttrs (prev: {
  libraries = scope: prev.libraries scope // { extra = scope.extra; };
})

# Removing a dependency
pkg.overrideAttrs (prev: {
  libraries = scope: removeAttrs (prev.libraries scope) [ "libxslt" ];
})

# Replacing a dependency
pkg.overrideAttrs (prev: {
  libraries = scope: prev.libraries scope // { openssl = myCustomOpenssl; };
})
```

This is a significant improvement over list manipulation. Dependencies are
targeted by name rather than by value equality.

## Language-specific helpers

Helpers like `buildPythonPackage` follow the same pattern: they are `__functor`
attrsets in their respective package scope that wrap `mkEkaPackage`. Because
`mkEkaPackage` is automatically constructed in each scope via
`makeScopeWithSplicing'`, the helper doesn't need to receive its internal
dependencies via `callPackage` — it pulls them from the scope that
`mkEkaPackage` already knows about.

```nix
# Simplified buildEkaPythonPackage — defined within the python package scope
buildEkaPythonPackage = {
  inherit (mkEkaPackage) stdenv scopes;
  inherit python;

  __functor = self: fnOrAttrs:
    mkEkaPackage (finalAttrs:
      let userAttrs = lib.toFunction fnOrAttrs finalAttrs;
      in userAttrs // {
        commands = scope:
          (userAttrs.commands or (_: {}) scope) // {
            inherit (scope) python wrapPython;
            # conditional hooks based on format, etc.
          };
        propagatedLibraries = scope:
          (userAttrs.propagatedLibraries or (_: {}) scope) // {
            inherit (scope) python;
          };
      }
    );
};
```

The helper's `callPackage` argument list shrinks from 30+ packages to just
`{ lib, mkEkaPackage, python }`. Internal dependencies like `pypaBuildHook`
and `pythonCatchConflictsHook` come from the scope passed to `commands`, not
from `callPackage` injection.

# Example usage

Old usage with `mkDerivation`:
```nix
{ lib, stdenv, fetchurl, openssl, zlib, pcre2, libxml2, libxslt,
  installShellFiles, removeReferencesTo, withPerl ? false, perl }:

stdenv.mkDerivation (finalAttrs: {
  pname = "nginx";
  version = "1.30.4";
  src = fetchurl {
    url = "https://nginx.org/download/nginx-${finalAttrs.version}.tar.gz";
    hash = "sha256-QmHckOnkfBxAQSdumqo9SOvi5mT3KOFPqVrmxn1XoIs=";
  };

  nativeBuildInputs = [ installShellFiles removeReferencesTo ];
  buildInputs = [ openssl zlib pcre2 libxml2 libxslt ]
    ++ lib.optional withPerl perl;
})
```

New usage with `mkEkaPackage`:
```nix
{ mkEkaPackage, fetchurl, lib, withPerl ? false }:

mkEkaPackage (finalAttrs: {
  pname = "nginx";
  version = "1.30.4";
  src = fetchurl {
    url = "https://nginx.org/download/nginx-${finalAttrs.version}.tar.gz";
    hash = "sha256-QmHckOnkfBxAQSdumqo9SOvi5mT3KOFPqVrmxn1XoIs=";
  };

  commands = scope: {
    inherit (scope) installShellFiles removeReferencesTo;
  };

  libraries = scope: {
    inherit (scope) openssl zlib pcre2 libxml2 libxslt;
  } // lib.optionalAttrs withPerl {
    inherit (scope) perl;
  };
})
```

The `callPackage` function argument list shrinks to only non-package items:
`mkEkaPackage`, `fetchurl`, `lib`, and configuration flags. All package
dependencies come from the scope. Platform details are available via
`mkEkaPackage.stdenv`.

Referencing commands in build phases works through `finalAttrs`:
```nix
mkEkaPackage (finalAttrs: {
  ...
  commands = scope: {
    cmake = scope.cmake.minimal;
  };

  checkPhase = ''
    ${lib.getBin finalAttrs.commands.cmake}/bin/ctest --test-dir build
  '';
})
```

## Conditional dependencies

Two patterns are supported:

```nix
# Null filtering -- good for single deps
libraries = scope: {
  inherit (scope) openssl zlib;
  perl = if withPerl then scope.perl else null;
};

# optionalAttrs -- good for groups
libraries = scope: {
  inherit (scope) openssl zlib;
} // lib.optionalAttrs withGui {
  inherit (scope) gtk3 cairo pango;
};
```

## Non-scope items

Items that aren't in the package scope can be added directly:

```nix
commands = scope: {
  inherit (scope) pkg-config ninja;
  mesonHook = scope.meson.configurePhaseHook;   # sub-attributes
  myLocalTool = someLocalDerivation;             # locally defined
};
```

# Cross-compilation

The current cross-compilation flow is:
1. `callPackage` injects spliced packages (carrying `__spliced` with all 6
   platform variants)
2. `make-derivation.nix` extracts the correct variant via
   `drv.__spliced.buildHost or drv`
3. `getDev` is applied to get the development output

With `mkEkaPackage`, the flow becomes:
1. `commands` receives `pkgsBuildHost` directly
2. `libraries` receives `pkgsHostTarget` directly
3. `getDev` is applied during flattening

Splicing is bypassed entirely. The packages are already the correct platform
variant because each dependency function receives the scope for its platform
offset. This eliminates the entire `splice.nix` machinery for packages using
`mkEkaPackage` and should improve cross-compilation evaluation performance.

## Setup hook compatibility

Setup hooks use `hostOffset` and `targetOffset` variables (provided by
`setup.sh`'s `activatePackage`) to determine their role in the build. These
offsets are assigned based on which dependency slot a package lands in, not how
it was declared in the Nix expression. Since `mkEkaPackage` maps `commands` to
the `nativeBuildInputs` slot `(-1, 0)`, `libraries` to the `buildInputs` slot
`(0, 1)`, etc., the offsets are identical to what `mkDerivation` produces.
Hooks like `cc-wrapper` and `pkg-config-wrapper` that branch on
`hostOffset`/`targetOffset` are unaffected.

# Unresolved questions

- Not all derivation inputs fit neatly into `commands` and `libraries`. Setup
  hooks are paths, not derivations. Shell hooks don't have a clear home.
  These can still be placed in the attrset as raw paths, but the naming motif
  doesn't perfectly describe them.
- `fetchurl` and other builder functions are not derivations and can't be
  `inherit`ed from a scope. These remain as `callPackage` function arguments.
  This limits how much the function argument list can shrink in practice.
- Whether `overrideCommands` and `overrideLibraries` convenience functions
  should exist as separate passthru attrs or whether `overrideAttrs` alone is
  sufficient.
- Dependency ordering via `builtins.attrValues` is alphabetical by key. While
  deterministic, this differs from the original author-specified order. In
  practice, ordering rarely affects correctness, but setup hook execution order
  could matter in edge cases.
- This is a large divergence from Nixpkgs paradigms. Packages written for
  `mkEkaPackage` will not be trivially portable back to Nixpkgs.
- Whether `mkEkaPackage` should be added to `lib.makeScope` as well or only
  to `lib.makeScopeWithSplicing'`. Simpler scopes using `makeScope` don't have
  `otherSplices`, so their `mkEkaPackage` would have limited cross-compilation
  support.

# Future work

- Modify `lib.makeScopeWithSplicing'` and `lib.makeScope` in nix-lib to
  automatically construct `mkEkaPackage` on every scope.
- Add `mkEkaPackage` to the top-level `splice` overlay in `stdenv/splice.nix`.
- Migrate package expressions to the new paradigm incrementally.
- When `builtins.strictDerivation` becomes available, adopt it as the
  underlying derivation call. The explicit scope-based dependency model
  aligns well with strict derivation's requirement that all inputs be
  explicitly listed.
- Evaluate whether splicing can be disabled entirely for package sets where
  all packages use `mkEkaPackage`.
- `__structuredAttrs = true` should be the default. Attrsets as dependency
  values cannot be stringified without structured attrs.

# Changes

N/A
