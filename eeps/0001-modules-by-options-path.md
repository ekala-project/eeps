---
EEP: TBD
Title: Organize module file paths by option path
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2025-05-25
---

# Terminology

- Option Module
  - A NixOS module which exposes `options.<option path>` to the module evaluation
- Profile Module
  - A NixOS module which includes opinionated configuration. For Ekapkgs, this will
    be used to for distributing a base installable images (e.g. minimal ISO).

# Motivation

Similar to (package expression EEP), modules should also be organized in a similar
manner. This should make finding the related module and option significantly more
intuitive. Similarly, since modules are distributed amongst many repositories now,
there's less of a need to have deeper directories.

# Detailed Specification

When adding an option, the file path for the module should reflect the option
attr path. For example, adding `services.openssh`, the related module should be
located at `/modules/services/openssh/default.nix`.

In general, the formula can be described as `lib.concatStringSep "/" <attr path>`,
where `<attr path>` is a `listOf str` similar to what `lib.attrByPath` expects.

Similarly, "profile modules" should be a `/profiles` directory, the specifics
of which will not be addressed here. For example, the minimal iso module could
have a valid path of `/profiles/installable/iso/minimal.nix`. The specifics
of opinionated configuration needs to be addressed in a further EEP.

# Example usage

- Postgresql, `services.postgresql`:
  - In nixpkgs: `/nixos/modules/services/databases/postgresql.nix`
  - In corepkgs: `/modules/services/postgresql/default.nix`

# Unresolved questions

- File structure for `/profiles` directory
- Normalize attr paths for Ekapkgs?

# Future work

None, will be done when porting NixOS modules to corepkgs and ekapkgs.

Will be enforced when new modules are added.

