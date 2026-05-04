---
EEP: 0039
Title: Communicate ekapkgs-update policy through passthru
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-05-03
---

# Motivation

Package maintainers often need fine-grained control over how automated update tools handle their packages. Some packages require pinned versions for stability, others need to filter out pre-release versions, and some may have custom version detection requirements. Currently, there is no standardized way to communicate these preferences to the ekapkgs update tool.

This proposal introduces a `passthru.ekapkgs-update` attribute scope that allows package maintainers to declaratively specify update behavior directly in their package definitions. This approach:

- Provides a clear, self-documenting way to communicate update intent
- Reduces maintenance burden by eliminating the need for external configuration files
- Aligns with similar patterns in the Nix ecosystem (e.g., nix-update)
- Enables package-specific update policies that travel with the package definition

# Detailed Specification

The `passthru.ekapkgs-update` attribute set accepts the following options:

## `skip` (boolean)

When set to `true`, instructs ekapkgs to skip attempting to update this package entirely.

**Type:** `bool`
**Default:** `false`
**Use cases:**
- Packages with intentionally pinned versions
- Packages undergoing critical testing that shouldn't be updated automatically
- Deprecated packages that should remain frozen
- Packages which require a lot of bespoke fixes to make work

## `version-regex` (string)

A regular expression pattern that valid version strings must match. The update tool will only consider versions that match this pattern.

**Type:** `string` (POSIX extended regular expression)
**Default:** `null` (no filtering)
**Use cases:**
- Filtering out beta, rc, or alpha releases (e.g., `"^[0-9]+\\.[0-9]+\\.[0-9]+$"`)
- Restricting to specific version branches (e.g., `"^2\\..*"` for v2.x only)
- Excluding versions with specific suffixes or prefixes

## `version-type` (string)

Specifies the version detection strategy to use when checking for updates.

**Type:** `string`
**Default:** `null` (auto-detect)
**Supported values:**
- `"stable"` - Only consider stable releases (default)
- `"unstable"` - Include pre-release versions
- `"branch"` - Track a specific branch
- `"commit"` - Track specific commits. Questionable value, `skip` may be more appropriate
- Custom values as supported by the update tool implementation

**Use cases:**
- Packages that track unstable/development versions
- Packages that follow a specific branch rather than tags
- Packages with non-standard versioning schemes

## `branch` (string)

Specifies the branch to follow. Useful for maintenance branches. Only supported for some fetchers.

**Type:** `string`
**Default:** `null` (auto-detect)

**Use cases:**
- Maintenance branches which receive backports


# Example Usage


```nix
{

    # Skip updates for a pinned package
    passthru.ekapkgs-update.skip = true;

    # Filter out pre-release versions
    passthru.ekapkgs-update = {
      version-regex = "^[0-9]+\\.[0-9]+\\.[0-9]+$";
    };

    # Track only v2.x releases
    passthru.ekapkgs-update = {
      version-regex = "^2\\..*";
    };

    # Allow for prereleses. E.g. "0.1.0-alpha.5";
    passthru.ekapkgs-update = {
      version-type = "unstable";
    };

    # Pin to a version range, but allow unstable tags
    passthru.ekapkgs-update = {
      version-regex = "^1\\.[0-9]+\\.[0-9]+$";
      version-type = "stable";
    };

}
```

# Still questionable features

- **`commit-message-template` (string)** - Custom commit message format

# Future work

- Implementation into [ekapkgs-update](https://github.com/ekala-project/ekapkgs-update/)
