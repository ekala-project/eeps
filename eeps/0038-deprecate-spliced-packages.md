---
EEP: TBD
Title: Deprecate spliced packages and .override
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 03 Jan 2026
---

# Motivation

This proposal is meant to address both usability and ergonomics concerns when
using mkManyVariants, cross compilation, and dependency overriding inside of
an ekapkgs package set.

Current issues:
- mkManyVariants awkardness with overriding:
  - `buildInputs = [ ffmpeg.v9 ];` will not respect if someonen does `<pkg>.override { ffmpeg = ffmpeg.v8;}`
    - Here, ffmpeg will revert to using v9 as ffmpeg.v8 also exports the .v9 variant as well.
- Cross compilation names of `nativeBuildInputs` is unintuitive
  - Usage of `buildInputs` and `nativeBuildInputs` are largely holdovers from previous cross compilation implementations. However, these terms are generally considered unintuitive.

# Detailed implementation

The desire here is to replace `buildInputs` and `nativeBuildInputs` with `libraries`
and `commands` (and their propagated counterparts) respectively. Instead of a list,
the new attrs will also be attrsets which allow for selective replacement of commands.

# Example usage

Old usage. No mkManyVariants usage and spliced packages:
```nix
{ ...,
  cmakeMinimal,
  pkgsBuildHost,
}:

mkDerivation {
  ...
  # This is correctly dereferenced to the buildHost splice by mkDerivation by doing `drv.__spliced.buildHost or drv`
  nativeBuildInputs = [ cmakeMinimal ];

  # Previously, you would just use the input, this was always selected for hostTarget splice
  # To get the buildHost splice, you need to do additional work
  checkPhase = ''
    # This selects the cmake variant from the spliced package scope, which negates the usage of `.override`
    # If `.override { cmakeMinimal = <pkg>; }` is done, then there will still be a lingering reference to the original
    # cmakeMinimal package
    ${lib.getBin pkgsBuildHost.cmakeMinimal}/bin/ctest ...

    # Or the following:
    # cmakeMinimal.__spliced.buildHost is only available when "actually splicing", not availbe on non-cross builds
    # cmakeMinimal doesn't need to be spliced in non-cross builds so use it as default
    ${lib.getBin (cmakeMinimal.__spliced.buildHost or cmakeMinimal)}/bin/ctest ...
  ''
}

# When overriding
pkg.override { cmakeMinimal = <modified cmake>; };
```

Possible transitory usage (adopt attrs and mkManyVariants, but not explicit package set splice):
```nix
{ ...,
  cmake,
  pkgsBuildHost,
}:

mkDerivation (finalAttrs: rec {
  ...
  commands = {
    cmake = cmake.minimal;
  }

  # This is still awkward, and largely unchanged
  checkPhase = ''
    # This selects the cmake variant from the spliced package scope, doesn't respect overriding
    ${lib.getBin pkgsBuildHost.cmake.minimal}/bin/ctest ...

    # Or the following, however, this will not respect .override usage:
    # commands.cmake.__spliced.buildHost is only available when "actually splicing", not availbe on non-cross builds
    # commands.cmake.minimal doesn't need to be spliced in non-cross builds so use it as default
    ${lib.getBin (finalAttrs.commands.cmake.__spliced.buildHost or finalAttrs.commands.cmake)}/bin/ctest ...
  ''
})

# When overriding
pkg.overrideCommands { cmake = <modified cmake>; };
```

If the input in the nix expression is left unchanged (e.g. a specific variant isn't selected)
then `.override` still works as expected for the above use case.

Ideal usage (adopt attrs and explicit spliced package scope):
```nix
{ ...,
  # .override is deprecated altogether as expected way to affect existing drvs
  pkgsBuildHost,
}:

mkDerivation (finalAttrs: rec {
  ...
  commands = {
    cmake = pkgsBuildHost.cmake.minimal;
  }

  # Somewhat awkward to reference through finalAttrs, but no longer need to deal
  # with deferred splicing
  checkPhase = ''
    ${lib.getBin finalAttrs.commands.cmake}/bin/ctest ...
  ''
})

# When overriding, pass cmake variant for respective splice
pkg.overrideCommands { cmake = pkgsbuildHost.<modified cmake for buildHost>; };
```

If the package is dereferenced to it's expected splice, then the need for
`__splicedPackages` goes away along with the need for "spliced package scopes" which
need to map `<attr path>.<splice>` to `<splice>.<attr path>`. This should be a
significant improvement in cross compilation evaluation.

# Unintended benefits

Attrsets are ordered by their keys, you no longer have to worry about affecting
the .drv calculation by reording elements.

# Unresolved questions

Usage of `.override` was quite nice in many cases since it re-applied all packaging
logic of the expression, so invariance between attrs was less likely to happen.
This is somewhat mitigated by using `finalAttrs` by convention, however, still not
quite as powerful, especially when dealing with things like shared values between
two or more attrs.

This will be a fairly large divergence from Nixpkgs paradigms.

Not all drv inputs fit nicely into commands and libraries motifs. E.g. shellHooks

# Future work

- Change package expressions to new paradigm.
- Add `overrideCommands`, `overrideLibraries` and their propagated equivalents

# Changes

- `__structuredAttrs = true;` will be the default. Otherwise attrsets can't be stringified.

