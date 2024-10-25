---
Title: Extend mkDerivation to provide development shell helper
Author: jonringer
Discussions-To: 
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-10-03
---

# Summary

Creating development shells from derivations which deviate from the standard
stdenv.mkDerivation usage have some mismatch between what `mkShell` (and variants)
will provide vs. what is intended. Allowing for any variation of mkDerivation 
to be converted will preserve details such as non-default compilers, build flags,
build phases, and environment variables.

## Detailed Implementation

Expose a toDevShell function which will allow for users to pass additional
derivation arguments which will be available in the dev shell. This will be
heavily inspired by the existing logic in `mkShell` which allows for easier
merging of inputs from other derivations.

Example PR: https://github.com/jonringer/stdenv/pull/4

```nix
  # stdenv/generic/make-derivation;
  toDevShell = let
    originalArgs = deleteFixedOutputRelatedAttrs derivationArg;
    toShell = import ./to-dev-shell.nix lib originalArgs;
    shellFunc = f: if builtins.isFunction f then f stdenv else f;
  in f: derivation (toShell (shellFunc f));
```

```nix
# stdenv/generic/to-dev-shell.nix
lib:

originalArgs:

# A special kind of derivation that is only meant to be consumed by the
# nix-shell.
{ name ? if originalArgs ? name then "${originalArgs.name}-dev-shell" else "nix-shell"
, # a list of packages to add to the shell environment
  packages ? [ ]
, # propagate all the inputs from the given derivations
  inputsFrom ? [ ]
, buildInputs ? [ ]
, nativeBuildInputs ? [ ]
, propagatedBuildInputs ? [ ]
, propagatedNativeBuildInputs ? [ ]
, ...
}@attrs:
let
  mergeInputs = name:
    (originalArgs.${name} or [ ]) ++
    (attrs.${name} or [ ]) ++
    # 1. get all `{build,nativeBuild,...}Inputs` from the elements of `inputsFrom`
    # 2. since that is a list of lists, `flatten` that into a regular list
    # 3. filter out of the result everything that's in `inputsFrom` itself
    # this leaves actual dependencies of the derivations in `inputsFrom`, but never the derivations themselves
    (lib.subtractLists inputsFrom (lib.flatten (lib.catAttrs name inputsFrom)));

  rest = builtins.removeAttrs attrs [
    "name"
    "packages"
    "inputsFrom"
    "buildInputs"
    "nativeBuildInputs"
    "propagatedBuildInputs"
    "propagatedNativeBuildInputs"
    "shellHook"
  ];
in originalArgs // {
  inherit name;

  buildInputs = mergeInputs "buildInputs";
  nativeBuildInputs = packages ++ (mergeInputs "nativeBuildInputs");
  propagatedBuildInputs = mergeInputs "propagatedBuildInputs";
  propagatedNativeBuildInputs = mergeInputs "propagatedNativeBuildInputs";

  # Avoid explicit checkout, and assume that shell will be used on source in repo
  src = null;

  shellHook = lib.concatStringsSep "\n" (lib.catAttrs "shellHook"
    (lib.reverseList inputsFrom ++ [ attrs ]));

  phases = [ "buildPhase" ];

  buildPhase = ''
    { echo "------------------------------------------------------------";
      echo " WARNING: the existence of this path is not guaranteed.";
      echo " It is an internal implementation detail for pkgs.mkShell.";
      echo "------------------------------------------------------------";
      echo;
      # Record all build inputs as runtime dependencies
      export;
    } >> "$out"
  '';

  preferLocalBuild = true;
} // rest
```

## Rust package example

Instead of creating a separate dev shell, one can largely reuse the packaging
logic from the nix expression.
```nix
# Before
mkShell {
  nativeBuildInputs = [
    cargo
    rustc
    clippy
  ];
  ...
}

# After, re-using the packaging logic from myPackage expression
myPackage.toDevShell {
  nativeBuildInputs = [ clippy ];
}

# Stdenv can be passed to inspect compatibility
myPackage.toDevShell (stdenv: {
  nativeBuildInputs = lib.optionals stdenv.buildPlatform.isLinux [
    gdb
  ] ++ [
    clippy
  ];

  buildInputs = lib.optionals stdenv.isDarwin [
    CoreServices
  ];
})
```

## Unresolved issues or concerns

- Adding a `.toDevShell` attr on every derivation does have a cost
  - The expense of this should be carefully analyzed before fully adopting

## Future work

N/A

