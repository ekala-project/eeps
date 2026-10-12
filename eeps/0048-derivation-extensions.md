---
EEP: 0048
Title: Derivation Extensions — compile-time feature suffixes in mkDerivation
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-10-12
---

# Motivation

Packages with optional compile-time features (JDBC drivers, GTK support, minimal builds, GPU backends, etc.) need a way to reflect enabled features in the derivation name. Today this is done ad-hoc: each package manually constructs a pname suffix with `lib.optionalString` or string interpolation:

```nix
pname = "libcanberra" + lib.optionalString (gtkSupport != null) "-${gtkSupport}";
pname = "qbittorrent" + lib.optionalString (!guiSupport) "-nox";
pname = if withDPDK then "openvswitch-dpdk" else "openvswitch";
```

This leads to inconsistent naming, duplicated boilerplate, and — critically — clobbers `pname`. Downstream tools and queries that match on `pname` must account for every possible suffix a package might append. The real package name gets lost in variant information.

A first-class `extensions` attribute in `mkDerivation` separates the package identity (`pname`) from variant information (`extensions`). The build system appends enabled extensions to the derivation name automatically, producing predictable names like `duckdb-jdbc-1.4.3` or `libcanberra-gtk3-0.30` while keeping `pname` stable as `duckdb` and `libcanberra` respectively.

# Specification

## The `extensions` attribute

`mkDerivation` accepts an optional `extensions` attribute: a list of strings naming compile-time features enabled in this build.

```nix
stdenv.mkDerivation {
  pname = "duckdb";
  version = "1.4.3";

  extensions =
    lib.optionals withJdbc [ "jdbc" ]
    ++ lib.optionals withOdbc [ "odbc" ];

  # ...
}
```

## Name construction

When `extensions` is set and non-empty, `mkDerivation` inserts the extensions into the derivation name between `pname` and the version:

```
${pname}${extensionsSuffix}${staticMarker}${hostSuffix}-${version}
```

where `extensionsSuffix` is each extension prefixed with `-`, concatenated in list order:

```nix
extensionsSuffix = concatMapStrings (ext: "-${ext}") (attrs.extensions or [ ]);
```

| `pname` | `extensions` | `version` | Derivation name |
|---------|-------------|-----------|-----------------|
| `duckdb` | `[]` | `1.4.3` | `duckdb-1.4.3` |
| `duckdb` | `["jdbc"]` | `1.4.3` | `duckdb-jdbc-1.4.3` |
| `duckdb` | `["jdbc" "odbc"]` | `1.4.3` | `duckdb-jdbc-odbc-1.4.3` |
| `libcanberra` | `["gtk3"]` | `0.30` | `libcanberra-gtk3-0.30` |
| `qbittorrent` | `["nox"]` | `5.0.4` | `qbittorrent-nox-5.0.4` |

## Attribute stripping

`extensions` is added to `removedOrReplacedAttrNames` so it does not leak into the derivation environment. It is consumed only during name construction.

## Interaction with `name`

When the caller sets `name` directly instead of `pname`/`version`, `extensions` has no effect. This matches the existing behavior of `staticMarker` and `hostSuffix`.

# Conventions

- Extension names SHOULD be lowercase, short, and hyphen-free (e.g., `jdbc`, `odbc`, `cuda`, `gtk3`, `nox`, `minimal`).
- Extension order SHOULD be alphabetical unless a different order is more natural for the domain.
- Extensions are for compile-time feature gates that produce a meaningfully different build. Runtime plugins, optional dependencies that don't change the build output, or feature flags that don't warrant a distinct store path should not use this mechanism.

# Examples

## Database driver variants

```nix
# duckdb generic.nix
extensions =
  lib.optionals withJdbc [ "jdbc" ]
  ++ lib.optionals withOdbc [ "odbc" ];
```

Produces: `duckdb-1.4.3`, `duckdb-jdbc-1.4.3`, `duckdb-odbc-1.4.3`, `duckdb-jdbc-odbc-1.4.3`.

## GTK support

```nix
# libcanberra generic.nix
extensions = lib.optionals (gtkSupport != null) [ gtkSupport ];
```

Produces: `libcanberra-0.30` (base), `libcanberra-gtk3-0.30`.

## Minimal / headless builds

```nix
# qbittorrent
extensions = lib.optionals (!guiSupport) [ "nox" ];
```

Produces: `qbittorrent-5.0.4` (full), `qbittorrent-nox-5.0.4` (headless).

# Implementation

The change is a single addition to `stdenv/generic/make-derivation.nix` in corepkgs:

1. Add `"extensions"` to `removedOrReplacedAttrNames`.
2. Compute `extensionsSuffix` from `attrs.extensions or [ ]` in the name construction block.
3. Insert `extensionsSuffix` after `pname` in the name template.

No existing packages are affected since `extensions` defaults to `[]`.
