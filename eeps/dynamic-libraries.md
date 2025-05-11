---
Title: Dynamic Libraries
Author: interacsion
Status: Draft
Type: Informational
Created: 2025-05-11
---

# Preface

*Dynamic Libraries* (also referred to as *Shared Libraries*, *Shared Objects* or just *Libraries*) are files containing executable code (*Executables*) that can be loaded into a process at runtime. This is generally done in one of 2 ways:

### 1. *Dynamic Linking*

An *Executable* (either a *Binary*, which contains an entry point, or a *Library*) can be compiled against a Dynamic Library. When such Binary is run it will first invoke the *Dynamic Linker* (also referred to as *Dynamic Loader* or *ELF Interpreter*) which will in turn recursively load all required Dynamic Libraries into process memory.

### 2. *Dynamic Loading*

A Dynamic Library can also be loaded into process memory by calling `dlopen` (or `dlmopen`, more on that later). The Executable does not need to be compiled against the Library, and can conditionally load different Libraries.

## Dynamic Libraries in Nix

On traditional Linux distributions, Dynamic Libraries are usually linked to by their *`SONAME`* (The filename if the Shared Library). The Dynamic Linker then searches a predefined set of system directories to find it.

Nix, on the other hand, takes a different approach; since it doesn't install packages into standard directories, it links Dynamic Libraries by their full filepath, and this is where the problem starts.

# Problem Statement

Most Dynamic Libraries assume that the Library and the Executable that uses it have access to the same instance of a Dynamic Library they both depend on.

The most obvious example is a Library which allocates memory and expects the User to free it. This requires that both the Library and the User share the same instance of, usually, glibc.

The way the Dynamic linker accommodates this is as following: Before a Library is loaded it checks if a Library with the same `SONAME` is already loaded into the process, disregarding the full filepath, if specified. if so, it substitutes it for the one that is already loaded.

This works fine on traditional Linux distributions, but **actively undermines** the purity guarantees Nix tries to make. For example, if a Binary is linked to a certain version of glibc and to a Library that's linked to a different version of glibc, the first one that gets loaded will trump the other.

Nixpkgs manages to mostly avoid this problem due to its monorepo nature; there's a single attribute set containing all the packages and as long as a Binary only links to dependencies from a single Nixpkgs revision, no conflict can occur.

Ekapkgs, on the other hand, would have to manage this more carefully due to its polyrepo structure. For example, a binary from `pkgs` could depend on one `corepkgs` revision and on a library from `ocaml-pkgs` which depends on a different `corepkgs` revision, which would attempt to load two different glibc versions.

## System Libraries

A small subset of Dynamic Libraries must be installed as part of the system; most notable are graphics drivers (which are loaded using  `libvulkan`, `libglvnd` or `libgbm`), but GPGPU and sound systems probably fall into this category as well.

Since these Libraries are installed separately than the applications using them, they cannot be expected to link to the same version of glibc as the application. This is a problem that actually manifests in NixOS (Citation Needed).

# Non-Solutions

### 1. Patching the Dynamic Linker to load multiple versions of the same Library

This is not a solution since *Most Dynamic Libraries assume that the Library and the Executable that uses it have access to the same instance of a Dynamic Library they both depend on.*
