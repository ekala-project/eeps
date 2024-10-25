---
Title: Allow for many variants to be exposed by a package
Author: jonringer
Discussions-To: 
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-9-19
---

# Summary

Generally multiple versions or variants are introduced by adding a suffix to similar package
(e.g. `ffmpeg_4`), and generally a default is exposed for the prefered package
version to be used across the package set. However, this is less than ideal
as you pollute a package scope with multiple variants which are treated as
separate packages in terms of overriding (e.g. overlays), deprecation policies,
and it is unintuitive to users how packages expose these versions (e.g. `ffmpeg_4` vs `nixVersions.nix_2_18`).

## Detailed Implementation

Expose `mkPolyPkg` which allows for constructing this variant scheme in an uniform style.
`mkPolyPkg` in this fashion can be thought of as a partially applied function, where
version information is passed as one set of arguments before passing the nix expression
to callPackage.

Example PR: https://github.com/jonringer/core-pkgs/pull/5

```nix
{
  ... 
  # top-level.nix
  mkPolyPkg = callPackage ./build-support/mkgeneric { };
  ... 
}
```

```nix
# build-support/mkgeneric/default.nix

{ lib, config }:

{
# Intended to be an attrset of { "<exposed version>" = { version = "<full version>"; src = <path>; } }
# or a file containing such version information
# Type: AttrSet AttrSet
variants,

# Similar to variants arg, but instead contain deprecation and removal messages
# Only added when `config.allowAliases` is true
# This is passed the variants attr set to allow for directly referencing the version entries
# Type: AttrSet AttrSet -> AttrSet AttrSet.
aliases ? { ... }: { },

# A "projection" from the version set to a version to be used as the default
# Type: AttrSet package -> package
defaultSelector,

# Nix expression which takes version and package args, and returns an attrset to pass to mkDerivation
# Type: AttrSet -> AttrSet -> AttrSet
genericBuilder,
}:

# Some assertions as poor man's type checking
assert builtins.isFunction defaultSelector;

let
  variantsRaw = if builtins.isPath variants then import variants else variants;
  aliasesExpr = if builtins.isPath aliases then import aliases else aliases;
  genericExpr = if builtins.isPath genericBuilder then import genericBuilder else genericBuilder;

  aliases' = aliasesExpr { inherit lib; variants = variantsRaw; };
  variants' = if config.allowAliases then
      # Not sure if aliases or variants should have priority
      variantsRaw // aliases'
    else variantsRaw;

  # This also allows for additional attrs to be passed through besides version and src
  mkVersionArgs = { version, ... }@args: args // rec {
    # Some helpers commonly used to determine packaging behavior
    packageOlder = lib.versionOlder version;
    packageAtLeast = lib.versionAtLeast version;
    packageBetween = lower: higher: packageAtLeast lower && packageOlder higher;
    mkVersionPassthru = packageArgs: let
      variants = builtins.mapAttrs (_: v: mkPackage v packageArgs) variants';
    in variants // { inherit variants; };
  };

  # Re-call the generic builder with new version args, re-wrap with makeOverridable
  # to give it the same appearance as being called by callPackage
  mkPackage = version: lib.makeOverridable (genericExpr (mkVersionArgs version));
in
  # The partially applied function doesn't need to be called with makeOverridable
  # As callPackage will be wrapping this in makeOverridable as well
  genericExpr (mkVersionArgs (defaultSelector variants'))
```

## Openssl example

```diff
   # top-level.nix, pkgs/openssl/default.nix is called automatically

   # Aliasing from nixpkgs
-  openssl = openssl_3;
-  inherit (callPackages ./pkgs/openssl { })
-    openssl_1_1
-    openssl_3
-    openssl_3_2
-    openssl_3_3
-    ;
+  # Aliases for backwards compat
+  openssl_1_1 = openssl.v1_1;
+  openssl_3 = openssl.v3_0;
+  openssl_3_2 = openssl.v3_2;
+  openssl_3_3 = openssl.v3_3;
```

```nix
# pkgs/openssl/variants.nix
{
  v1_1 = {
    version = "1.1.1w";
    hash = "sha256-zzCYlQy02FOtlcCEHx+cbT3BAtzPys1SHZOSUgi3asg=";
  };
  ...
}
```

```nix
# pkgs/openssl/default.nix
{ callPackage
, mkPolyPkg
, ...
}@args:

# TODO: future proposal, make this less ugly
callPackage (mkPolyPkg {
  variants = ./variants.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}) args
```

New generic.nix:
```nix
# pkgs/openssl/generic.nix
{ version
, hash
, packageOlder
, packageAtLeast
, packageBetween
, mkVersionPassthru
, ...
}:

# Former packaging arguments:
{ lib
, stdenv
...
}@args:

stdenv.mkDerivation {
  ...
  passthru = (mkVersionPassthru args ) // {
    ... # existing args
  };
}
```

Usage of openssl from top-level:
```
nix-repl> openssl     
«derivation /nix/store/ayvv16slswqfps7c1zql8rml5nla6zky-openssl-3.3.1.drv»

nix-repl> openssl.v3_2 
«derivation /nix/store/77794w2h7xslv02k54xxc8x7krc1aazg-openssl-3.2.2.drv»

# Overriding still works
nix-repl> openssl.v3_2.override { withDocs = false; }
«derivation /nix/store/2196b3bj4vk2p6i7f3ai3p0n1m5im5iz-openssl-3.2.2.drv»

# To build all variants, need to pass `allowAliases = false;` to avoid eval error
nix-build -A openssl.variants
```

## Unresolved issues

- Argument passing is ugly
  - This is caused by us trying to satisfy mkPolyPkg and the package's arguments with the same arguments
  - This could be improved by importing default.nix with just mkPolyPkg, and passing the partially applied mkPolyPkg function to callPackage
    - E.g. `openssl = callPackage (import ./pkgs/openssl { inherit mkPolyPkg; }) { };` 
    - Similar to auto call-by-name, we could also create another directory which would just auto call generic packages, TBD in another proposal.

```nix
# pkgs/openssl/default.nix, with the result of this partially applied function
# being passed to callPackage
{ mkPolyPkg }:

mkPolyPkg {
  variants = ./variants.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}
```

- Should you be able to recurse variants? E.g. `openssl.v3_2.v3_1` (returns v3.1)
- Add `.overrideVersion`?
  - Unsure if having yet-another way to `.overrideX` is worth it
  - In most cases, this should be the same as doing `drv.overrideAttrs { version = ...; src = ...; }`
  - However, this would apply version information before the expression gets called with package arguments
    - This allows for logic which references version in many locations to work without use of mkDerivation fixed points. E.g. `(finalAttrs: { ... })`

## Future work

- Mitigate argument passing awkwardness with mkAutoCalledGenericDir proposal (TBD).
- Provide updater-script for use with generic packages

## Meta concerns

- This potentially shifts a lot of logic (version and deprecation aliases) from what was previously "package sets concerns" into a "package concern".
- Hard divergent behavior from upstream nixpkgs, many expressions will be less interchangable

