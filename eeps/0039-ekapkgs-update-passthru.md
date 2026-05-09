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

## `semver-strategy` (string)

Controls which version updates are acceptable based on semantic versioning constraints.

**Type:** `string`
**Default:** `"latest"` (accept any newer non-prerelease version)
**Supported values:**
- `"latest"` - Accept any newer non-prerelease version (default)
- `"major"` - Same as "latest" - allow major version updates
- `"minor"` - Only update to latest minor version within the same major version (e.g., 1.x.x)
- `"patch"` - Only update to latest patch version within the same major.minor version (e.g., 1.2.x)

**Use cases:**
- Conservative updates: Use `"patch"` to minimize breaking changes
- Moderate updates: Use `"minor"` to get new features without major version jumps
- Packages with non-standard versioning schemes

**Note:** Maps to the existing `SemverStrategy` enum in ekapkgs-update.

## `include-prereleases` (boolean)

When set to `true`, allows the update tool to consider pre-release versions (alpha, beta, rc, etc.) as valid update candidates.

**Type:** `bool`
**Default:** `false` (exclude prereleases)

**Use cases:**
- Packages that track unstable/development versions
- Tracking development versions (e.g., Rust nightly, beta channels)
- Early adopters who want to test upcoming versions


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

    # Allow for prereleases. E.g. "0.1.0-alpha.5";
    passthru.ekapkgs-update = {
      include-prereleases = true;
    };

    # Only accept patch updates within current version
    passthru.ekapkgs-update = {
      semver-strategy = "patch";
    };

    # Track minor versions, including prereleases
    passthru.ekapkgs-update = {
      semver-strategy = "minor";
      include-prereleases = true;
    };

}
```

# Future work

## Branch Tracking

Tracking specific Git branches (e.g., `stable`, `lts-v2`) is deferred to future work as it requires a fundamentally different update mechanism than release/tag-based updates. Proposed attribute:

```nix
passthru.ekapkgs-update = {
  follow-branch = "stable";  # Track specific branch instead of releases
};
```

**Challenges:**
- Branches don't have inherent version numbers
- Requires different API calls (branches endpoint vs releases/tags)
- Version comparison would need to use commit metadata instead of semver
