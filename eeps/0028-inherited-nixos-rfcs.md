---
EEP: TBD
Title: Accepted NixOS RFCs
Author: jonringer
Status: Draft
Type: Standards Track
Created: 2025-05-25
---

# Motivation

Ekapkgs still aligns with the goals of some NixOS RFCs. There should be an exlicit
list of which RFCs are going to be adopted, revoked, or superceded by an EEP.

# Detailed Specification

Each accepted NixOS RFC should have a related EEP issue opened up to determine
it's viability.

Both Accepted:
- [0015 - Release managers](https://github.com/NixOS/rfcs/pull/15):
  - Primary+secondary RM process is good for getting more investment and contributions

Rejected NixOS RFCs, but interesting ideas:
- [0012 - Declarative virutal machines](https://github.com/NixOS/rfcs/pull/12):
  - No agreeance on what implementation to do. But would be interesting to revisit
- [0013 - Ergonomic cmakeFlags](https://github.com/NixOS/rfcs/pull/13):
  - Now with `__structuredAttrs`, this might be more feasible.
  - Replace `cmakeFlags` and `cmakeFlagsArray` with `cmakeAttrs` which is an attrset
- [0024 - Python scope package set](https://github.com/NixOS/rfcs/pull/24):
  - How large language packge sets should be revisited

Rejected NixOS RFCs, but superceded by Ekapkgs:
- [0022 - Minimal module list](https://github.com/NixOS/rfcs/pull/22)
  - Corepkgs will expose a minimal module set for creating a system
  - Ekapkgs will have the option to do a more "feature rich" module evaluation

Both Rejected:
- [0003 - Simple Override Strategy](https://github.com/NixOS/rfcs/pull/3):
  - Proposes to remove `.override` and `.overrideDerivation` and replace with deep `//`
  - Rejection reasoning: `//` is very limited to reducing invariance, and `.overrideX` is much more ergonomic
- [0010 - Nixpkgs development Support](https://github.com/NixOS/rfcs/pull/10)
  - It's a package repository, not a dumping ground for helper functions
- [0012 - Nixpkgs development Support](https://github.com/NixOS/rfcs/pull/10)
  - It's a package repository, not a dumping ground for helper functions
- [0019 - Maintainers file](https://github.com/NixOS/rfcs/pull/19)
  - Tries to do a CODEOWNERS like "maintainership over paths"
  - A committer should just have domain over the entire repo (ekapkgs being poly-repo mitigates this desire)
- [0020 - Security On Call](https://github.com/NixOS/rfcs/pull/20)
  - "On call" to respond to events
  - No, just have a healthy collection of core maintainers which can do minor security pushes

Ignored RFCs (Nix-cli related):
- [0004 - Replace Unicode Quotes](https://github.com/NixOS/rfcs/pull/3):
  - Mostly related to Nix
- [0005 - Nix encryption](https://github.com/NixOS/rfcs/pull/5):
  - Mostly related to new Nix builtins
  - No resolution, closed for being draft
- [0006 - Runtime references interface](https://github.com/NixOS/rfcs/pull/6):
  - Allow for runtime references to be configured as part of the build
  - Requires changes to Nix cli to alter retained dependencies behavior
- [0008 - Readonly recursive Nix](https://github.com/NixOS/rfcs/pull/8)
- [0009 - Nix rapid release](https://github.com/NixOS/rfcs/pull/9)
  - Relevant to nixos/nix release cadence
- [0011 - Per project Config](https://github.com/NixOS/rfcs/pull/11)
  - Allow for nix.conf to be extended by a repository
- [0014 - Improve import from derivation](https://github.com/NixOS/rfcs/pull/14)
  - Mainly a Nix concern, but interesting

# Changes


