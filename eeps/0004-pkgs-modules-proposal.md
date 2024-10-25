---
Title: Allow for users to pass modules to pkgs.config
Author: jonringer
Discussions-To: https://github.com/jonringer/language-specific-config-overlays-proposal
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-10-17
---

# Summary

In addition to the [language overlay proposal](https://github.com/jonringer/language-specific-config-overlays-proposal),
the ability to configure the language overlays should also not be burdensome to compose.
The solution here would be to extend the existing `pkgs.config` evaluation to include
many modules which can be passed to its creation. The goal is to allow for both
the overlays (e.g. `core-pkgs`) to pass modules describing their overlays in addition
to the overlays supplied by the user.

## Detailed Implementation

The totality of https://github.com/jonringer/language-specific-config-overlays-proposal:

```diff
@@ -27,6 +27,9 @@
 , # Allow a configuration attribute set to be passed in as an argument.
   config ? {}

+, # Allow for users to pass modules for the evaluation of pkgs.config
+  modules ? []
+
 , # List of overlays layers used to extend Nixpkgs.
   overlays ? []

@@ -93,7 +96,7 @@ in let
         _file = "nixpkgs.config";
         config = config1;
       })
-    ];
+    ] ++ modules;
     class = "nixpkgsConfig";
   };
```

## Poly repo example

For the "capstone" pkgs repo, the composition of all sub-overlays would roughtly follow:


```nix
# default.nix in pkgs repo

{ system ? builtins.currentSystem
, modules ? []
, ...
}@args:

let
  # pins to language repos and core-pkgs
  pins = import ./pins.nix;

  # stdenv should rarely change, as pkgs.stdenv is largely created in core-pkgs
  stdenv = ...;

  filteredAttrs = builtins.remoteAttrs [ "modules" "system" ] args;
in
import stdenv ({
  inherit system;
  modules = let
    poly-modules = builtins.mapAttrs (_: v: import "${v}/pkgs-module.nix") pins;
  in poly-modules ++ modules;
} // filteredAttrs)
```

Each poly-repo would just expose a `/pkgs-module.nix` which would contain all information
about the language specific repos to the package set.

## User package example

A general flake exposing a python package could be expressed as:

```nix
# flake.nix

let
  pythonOverlay = final: _: {
    my-pkg = final.callPackage ./package.nix { };
  };

  top-level-overlay = final: _: {
    my-pkg = with final.python3.pkgs; toPythonApplication my-package;
  };

  pkgs-module = {
    overlays.pkgs = [ top-level-overlay ];
    overlays.python = [ pythonOverlay ];
  };
in flake-utils.eachDefaultSystem (system: rec {

  legacyPackages = import core-pkgs {
    inherit system;
    modules = [ pkgs-module ];
  };

  packages.default = legacyPackages.my-pkg;

  modules = {
    pkgs = pkgs-module;
  };
})
```

The benefit here is that the package is now available for each `pkgs.pythonX` version
as well as the module is available for downstream flakes/repos to also consume in an easy fashion.
For example:

```nix
# flake.nix
inputs.foo = "github:....";

...
{
  legacyPackages = import core-pkgs {
    inherit system;
    # This would now apply the python and top-level overlay from foo as well
    modules = [ foo.modules.pkgs my-repo-module ];
  };
}
```

## Unresolved questions

- To remove ambigiuity with system module eval, should the attr be named `pkgsModules` instead?
- What to do with legacy `config` option?
  - Most likely keep for backwards compatibility with people wanting to interop with nixpkgs

## Future work

- Implement `options.overlays.<language` for each language repository
- As part of system mdoule eval, a `nixpkgs.modules` option should be added 
  - `nixpkgs` should be renamed to a more generic term


