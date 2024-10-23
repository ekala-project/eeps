---
Title: Align file paths to the exposed attrs
Author: jonringer
Discussions-To: 
Status: Draft
Type: Standards Track
Repo: All poly repos
Topic: Packaging
Created: 2024-10-06
---

# Summary

Nixpkgs contains a lot of legacy "oddities" which cause some impediance mismatch
when currating a large package set. One of these "oddities" is that is fairly
common for packages to not align the expected file path. This proposal wants
to enforce that the "way a package is consumed should reflect where it is placed.

Normalizing names and file paths align nicely with [auto-called-directories](https://github.com/jonringer/nix-lib/commit/86961f6f0139a70ee29a74237dd59a9936299ed0)
which eliminates the need for an explicit attr to reference the file or directory.

## Detailed Implementation

There's four scenarios which in particular which should be addressed:
- Attrs referencing a wrapper or specific version. E.g. `gobjection-introspection` points to a wrapper.nix
  - Instead, the default attr should point to `default.nix` and the `-unwrapped` variant should point to an `unwrapped.nix`
- File locations should be consolidated in a way to reflect the package scope in which they are contained
  - E.g. `attr: pkgs.expat` -> `file: pkgs/expat/default.nix`
  - E.g. `attr: python.pkgs.pip` -> `file: python/pkgs/pip/default.nix`
- Directory name should directly correspond with attr
  - E.g. `gtk/3.x.nix` should be `gtk3/default.nix`
- Passing overrides to `callPackage` should be discouraged, instead the specific attr should be referenced by the nix expression
  - E.g. Listing `ffmpeg` in the nix expression, but passing `ffmpeg = ffmpeg_7;` in the overrides, it should just be `ffmpeg_7` in the nix expression

## Adjacent efforts

- [PolyPkg proposal](https://github.com/jonringer/multiple-package-versions-proposal) mitigates the "which variant do I need?"/overrides issue.
  - E.g. `ffmpeg` can be passed to the nix expression, but `ffmpeg.v7` can be passed as an input, which eliminates the need to pollute a package scope with many varaints

## Future work

- Do the work

## Meta concerns

- Collapsing the folders could result in absurdly large directories
  - This is somewhat mitigated by the poly-repo structure. There should only be a few thousand packages in any one directory in a given repo.

