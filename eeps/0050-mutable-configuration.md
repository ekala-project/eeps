---
EEP: 0050
Title: Mutable configuration files
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-10-12
---

# Motivation

Configuration management tools face a fundamental tension: the system wants to enforce a declared state, but users and administrators want to edit files freely. NixOS and ekaos currently deploy all managed files as read-only symlinks into the nix store. This forces users to nixify every configuration file, creating an adoption barrier for users who want sensible defaults they can hand-edit.

Four major tools — systemd-tmpfiles (`f`/`C`), Puppet (`replace => false`), Ansible (`force: no`), and chezmoi (`create_`) — converge on a "create-only copy" pattern: deploy defaults once, never overwrite. This EEP adopts the same pattern for both `home.users.<name>.file` and `environment.etc`.

# Specification

## New options

Two boolean options are added to both `home.users.<name>.file.<path>` and `environment.etc.<name>`:

| Option | Default | Behavior |
|--------|---------|----------|
| `mutable` | `false` | When `true`, deploy as a writable copy instead of a read-only symlink. The file is created only if it does not already exist. |
| `forceUpdate` | `false` | When `true` (requires `mutable = true`), overwrite the file even if it exists. The previous version is backed up as `*.ekaos-backup`. |

## Deployment semantics

| `mutable` | File exists | Action |
|-----------|-------------|--------|
| `false` | — | Symlink to nix store (current behavior) |
| `true` | No | Copy reference content into target |
| `true` | Yes (regular file) | Skip (preserve user edits) |
| `true` | Yes (symlink from previous immutable) | Replace symlink with writable copy |

Mutable files are never removed during deactivation or generation rollback.

## Manifest format

The home-files manifest changes from a flat JSON array of paths to an array of objects:

```json
[
  {"target": ".bashrc", "mutable": false},
  {"target": ".config/git/config", "mutable": true}
]
```

The activation script parses both formats for backward compatibility.

## Program module convenience

Each home program module (`programs.bash`, `programs.git`, `programs.ssh`, `programs.gpg-agent`) gains a `mutable` option that propagates to all files it generates. Mutable program configs use the header "Feel free to edit." instead of "Do not edit."

## System-level (`environment.etc`)

The same `mutable` and `forceUpdate` options apply to `environment.etc.*`. Immutable files (default) are overwritten from the nix store on every activation, matching NixOS behavior. Mutable files use create-only semantics. Separate build outputs track each category:

- `system.build.etc` — immutable files
- `system.build.etcMutable` — mutable reference copies
- `system.build.etcManifest` — JSON manifest
- `system.build.etcForceUpdateTargets` — force-overwrite list

# Examples

Per-file mutable config:

```nix
home.users.alice.file.".config/myapp/config.toml" = {
  text = ''
    theme = "dark"
    font_size = 14
  '';
  mutable = true;
};
```

Per-program mutable config:

```nix
home.users.alice.programs.git = {
  enable = true;
  mutable = true;
  settings.user.name = "Alice";
};
```

System-level mutable /etc file:

```nix
environment.etc."myapp/config.toml" = {
  text = ''
    key = "value"
  '';
  mutable = true;
};
```

# Prior art

| Tool | Mechanism | Behavior |
|------|-----------|----------|
| systemd-tmpfiles | `f` vs `f+` | `f` creates if absent; `f+` always overwrites |
| Puppet | `replace => false` | Creates file if absent, never overwrites content |
| Ansible | `copy` with `force: no` | Transfers only if destination does not exist |
| chezmoi | `create_` prefix | Creates target only if it does not already exist |
| home-manager | `mutable = true` | Overwrites on every activation (different semantics) |

This EEP deliberately chooses create-only semantics over home-manager's overwrite-copy approach, since the goal is preserving user edits.

# Implementation

Reference implementation: https://github.com/ekala-project/corepkgs (branch `jonringer/mutable-home-configuration`)

Files changed:

- `ekaos/modules/home/options.nix` — `mutable`, `forceUpdate` on `homeFileOpts`
- `ekaos/modules/home/build.nix` — split `mkHomeFiles` into immutable/mutable trees, updated manifest and activation script
- `ekaos/modules/home/bridge.nix` — matching `homeFileOpts` extension
- `ekaos/modules/home/programs/{bash,git,ssh,gpg-agent}.nix` — per-program `mutable` option
- `ekaos/modules/system/etc.nix` — `mutable`, `forceUpdate` on `environment.etc`
- `ekaos/modules/system/activation.nix` — three-phase deployment (immutable, mutable, force-update)
- `ekaos/modules/config/home.nix` — legacy module parity
