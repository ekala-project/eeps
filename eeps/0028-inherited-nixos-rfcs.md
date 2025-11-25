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
- [0035 - Default name from pname](https://github.com/NixOS/rfcs/pull/35):
- [0037 - add x86_32](https://github.com/NixOS/rfcs/pull/37)
  - This later became x86-i686Linux in [RFC0038](https://github.com/NixOS/rfcs/pull/38), but becoming less relevant each year
- [0042 - NixOS settings options](https://github.com/NixOS/rfcs/pull/37)
  - Absolute ergonomic win

Rejected NixOS RFCs, but interesting ideas:
- [0012 - Declarative virutal machines](https://github.com/NixOS/rfcs/pull/12):
  - No agreeance on what implementation to do. But would be interesting to revisit
- [0013 - Ergonomic cmakeFlags](https://github.com/NixOS/rfcs/pull/13):
  - Now with `__structuredAttrs`, this might be more feasible.
  - Replace `cmakeFlags` and `cmakeFlagsArray` with `cmakeAttrs` which is an attrset
- [0024 - Python scope package set](https://github.com/NixOS/rfcs/pull/24):
  - How large language packge sets should be revisited
- [0027 - Trusted bots](https://github.com/NixOS/rfcs/pull/27):
  - Automation is good, stop fighting it
- [0034 - Expression Integrity](https://github.com/NixOS/rfcs/pull/34)
  - Some way to approve/sign nix expressions
  - Currently code just relies on author and committer acting in good faith
- [0039 - Unprivileged maintainer team](https://github.com/NixOS/rfcs/pull/34)
  - Would be nice to have finer granularity of merge abilities
  - Mitigated by EkaCI and tooling

Rejected NixOS RFCs, but superceded by Ekapkgs:
- [0022 - Minimal module list](https://github.com/NixOS/rfcs/pull/22)
  - Corepkgs will expose a minimal module set for creating a system
  - Ekapkgs will have the option to do a more "feature rich" module evaluation
- [0029 - Backports team](https://github.com/NixOS/rfcs/pull/29)
  - Most backport decision making should be deterministic, any committer should be able to do this
  - Should be revisited to see if tooling can't be improved
- [0030 - Formalize review workflow](https://github.com/NixOS/rfcs/pull/29)
  - Purpose of EkaCI was to make review significantly easier for maintainers
  - Lower barrier to EEPs means less churn of leveraging large PRs as mini-RFCs
- [0033 - Deprecation](https://github.com/NixOS/rfcs/pull/33)
  - `mkManyVariants` will include a "deprecated variants" projection, but builds will always be available, just not in cache or tested in CI
- [0036 - Improving the RFC process](https://github.com/NixOS/rfcs/pull/36):
  - Formalized the NixOS RFC process and Steering Committee
  - To be replaced by EEPs and related Steering Committee (which will avoid the need for shepards)
    - Current NixOS RFC process takes many months and the barriers to landing anything makes contributions unlikely

Both Rejected:
- [0003 - Simple Override Strategy](https://github.com/NixOS/rfcs/pull/3):
  - Proposes to remove `.override` and `.overrideDerivation` and replace with deep `//`
  - Rejection reasoning: `//` is very limited to reducing invariance, and `.overrideX` is much more ergonomic
- [0010 - Nixpkgs development Support](https://github.com/NixOS/rfcs/pull/10)
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
- [0025 - Nix Core Team](https://github.com/NixOS/rfcs/pull/25)
  - NixOS/nix team, nothing to do with packaging
- [0028 - Nix Release Model](https://github.com/NixOS/rfcs/pull/28)
- [0040 - "Ret-cont" recursive Nix](https://github.com/NixOS/rfcs/pull/40)
- [0041 - SELinux Support](https://github.com/NixOS/rfcs/pull/41)
  - Intesting, but requires Nix cli/daemon/store changes

# Changes


