---
Title: Auto call manyVariant packages to separate variant and dependency concerns
Author: jonringer
Discussions-To: https://github.com/jonringer/multiple-package-versions-proposal?
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-9-22
---

# Summary

Argument passing in multiple-package-versions-proposal was awkward, this creates
a mkAutoCalledPolyPkgDir function which creates an overlay in which the directories
in a specified directory will be automatically called with mkPolyPkg, and that
partially applied function then passed to callPackage.

This separates the creation of a package which contains multiple variants and
the callPackage call which should only be passing arguments needed to conduct
a build.

## Detailed Implementation

Example PR: https://github.com/jonringer/core-pkgs/pull/6

```nix
let
  mkAutoCalledPolyPackageDir = baseDirectory:
  let
    namesForShard = lib.packageSets.mkNamesForDirectory baseDirectory;
    # This is defined up here in order to allow reuse of the value (it's kind of expensive to compute)
    # if the overlay has to be applied multiple times
    packageFiles = mergeAttrsList (mapAttrsToList namesForShard (builtins.readDir baseDirectory));
  in
  # TODO: Consider optimising this using `builtins.deepSeq packageFiles`,
  # which could free up the above thunks and reduce GC times.
  # Currently this would be hard to measure until we have more packages
  # and ideally https://github.com/NixOS/nix/pull/8895
  self: _:
    {
      _internalCallPolyFile = file: self.callPackage (import file { inherit (self) mkPolyPkg; }) { };
    }
    // builtins.mapAttrs
      (name: value: self._internalCallPolyFile value)
      packageFiles;

  polyPkgOverlay = mkAutoCalledPolyPackageDir ./polyPkgs;
in

import stdenvRepo {
  overlays = [
    polyPkgOverlay
    ...
  ];
}

```

## Openssl example

Previously, arguments for mkPolyPkg needed to be passed along with the arguments
for the underlying package, this now separates the two concerns.

```nix
# before changes
{ callPackage
, mkPolyPkg
, ...
}@args:

callPackage (mkPolyPkg {
  versions = ./versions.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}) args
```

```nix
# post changes
{ mkPolyPkg }:

mkPolyPkg {
  versions = ./versions.nix;
  aliases = ./aliases.nix;
  defaultSelector = (p: p.v3_3);
  genericBuilder = ./generic.nix;
}
```

