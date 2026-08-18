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
is a member of the package scope itself, defined in `stage.nix` or a top-level
overlay. This is a deliberate break from the `stdenv.mkDerivation` paradigm.

`mkEkaPackage` is an attrset with `__functor`, making it both callable and
introspectable:

```nix
# Defined in stage.nix overlay
mkEkaPackage = {
  inherit stdenv;
  inherit scopes;

  __functor = self: fnOrAttrs:
    (import ./generic/make-eka-package.nix {
      inherit lib config;
      inherit (self) stdenv scopes;
    }).mkEkaPackage fnOrAttrs;
};
```

Where `scopes` is:
```nix
scopes = {
  buildBuild   = self.pkgsBuildBuild;
  buildHost    = self.pkgsBuildHost;
  buildTarget  = self.pkgsBuildTarget;
  hostHost     = self.pkgsHostHost;
  hostTarget   = self.pkgsHostTarget;
  targetTarget = self.pkgsTargetTarget;
};
```

Because `mkEkaPackage` lives in the package scope, it naturally has access to
the correct scopes for that package set. No factory parameter threading through
`stdenv` is needed. The `stdenv` derivation stays unchanged.

Users can inspect build details through `mkEkaPackage`:
- `mkEkaPackage.stdenv.cc` -- the compiler
- `mkEkaPackage.stdenv.hostPlatform` -- platform information
- `mkEkaPackage.stdenv.isLinux` -- convenience flags
- `mkEkaPackage.scopes.buildHost` -- the package scope for build-time tools

## Dependency declaration

`mkEkaPackage` introduces `commands` and `libraries` as functions that receive
the appropriate package scope and return a named attrset of dependencies. The
scope is the un-spliced package set for the correct platform offset, so no
`__spliced` extraction is needed.

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
the helper lives in the scope, it can pull its internal dependencies (hooks,
wrappers, etc.) from the same scope rather than receiving them via a long
`callPackage` argument list.

```nix
# Simplified buildEkaPythonPackage
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
`{ lib, mkEkaPackage, python }`. All internal dependencies (hooks like
`pypaBuildHook`, `pythonCatchConflictsHook`, etc.) come from the scope passed
to `commands`, not from `callPackage` injection.

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

# Future work

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
