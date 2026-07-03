---
EEP: 0041
Title: Port contracts for service networking
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-07-02
---

# Motivation

Services in `corepkgs` declare ports ad-hoc -- hardcoded in config strings,
passed via environment variables, or buried in service-specific settings
(e.g. `sshd.settings.ports`). No centralized tracking exists, so:

- Two services can silently claim the same port
- Reverse proxy modules (traefik, nginx) cannot auto-discover upstream services
- Docker image builds require manual `exposedPorts` lists
- Firewall rules must be hand-written per service
- There is no structured way to express "this service needs TLS" or "this
  service should be reachable at app.example.com"

NixOS uses a **decentralized** approach: each service has an `openFirewall`
bool that manually writes to `networking.firewall.allowedTCPPorts`; nginx
`virtualHosts` must be configured per upstream service; ACME certificates are
requested per virtual host. There is no central service registry, no collision
detection, and no auto-discovery. Every cross-service wiring is manual.

This proposal introduces **port contracts** -- a centralized, structured
option on every service that declares which ports it listens on, what
hostnames they map to, and whether they are internal or external-facing.
An aggregation module collects all contracts at evaluation time and provides
derived read-only options that consumer modules (reverse proxy, firewall, DNS,
monitoring) read to auto-configure themselves. This eliminates an entire class
of configuration errors and boilerplate.

# Detailed Specification

## The port contract type

A new `ports` option is added to the common service options in
`services/lib/options.nix`. It is an `attrsOf (submodule portContract)`,
keyed by a logical port name (e.g. `"http"`, `"metrics"`, `"grpc"`).

Each port contract contains:

```nix
services.myapp.ports = {
  http = {
    port = 8080;               # required, types.port
    protocol = "tcp";           # "tcp" | "udp" (default: "tcp")
    transport = "http";         # "http" | "https" | "http2" | "grpc" | "tcp" | "udp"
    hostname = "app.example.com"; # null = no hostname-based routing
    path = "/";                 # path prefix for reverse proxy routing
    internal = false;           # true = not exposed via proxy or firewall
    openFirewall = false;       # true = auto-open in networking.firewall
    tls = {
      enable = false;           # whether the upstream speaks TLS natively
      forceRedirect = true;     # proxy should redirect HTTP -> HTTPS
      acme = false;             # request a Let's Encrypt certificate
    };
    healthCheck = {
      path = "/health";         # null = no health check
      interval = 30;            # seconds
    };
  };
  metrics = {
    port = 9090;
    internal = true;
    transport = "http";
    healthCheck.path = "/metrics";
  };
};
```

The option defaults to `{}`, so every existing service continues to work
unchanged.

## Design decisions

**No `protocol = "both"`.**  A service listening on both TCP and UDP on the
same port declares two entries (`dns-tcp`, `dns-udp`). This keeps collision
detection trivial and makes the contract explicit about what each entry
represents.

**`openFirewall` per-port, not per-service.**  NixOS puts `openFirewall` at
the service level. A service with an HTTP port (external) and a metrics port
(internal) needs different firewall behavior per port. Per-port granularity
eliminates manual firewall exceptions.

**`healthCheck` in the contract.**  Health check endpoints are already used
ad-hoc in Docker examples (`/health`, `/nginx_status`). Formalizing them
enables reverse proxy modules to auto-configure upstream health checks and
monitoring consumers to auto-discover scrape targets.

**`tls.acme` in the contract.**  NixOS requires manual `enableACME` per nginx
virtual host. By putting ACME in the port contract, certificate provisioning is
declared by the service that needs it, not by the proxy that happens to serve
it. The proxy consumer reads the contract and auto-wires ACME.

## Architecture: two evaluation layers

`corepkgs` has two independent service evaluation paths:

1. **Standalone services library** (`services/`) -- uses its own
   `lib.evalModules`; all services go through `serviceOpts` in
   `service-module.nix`, which applies `commonOptions` automatically.
2. **ekaos system builder** (`ekaos/`) -- uses a separate `lib.evalModules`;
   each service module (sshd, crond, getty) manually redeclares common options.

Port contracts work in both:

- `commonOptions` gets the `ports` option, covering the standalone library
  automatically.
- ekaos service modules that want port contracts add `ports` to their own
  option definitions.
- A new ekaos aggregation module (`networking/port-contracts.nix`) collects
  contracts, detects collisions, and exposes derived data for consumers.

## Type definitions

Added to `services/lib/types.nix`:

```nix
protocol = types.enum [ "tcp" "udp" ];

transport = types.enum [ "http" "https" "http2" "grpc" "tcp" "udp" ];

portContract = {
  options = {
    port         = mkOption { type = types.port; };
    protocol     = mkOption { type = protocol; default = "tcp"; };
    transport    = mkOption { type = transport; default = "http"; };
    hostname     = mkOption { type = types.nullOr types.str; default = null; };
    path         = mkOption { type = types.str; default = "/"; };
    internal     = mkOption { type = types.bool; default = false; };
    openFirewall = mkOption { type = types.bool; default = false; };
    tls          = mkOption { type = types.submodule tlsOpts; default = {}; };
    healthCheck  = mkOption { type = types.submodule healthCheckOpts; default = {}; };
  };
};
```

## Validation

### Per-service consistency checks

Added to `services/lib/validate.nix` and run for every enabled service:

- `tls.acme = true` requires `hostname` to be non-null
- `tls.acme = true` requires `transport` to be `http`, `https`, `http2`, or
  `grpc`
- `healthCheck.path` set with a non-HTTP transport emits a warning
- `openFirewall = true` and `internal = true` emits a warning (contradiction)

### Cross-service collision detection

A new `checkPortCollisions` function collects all `(port, protocol)` pairs
across enabled services, groups them, and returns an error for any key claimed
by more than one service:

```nix
checkPortCollisions = services:
  let
    allPorts = lib.concatLists (lib.mapAttrsToList (svcName: cfg:
      lib.mapAttrsToList (portName: pc: {
        inherit svcName portName;
        inherit (pc) port protocol;
        key = "${toString pc.port}/${pc.protocol}";
      }) (cfg.ports or {})
    ) services);
    grouped = lib.groupBy (p: p.key) allPorts;
    collisions = lib.filterAttrs (_: es: builtins.length es > 1) grouped;
  in
  lib.mapAttrsToList (key: entries:
    mkError "Port collision on ${key}: claimed by ${
      lib.concatMapStringsSep ", " (e: "${e.svcName}.${e.portName}") entries
    }"
  ) collisions;
```

This runs in both the standalone `validateServices` function and the ekaos
aggregation module (via assertions).

## ekaos aggregation module

`ekaos/modules/networking/port-contracts.nix` is the central consumer hub.
It iterates over all `config.services.*`, collects port contracts, and exposes
derived read-only options:

| Option                         | Type                        | Description                                    |
|--------------------------------|-----------------------------|------------------------------------------------|
| `networking.ports.contracts`   | `listOf attrs`              | All port contracts from all enabled services    |
| `networking.ports.external`    | `listOf attrs`              | Subset where `internal = false`                 |
| `networking.ports.byHostname`  | `attrsOf (listOf attrs)`    | Grouped by hostname for reverse proxy consumers |
| `networking.ports.acmeHosts`   | `listOf str`                | Hostnames needing ACME certificates             |
| `networking.ports.lookup`      | `attrsOf port`              | Flat `"svc.port" -> portNumber` map             |
| `networking.ports.firewall.tcp`| `listOf port`               | TCP ports with `openFirewall = true`            |
| `networking.ports.firewall.udp`| `listOf port`               | UDP ports with `openFirewall = true`            |
| `networking.ports.healthChecks`| `listOf attrs`              | Contracts with health check paths               |

The module also:

- Generates `config.assertions` for port collisions
- Auto-generates `/etc/hosts` entries mapping declared hostnames to `127.0.0.1`

## Docker integration

`services/lib/docker-image.nix` auto-derives `exposedPorts` from port
contracts when the user does not explicitly provide them:

```nix
autoExposedPorts = concatLists (mapAttrsToList (_: cfg:
  mapAttrsToList (_: pc: "${toString pc.port}/${pc.protocol}")
    (cfg.ports or {})
) (filterAttrs (_: cfg: cfg.enable) services));

effectiveExposedPorts = if exposedPorts != [] then exposedPorts
                        else lib.unique autoExposedPorts;
```

Explicit `exposedPorts` takes precedence for backward compatibility.

## Public API

Two new functions are exported from `services/default.nix`:

- `getPortContracts` -- extracts all port contracts from an evaluated services
  config as a flat list of records
- `checkPortContracts` -- validates port contracts and throws on collisions
  (useful for CI without building)

# Example Usage

## Declaring port contracts on a service

```nix
# ekaos/modules/services/networking/sshd.nix
services.openssh.ports.ssh = {
  port = cfg.settings.ports;
  protocol = "tcp";
  transport = "tcp";
  internal = true;
  openFirewall = true;
};
```

## A web application with multiple ports

```nix
services.myapp = {
  enable = true;
  command = "${myapp}/bin/server";
  ports = {
    http = {
      port = 8080;
      hostname = "app.example.com";
      tls.acme = true;
      healthCheck.path = "/health";
    };
    metrics = {
      port = 9090;
      internal = true;
      healthCheck.path = "/metrics";
    };
    grpc = {
      port = 9000;
      transport = "grpc";
      hostname = "app.example.com";
      path = "/grpc";
    };
  };
};
```

## Auto-configured reverse proxy

Enabling the reverse proxy module reads `networking.ports.byHostname` and
generates a complete nginx configuration with virtual hosts, proxy_pass
directives, and optional TLS/ACME:

```nix
services.reverseProxy.enable = true;

# That's it. The proxy reads port contracts and auto-configures:
# - server blocks for each hostname
# - proxy_pass to each upstream service
# - SSL certificates for hostnames with tls.acme = true
# - HTTP -> HTTPS redirects for hostnames with tls.forceRedirect = true
```

## Querying port contracts in the standalone library

```nix
let
  services = import ./services { inherit pkgs; };

  contracts = services.getPortContracts {
    web = { enable = true; command = "..."; ports.http = { port = 8080; }; };
    api = { enable = true; command = "..."; ports.http = { port = 3000; }; };
  };
in
# contracts is a list:
# [ { serviceName = "web"; portName = "http"; port = 8080; ... }
#   { serviceName = "api"; portName = "http"; port = 3000; ... } ]
```

## CI collision check

```nix
# Throws an error at eval time if any two services claim the same port
services.checkPortContracts myServicesConfig
```

## Consumer integration patterns

The port contracts system enables these consumers, all reading from
`config.networking.ports.*`:

| Consumer          | Reads                              | Produces                                |
|-------------------|------------------------------------|-----------------------------------------|
| Firewall          | `firewall.tcp`, `firewall.udp`     | `networking.firewall.allowed*Ports`     |
| Reverse proxy     | `byHostname`                       | nginx/traefik vhost configs             |
| ACME/cert manager | `acmeHosts`                        | `security.acme.certs` entries           |
| DNS zone gen      | `byHostname`                       | DNS A/CNAME records                     |
| Docker builder    | `contracts`                        | OCI `ExposedPorts`                      |
| Monitoring        | `healthChecks`                     | Prometheus scrape targets               |
| `/etc/hosts`      | `contracts` with `hostname`        | Local DNS resolution                    |
| Service discovery | `lookup`                           | `"svc.port" -> portNumber` map          |

# Prior Art

**NixOS `openFirewall` pattern** -- each service defines its own
`openFirewall` bool and manually writes to
`networking.firewall.allowedTCPPorts`. Completely decentralized with no
collision detection. Port contracts improve on this with per-port granularity,
centralized aggregation, and automatic consumer wiring.

**NixOS nginx `virtualHosts`** -- rich per-vhost configuration with
`serverName`, `listen`, `locations`, `enableACME`, `forceSSL`. Each upstream
must be manually wired. Port contracts invert this: the upstream declares its
routing intent, and the proxy module auto-discovers it.

**NixOS ACME integration** -- nginx reads from `config.security.acme.certs`
for certificate paths and writes to `security.acme.certs` to request new
certificates. Port contracts simplify this: the service declares `tls.acme =
true` and the proxy consumer handles the rest.

**Kubernetes Service / Ingress** -- Kubernetes separates port declarations
(in Pod specs) from routing (Ingress resources). Port contracts combine both
in a single declaration, eliminating the need for separate routing resources
in a single-machine context.

**Docker Compose `ports` / `expose`** -- declares port mappings and internal
port exposure. Port contracts provide richer metadata (hostname, path, TLS,
health checks) and integrate with the Nix module system for type-safe
evaluation-time validation.

# Unresolved Questions

- Should the `transport` enum be extensible (e.g. for websocket, mqtt)?
  Currently it is a closed enum. A `types.str` with convention-based values
  would be more flexible but lose type checking.
- Should port contracts support port ranges (e.g. for passive FTP or RTP)?
  Currently only single ports are supported to keep collision logic simple.
- For multi-host ekaos deployments, should port contracts carry a `bindAddress`
  field to distinguish which interface a port binds to?
- The ekaos service modules redeclare common options ad-hoc rather than using
  the shared `serviceOpts` submodule. This means each ekaos service module that
  wants port contracts must add the `ports` option to its own definitions.
  Longer term, unifying the option declaration paths would eliminate this
  duplication.

# Future Work

- **Firewall module** -- ekaos does not yet have a firewall module.
  `networking.ports.firewall.{tcp,udp}` is ready for a future module to
  consume.
- **ACME module** -- a certificate management module that reads
  `networking.ports.acmeHosts` and auto-provisions Let's Encrypt certificates.
- **DNS zone generation** -- a module that reads `networking.ports.byHostname`
  and generates DNS zone files for authoritative DNS servers.
- **Prometheus scrape config** -- a monitoring module that reads
  `networking.ports.healthChecks` and auto-generates Prometheus scrape targets.
- **Multi-machine test wiring** -- the ekaos test infrastructure
  (`ekaos/lib/testing/nodes.nix`) could use port contracts to auto-wire
  inter-VM networking in multi-machine tests.
- **Gradual service migration** -- existing services (crond, getty, dhcpcd)
  can adopt port contracts incrementally. The system is fully backward
  compatible.

# Reference Implementation

[ekala-project/corepkgs#115](https://github.com/ekala-project/corepkgs/pull/115)

