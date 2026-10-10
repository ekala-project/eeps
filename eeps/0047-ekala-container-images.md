---
EEP: 0047
Title: Ekala Container — a Nix-native container image format
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Requires: 0041
Created: 2026-10-09
---

# Motivation

Building container images from Nix today requires translating Nix store closures into OCI tar layers, then stacking them through overlayfs at runtime. This translation is lossy and architecturally unnecessary:

- **Layer limits.** Docker caps at 125 layers; OCI backends at 256. A typical service closure contains hundreds of store paths, forcing tools like `dockerTools.buildLayeredImage` and `nix2container` to merge paths via popularity heuristics — destroying the 1:1 store-path-to-layer mapping and reducing cache hit rates.
- **Redundant data transformation.** Store paths must be tar-packed, compressed, and hashed even though the Nix store already holds this content in a content-addressed, deduplicated form.
- **Overlayfs overhead.** The overlay filesystem adds kernel overhead and has depth limits. For Nix containers — where the entire rootfs is read-only and derived from immutable store paths — the mechanism is unnecessary.
- **Accidental deduplication.** Two OCI images sharing the same `glibc` only share a layer if the same tool with the same heuristics assigned it to the same layer. In contrast, Nix store paths are globally unique — sharing is guaranteed by construction.

The core insight (articulated in [NixOS/nixpkgs#374288](https://github.com/NixOS/nixpkgs/issues/374288)): store paths do not require overlay semantics.

# Specification

## Ekala Container Manifest

An Ekala Container is defined by a JSON manifest with media type `application/vnd.ekala.container.manifest.v1+json`. The manifest declares the closure, filesystem layout, process configuration, and resource declarations (volumes, ports, health checks) needed by orchestrators.

```json
{
  "schemaVersion": 1,
  "mediaType": "application/vnd.ekala.container.manifest.v1+json",

  "architecture": "amd64",
  "os": "linux",
  "variant": "",

  "closure": [
    {
      "storePath": "/nix/store/abc123-glibc-2.39",
      "narHash": "sha256:...",
      "narSize": 31457280,
      "references": []
    },
    {
      "storePath": "/nix/store/def456-myapp-1.0",
      "narHash": "sha256:...",
      "narSize": 524288,
      "references": ["/nix/store/abc123-glibc-2.39"]
    }
  ],

  "rootfs": {
    "storeMount": "/nix/store",
    "symlinks": {
      "/bin/myapp": "/nix/store/def456-myapp-1.0/bin/myapp",
      "/etc/ssl/certs": "/nix/store/abc123-ca-bundle/etc/ssl/certs"
    },
    "writable": [
      { "path": "/tmp", "type": "tmpfs", "options": "size=64m" },
      { "path": "/var", "type": "tmpfs" },
      { "path": "/run", "type": "tmpfs" }
    ]
  },

  "hostname": "myapp",
  "domainname": "",

  "process": {
    "entrypoint": ["/nix/store/def456-myapp-1.0/bin/myapp"],
    "cmd": ["--config", "/etc/myapp/config.toml"],
    "env": [
      "PATH=/nix/store/def456-myapp-1.0/bin",
      "SSL_CERT_FILE=/etc/ssl/certs/ca-bundle.crt"
    ],
    "cwd": "/",
    "user": "65534",
    "group": "65534",
    "stopSignal": "SIGTERM"
  },

  "mounts": [
    {
      "destination": "/var/lib/myapp",
      "type": "tmpfs",
      "source": "tmpfs",
      "options": ["nosuid", "nodev", "size=256m"]
    },
    {
      "destination": "/etc/myapp",
      "source": "/host/config/myapp",
      "options": ["bind", "ro"]
    },
    {
      "destination": "/var/log/myapp",
      "source": "/volumes/myapp-logs",
      "options": ["rbind", "rw"]
    }
  ],

  "ports": {
    "http": {
      "port": 8080,
      "protocol": "tcp",
      "transport": "http",
      "hostname": "app.example.com",
      "path": "/",
      "internal": false,
      "openFirewall": false,
      "tls": { "enable": false, "forceRedirect": true, "acme": false },
      "healthCheck": { "path": "/health", "interval": 30 }
    },
    "metrics": {
      "port": 9090,
      "protocol": "tcp",
      "transport": "http",
      "internal": true,
      "healthCheck": { "path": "/metrics", "interval": 30 }
    }
  },

  "volumes": {
    "/var/lib/myapp": {},
    "/var/log/myapp": {}
  },

  "labels": {
    "org.opencontainers.image.title": "myapp",
    "org.opencontainers.image.version": "1.0"
  }
}
```

The manifest does **not** contain tar layers, overlayfs configuration, or a flattened filesystem image.

## Manifest Fields

### Platform

Follows the [OCI image spec](https://github.com/opencontainers/image-spec/blob/main/config.md) platform fields.

| Field | Required | Description |
| --- | --- | --- |
| `architecture` | yes | CPU architecture (e.g., `amd64`, `arm64`, `riscv64`) |
| `os` | yes | Target operating system (e.g., `linux`) |
| `variant` | no | Architecture variant (e.g., `v7` for 32-bit ARM) |

These are used for multi-arch image resolution. A registry or tool serving multiple architectures uses these fields to select the correct manifest for the target platform.

### `hostname` / `domainname`

Optional strings setting the container's hostname and domain name. Follows the [OCI runtime spec](https://github.com/opencontainers/runtime-spec/blob/main/config.md) `hostname` and `domainname` fields. The runtime may override these.

### `closure`

An ordered list of Nix store paths forming the container's complete dependency graph. Each entry contains:

| Field | Required | Description |
| --- | --- | --- |
| `storePath` | yes | Absolute path under the store prefix |
| `narHash` | yes | Content hash of the NAR archive (for verification) |
| `narSize` | yes | Size of the NAR archive in bytes |
| `references` | yes | Store paths this entry depends on (must also appear in the closure) |

### `rootfs`

Describes how the closure is projected into a filesystem tree.

- **`storeMount`** — mount point for the store (default `/nix/store`).
- **`symlinks`** — map of symlinks to create, producing an FHS-compatible layout from store paths.
- **`writable`** — paths mounted as tmpfs for runtime state. Each entry specifies `path`, `type` (currently only `tmpfs`), and optional `options`.

### `process`

| Field | Required | Description |
| --- | --- | --- |
| `entrypoint` | yes | Executable and fixed arguments |
| `cmd` | no | Default arguments appended to entrypoint (overridable at run time) |
| `env` | no | Environment variables |
| `cwd` | no | Working directory (default `/`) |
| `user` | no | UID to run as |
| `group` | no | GID to run as |
| `stopSignal` | no | Signal sent on container stop (default `SIGTERM`) |

### `mounts`

Follows the [OCI runtime spec](https://github.com/opencontainers/runtime-spec/blob/main/config.md#mounts) mount schema. Each entry is an object with:

| Field | Required | Description |
| --- | --- | --- |
| `destination` | yes | Absolute path inside the container |
| `source` | no | Device name, host path for bind mounts, or `tmpfs` for tmpfs |
| `type` | no | Filesystem type (e.g., `tmpfs`, `proc`). For bind mounts, omit or set to `none`. |
| `options` | no | Array of mount option strings (e.g., `["bind", "ro"]`, `["nosuid", "size=64m"]`) |

A mount is a **bind mount** when `options` contains `bind` or `rbind`. The `source` is the host path to bind into the container at `destination`. Read-only bind mounts include `ro` in `options`.

Mounts are processed in listed order, consistent with the OCI runtime spec. The runtime MUST mount entries in the order given.

Mounts declared in the manifest are defaults. The runtime or orchestrator may add, remove, or override mounts at container creation time.

### `ports`

Uses the port contract schema from [EEP 0041](eeps/0041-port-contracts.md). Keys are logical port names (e.g., `"http"`, `"metrics"`, `"grpc"`); values are port contract objects. Each contract contains:

| Field | Required | Description |
| --- | --- | --- |
| `port` | yes | Port number |
| `protocol` | no | `tcp` or `udp` (default `tcp`) |
| `transport` | no | `http`, `https`, `http2`, `grpc`, `tcp`, `udp` (default `http`) |
| `hostname` | no | Hostname for routing (e.g., reverse proxy virtual hosts) |
| `path` | no | Path prefix for routing (default `/`) |
| `internal` | no | If `true`, not exposed via proxy or firewall (default `false`) |
| `openFirewall` | no | If `true`, auto-open in host firewall (default `false`) |
| `tls.enable` | no | Whether the upstream speaks TLS natively (default `false`) |
| `tls.forceRedirect` | no | Proxy should redirect HTTP to HTTPS (default `true`) |
| `tls.acme` | no | Request a Let's Encrypt certificate (default `false`) |
| `healthCheck.path` | no | Health check endpoint path |
| `healthCheck.interval` | no | Seconds between checks (default `30`) |

This replaces both OCI's `ExposedPorts` (bare port declarations) and `Healthcheck` (per-container health check) with a richer per-port model. Port contracts carry enough metadata for reverse proxies, firewalls, monitoring scrapers, and ACME certificate provisioning to auto-configure themselves.

For OCI compatibility, runtimes SHOULD derive `ExposedPorts` from port contracts by collecting all non-internal `port/protocol` pairs.

### `volumes`

Declares paths that should receive external or persistent storage, following the [OCI image spec](https://github.com/opencontainers/image-spec/blob/main/config.md) `Volumes` format. Keys are absolute paths; values are empty objects.

```json
"volumes": {
  "/var/lib/myapp": {},
  "/var/log/myapp": {}
}
```

These are hints to the runtime or orchestrator. A Docker-compatible runtime creates anonymous volumes at these paths if no explicit mount is provided. Kubernetes maps them to `emptyDir` or PVC claims.

### `labels`

Arbitrary key-value metadata. OCI annotation keys (`org.opencontainers.image.*`) are recommended for interoperability.

## Runtime Assembly

A conforming runtime assembles the container as follows:

1. **Resolve closure** — ensure all store paths are present locally (fetch from binary cache or OCI registry if not).
2. **Create mount namespace** — isolate the container's mount table.
3. **Bind-mount store paths** — mount each closure path read-only under the store prefix (only the closure, not the full host store).
4. **Create symlinks** — apply the manifest's symlink directives.
5. **Mount writable layers** — mount tmpfs for paths listed in `rootfs.writable`.
6. **Process mounts** — apply `mounts` entries in order (bind mounts, tmpfs, etc.). Create anonymous volumes for any `volumes` paths not covered by an explicit mount.
7. **`pivot_root`** — switch to the assembled rootfs.
8. **Execute process** — start the entrypoint with the specified configuration.

## Service-to-Container Projection

An evaluated service module (as defined in `corepkgs`) already carries all the
information an Ekala Container manifest needs. `ekaTools.buildContainer` is a
Nix function that projects a service module evaluation into a manifest,
enabling a single service definition to target native ekaos, Ekala Container,
or OCI deployment without modification.

### Interface

```nix
ekaTools.buildContainer {
  # Required: the evaluated service config
  service = config.services.myapp;

  # Optional overrides
  name = "myapp";                          # defaults to service attr name
  tag = "1.0";                             # defaults to package version
  extraSymlinks = { "/etc/hosts" = hostsFile; };
  extraEnv = [ "LOG_LEVEL=info" ];
  extraMounts = [
    { destination = "/cache"; type = "tmpfs"; source = "tmpfs";
      options = [ "nosuid" "size=128m" ]; }
  ];
}
```

### Projection rules

The function evaluates the service module and maps its fields to manifest
fields:

| Service module | Manifest field | Notes |
| --- | --- | --- |
| `package` (full closure) | `closure` | `nix-store -qR` of the package |
| `command` or `package.meta.mainProgram` | `process.entrypoint` | Falls back to `${package}/bin/${pname}` |
| `commandArgs` | `process.cmd` | |
| `environment` | `process.env` | Merged with `extraEnv` |
| `user` / `group` | `process.user` / `process.group` | Numeric UIDs |
| `ports.*` (EEP 0041) | `ports.*` | Passed through verbatim — same schema |
| `dataDir` | `volumes.${dataDir}` | Also adds a writable tmpfs if no external mount |
| `stateDir`, `runtimeDir`, `logsDir` | `rootfs.writable` entries | tmpfs for runtime state |
| `migration.majorVersion` (EEP 0045) | `labels."dev.ekala.migration.majorVersion"` | Enables migration tooling in container context |
| `migration.strategy` | `labels."dev.ekala.migration.strategy"` | |
| Package `meta.description` | `labels."org.opencontainers.image.description"` | |
| Package `version` | `labels."org.opencontainers.image.version"` | |
| `system` | `architecture` + `os` | Parsed from Nix system string |

The symlink tree (`rootfs.symlinks`) is derived the same way `buildEnv`
assembles a profile: binaries, libraries, and `etc` files from the closure
are symlinked into FHS-conventional paths. `extraSymlinks` are merged on top.

### Example

Given this service module:

```nix
services.redis = {
  enable = true;
  package = pkgs.redis;
  dataDir = "/var/lib/redis";
  ports.redis = {
    port = 6379;
    transport = "tcp";
    internal = true;
    healthCheck.path = null;
  };
  migration.majorVersion = lib.versions.major pkgs.redis.version;
};
```

`ekaTools.buildContainer { service = config.services.redis; }` produces a
derivation that writes an Ekala Container manifest with:

- `closure` containing redis and all its runtime dependencies
- `process.entrypoint` = `["/nix/store/...-redis-7.4/bin/redis-server"]`
- `process.user` = `"65534"`
- `ports.redis` = `{ port = 6379; transport = "tcp"; internal = true; ... }`
- `volumes` = `{ "/var/lib/redis": {} }`
- `rootfs.writable` entries for `/tmp`, `/var/run`, `/var/log`
- `labels."dev.ekala.migration.majorVersion"` = `"7"`

### Deployment mode parity

The same service module works across three deployment modes:

| Mode | Port contracts used by | Closure managed by | State managed by |
| --- | --- | --- | --- |
| **Native ekaos** | Firewall, reverse proxy, monitoring modules | System profile | Host filesystem + EEP 0045 migration |
| **Ekala Container** | `ports` field in manifest; runtime maps to host | Binary cache / host store | Volumes + EEP 0045 migration via labels |
| **OCI image** (shim) | Derived `ExposedPorts` + `Healthcheck` | Registry layers | Docker volumes |

No service module changes are needed to switch between modes. The port
contracts, migration metadata, and dependency graph are the same — only the
packaging and runtime change.

## Distribution

Two strategies, not mutually exclusive:

**Nix binary cache.** Store paths are fetched via the existing narinfo/NAR protocol from any binary cache (cache.nixos.org, Cachix, self-hosted). No new infrastructure required.

**OCI registry.** Each NAR is stored as an OCI blob. The Ekala Container manifest is stored as an OCI artifact with `artifactType: application/vnd.ekala.container.manifest.v1+json`. This allows standard registry tooling (push, pull, garbage collection) to work unmodified.

# Prior Art

| Project | Approach | Limitations |
|---------|----------|-------------|
| [nix-snapshotter](https://github.com/pdtpartners/nix-snapshotter) | containerd plugin; bind-mounts store paths via OCI annotations | Tied to containerd; no portable format |
| [Nucleus](https://github.com/sig-id/nucleus) | Nix closure as rootfs; `pivot_root` | Local-only; no distribution format |
| [nix2container](https://github.com/nlewo/nix2container) | Archive-less OCI build via JSON + Skopeo | Still produces OCI tar layers |
| [Nixery](https://github.com/tazjin/nixery) | On-demand registry from Nix packages | Works within OCI layer model |
| [Imageless](https://github.com/claughd/imageless) | Ships flake, materializes rootfs at create time | Requires Nix on every node |
| [nix-csi](https://github.com/lillecarl/nix-csi) | K8s CSI driver mounting `/nix` into pods | CSI-specific; not a general format |
| [OCI issue #922](https://github.com/opencontainers/image-spec/issues/922) | Proposed NAR media types for OCI layers | Closed without adoption |
| [nixpkgs #374288](https://github.com/NixOS/nixpkgs/issues/374288) | Proposed `mounts` sibling to OCI `layers` | Requires OCI spec changes |

# Unresolved Questions

- **Store path prefix.** Should the format use `/nix/store` or a distinct prefix? A generic non-conflicting predicate would make the format applicable to Guix and OSTree as well.
- **Symlink conflicts.** How are collisions resolved when multiple store paths provide the same binary name? This is analogous to `buildEnv` collision handling.
- **Garbage collection.** The runtime must hold GC roots for running containers. The mechanism needs specification.
- **Backwards compatibility.** Options for Docker-only systems: (a) shim that converts to OCI at pull time, (b) dual-format manifests, (c) require an Ekala-aware runtime.
- **Many bind mounts.** Whether mount namespace setup time is a practical concern at real service scales needs benchmarking. Alternatives include composefs-style metadata images or a filtered bind of `/nix/store`.
- **Network policy.** Whether to include declarative network policy (egress rules, DNS config) in the manifest or leave it entirely to the orchestrator.

# Future Work

- **`ekala-snapshotter`** — containerd snapshotter plugin for Kubernetes integration.
- **`ekala-run`** — standalone CLI for running Ekala Containers without containerd.
- **OCI compatibility shim** — convert Ekala Container manifests to standard OCI images for legacy runtimes.
- **Manifest signing** — integrity verification via `narHash` and signed manifests, similar to Nucleus's rootfs attestation.
- **Compose format** — multi-container topology declaration with volume sharing and network links between Ekala Containers.
