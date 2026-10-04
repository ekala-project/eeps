---
EEP: 0045
Title: Safe service migration across major version boundaries
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-10-04
---

# Motivation

Switching an EkaOS system configuration can silently upgrade a stateful service across a major version boundary (e.g., PostgreSQL 15 → 16). If the new binary's on-disk format is incompatible with the old data, the result ranges from a hard startup failure to silent data corruption. Today there is no framework-level protection: each service module must independently reinvent version detection, backup, and migration logic — and most don't.

NixOS's `stateVersion` partially addresses this by pinning default package versions, but it is a *prevention* mechanism, not a *migration* mechanism. When a user intentionally (or accidentally) crosses a boundary, nothing coordinates a safe transition.

The goal is a general-purpose, opt-in migration framework that:

1. Detects major-version boundary crossings automatically.
2. Defaults to **refusing** the switch rather than risking data loss.
3. Provides well-defined hooks for service authors to implement safe migrations with minimal boilerplate.
4. Requires no user configuration for the safe default; only the migration strategy requires intent.

# Prior Art

| System | How it detects version change | Migration responsibility | Default behavior |
|--------|------------------------------|--------------------------|-----------------|
| **Debian dpkg** | Passes old version as argument to `postinst configure <old-ver>` | Package maintainer writes version-conditional shell | Fail-safe: script failure aborts upgrade |
| **NixOS** | `stateVersion` pins defaults; `switch-to-configuration` diffs unit files | Module author + user (PostgreSQL is fully manual) | Fail-safe for defaults; no migration framework |
| **K8s Operators** | Reconciliation loop compares CRD spec vs running state | Operator author | Fail-safe: health checks gate progression |
| **Terraform** | State file diff (desired vs actual) | Cloud provider executes; user configures | Dangerous: may destroy+recreate without `allow_major_version_upgrade` |
| **Docker** | None | App entrypoint author | Fail-open: starts new binary against old data |
| **FreeBSD pkg** | No old version passed to scripts | Port maintainer; advisory UPDATING file | Fail-open |
| **Guix** | No per-service version tracking | User | Fail-open for data |
| **Ansible/Chef/Puppet** | Manual checks by playbook/cookbook author | Playbook author | No default |

**Key takeaway:** Debian's approach — passing the old version to maintainer scripts — is the most mature. However, it relies on dpkg tracking installed versions externally. In a Nix-based system the package manager doesn't track mutable service state, so we need a different mechanism to remember "what version last touched this data."

The proposed design draws most heavily from:

- **Debian's** version-argument pattern (the framework handles detection; the service author handles migration logic).
- **Kubernetes operators'** explicit health-gated upgrades (refuse by default; require intent).
- **Docker entrypoint pattern** of writing a state marker into the persistent data directory itself.

# Specification

## State Manifest

Each participating service writes a JSON manifest into its data directory at `/var/lib/<service>/.ekaos-state.json` during activation, **after** the service is confirmed running on the new version. The manifest records what version last successfully operated on the data:

```json
{
  "service": "postgresql",
  "majorVersion": "16",
  "packageVersion": "16.4",
  "storePath": "/nix/store/abc123-postgresql-16.4",
  "activatedAt": "2026-10-04T12:00:00Z"
}
```

The manifest is read by the **incoming** system's `preStart` to detect boundary crossings. This approach was chosen over inspecting `/run/current-system` because:

- It is **per-service** rather than system-wide — no need to parse an opaque system closure.
- It survives crashes: it reflects what *actually ran* against this data, not what the system profile says.
- It is fast: a single `jq` read vs. evaluating a Nix derivation.
- It works in containers and non-NixOS contexts where `/run/current-system` doesn't exist.
- After a rollback, `/run/current-system` may still point to the broken generation; the manifest is always accurate.

## Module Option

A new `migration` option is added to the service module contract. The framework generates the boundary-detection and migration orchestration logic; the service author only supplies the version key and any service-specific scripts.

```nix
services.<name>.migration = {
  enable = mkEnableDefault true;  # on by default for stateful services

  majorVersion = mkOption {
    type = types.str;
    description = "Value whose change triggers migration. Typically lib.versions.major.";
  };

  strategy = mkOption {
    type = types.enum [ "auto" "backup-and-start" "dump-restore" "refuse" "custom" ];
    default = "refuse";
    description = ''
      - refuse: block activation, log instructions (safe default)
      - auto: service handles its own migration internally
      - backup-and-start: framework runs preMigration, then starts new version
      - dump-restore: dump with old binary, init new data dir, restore with new binary
      - custom: run customMigrationScript
    '';
  };

  preMigration = mkOption {
    type = types.lines;
    default = "";
    description = "Runs before migration. Typically creates a backup.";
  };

  postMigration = mkOption {
    type = types.lines;
    default = "";
    description = "Runs after successful migration.";
  };

  customMigrationScript = mkOption {
    type = types.lines;
    default = "";
    description = "Full migration script (strategy = custom).";
  };

  backupDir = mkOption {
    type = types.str;
    default = "/var/backup/<service>";
  };
};
```

### Default: `refuse`

The default strategy is `refuse`. When a boundary crossing is detected, the `preStart` script exits non-zero with a clear message:

```
[postgresql] REFUSING automatic migration: major version 15 -> 16.
[postgresql] Data directory: /var/lib/postgresql
[postgresql] To proceed, set: services.postgresql.migration.strategy = "dump-restore";
[postgresql] Or migrate manually and rebuild.
```

This ensures no data loss by default. The user must express explicit intent before any migration runs.

## Generated preStart Logic

The framework wraps service-author-provided scripts with boundary detection:

```bash
STATE_FILE="${DATA_DIR}/.ekaos-state.json"
NEW_MAJOR="@majorVersion@"

if [ -f "$STATE_FILE" ]; then
  OLD_MAJOR=$(jq -r '.majorVersion' "$STATE_FILE")
  if [ "$OLD_MAJOR" != "$NEW_MAJOR" ]; then
    echo "[@service@] Migration boundary: $OLD_MAJOR -> $NEW_MAJOR"
    case "@strategy@" in
      refuse)
        echo "[@service@] REFUSING automatic migration."
        echo "[@service@] Set services.@service@.migration.strategy to proceed."
        exit 1 ;;
      backup-and-start)
        mkdir -p "@backupDir@"
        @preMigration@
        echo "[@service@] Backup complete, starting new version." ;;
      dump-restore)
        mkdir -p "@backupDir@"
        @preMigration@
        @dumpRestoreScript@
        @postMigration@ ;;
      auto)
        echo "[@service@] Service handles migration internally." ;;
      custom)
        mkdir -p "@backupDir@"
        @preMigration@
        @customMigrationScript@
        @postMigration@ ;;
    esac
  fi
else
  echo "[@service@] First activation, no prior state."
fi

# Write manifest after successful start
# (handled by ExecStartPost or activation postlude)
```

The manifest is written in `ExecStartPost` (not `preStart`) so that it only records success after the service actually starts. If migration succeeds but the service crashes on startup, the old manifest is preserved and the boundary will be re-detected on next attempt.

## Old Binary Availability

For `dump-restore`, the old binary is needed to export data. The manifest records `storePath`, which can be used directly:

```bash
OLD_STORE_PATH=$(jq -r '.storePath' "$STATE_FILE")
if [ -x "$OLD_STORE_PATH/bin/pg_dumpall" ]; then
  "$OLD_STORE_PATH/bin/pg_dumpall" -f "$BACKUP_DIR/dump.sql"
else
  echo "[@service@] Old binary garbage-collected. Manual migration required."
  exit 1
fi
```

To prevent GC during migration, the framework adds the old store path as a temporary GC root before migration begins and removes it after completion.

# Example Usage

## PostgreSQL (service author)

```nix
services.postgresql.migration = {
  majorVersion = lib.versions.major cfg.package.version;
  strategy = "dump-restore";
  preMigration = ''
    OLD="$(jq -r '.storePath' "$STATE_FILE")"
    "$OLD/bin/pg_dumpall" -U postgres -f "$BACKUP_DIR/pre-migration.sql"
  '';
};
```

## Elasticsearch (service author, auto-migrating)

```nix
services.elasticsearch.migration = {
  majorVersion = lib.versions.major cfg.package.version;
  strategy = "auto";  # ES handles rolling upgrades internally
};
```

## End user (no configuration needed)

By default, if a user bumps PostgreSQL from 15 to 16 without setting a strategy, the system switch fails with a clear message. No data is touched. The user then either:

1. Sets `migration.strategy = "dump-restore"` and rebuilds, or
2. Performs a manual migration and rebuilds (the service starts, writes a new manifest, and future switches work normally).

# Unresolved Questions

- **Scope of corepkgs changes:** The migration option type and `preStart` generation belong in corepkgs' service infrastructure. The exact integration point with `eval-config.nix` needs prototyping.
- **Multi-step upgrades:** Should the framework enforce that PostgreSQL 14 → 16 must go through 15, or is that the service author's responsibility? Leaning toward service-author responsibility since skip policies are service-specific.
- **Manifest location:** `.ekaos-state.json` inside the data directory is simple but relies on the service having a single canonical data directory. Services with multiple state directories may need multiple manifests or a central registry under `/var/lib/ekaos/migrations/`.

# Future Work

- **Automatic backup integration:** A `backup` service contract that migration hooks can invoke, abstracting the backup medium (local snapshot, restic, etc.).
- **Migration dry-run:** `ekaos-rebuild --check-migrations` that reports which services would cross boundaries without activating.
- **Rollback integration:** If a migration fails, automatically restore from the pre-migration backup and leave the service on the old version.
- **CI validation:** Ensure that service modules with stateful data directories always define a `migration.majorVersion`, enforced by a linter.
