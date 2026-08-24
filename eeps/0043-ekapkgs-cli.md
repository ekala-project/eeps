---
EEP: 0043
Title: ekapkgs-cli -- negotiated binary cache protocol and CLI
Author: Jonathan Ringer
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2026-08-24
---

# Motivation

The nix binary cache protocol has a fundamental round-trip problem. When a
client runs `nix run nixpkgs#firefox`, nix evaluates the derivation, computes
the runtime closure (often 150+ store paths), and queries the binary cache for
each path individually. Each path requires a `HEAD` request to check existence,
a `GET` to fetch the `.narinfo` metadata, and another `GET` to download the
NAR archive. For a 150-path closure this means roughly 450 HTTP requests before
the package is ready to run. HTTP/2 multiplexing amortizes connection overhead,
but the sequential dependency between narinfo discovery and NAR download
remains.

The signing model is similarly rigid. Each binary cache signs with a single
ed25519 key. Clients pin trusted public keys in `nix.conf` via
`trusted-public-keys`. Rotating a cache's signing key requires every client to
update its configuration. There is no delegation, no certificate chain, and no
way to express "I trust anything signed by this cache operator" without pinning
the exact key material.

These limitations are structural. They cannot be resolved by optimizing the
existing protocol — the cache has no batch query endpoint, and the trust model
has no indirection layer. Addressing them requires a new protocol that
coexists with the existing one.

This proposal introduces `ekapkgs-cli`, consisting of two binaries: `ekapkgs`
(client) and `ekapkgs-serve` (server). The client wraps the nix CLI to provide
the same user experience as `nix run`, `nix build`, and `nix shell`, but uses
a negotiation protocol that resolves the entire closure in a single round trip.
The server supports this protocol alongside the standard nix binary cache
protocol, maintaining full backward compatibility with existing nix tooling.

# Design goals

1. **One round trip per closure.** The client sends the full set of needed
   store path hashes to the server. The server responds with a manifest of
   everything it can provide. No per-path discovery.

2. **Key rotation without client changes.** A certificate-based trust model
   lets the server rotate signing keys by deploying a new certificate signed
   by the same root CA. Clients trust the CA, not individual keys.

3. **Full nix compatibility.** The server serves standard binary cache
   endpoints. `nix build --substituters http://your-server` works unchanged.
   The negotiation protocol is an additional layer, not a replacement.

4. **Serve any nix store.** The server can serve directly from a machine's
   local `/nix/store` (like `nix-serve`), from a pre-populated cache
   directory, or from object storage. Any machine with a nix store can become
   a cache server with zero preparation.

5. **Extensible protocol.** The wire format supports forward and backward
   compatible schema evolution so that clients and servers at different
   versions interoperate without coordination.

# Negotiation protocol

## The problem

For each store path in a closure, nix makes independent HTTP requests:

```
Client → HEAD /{hash}.narinfo → Server     (×N, check existence)
Client → GET  /{hash}.narinfo → Server     (×N, fetch metadata)
Client → GET  /nar/{hash}.nar.zst → Server (×N, download archive)
```

For an N-path closure this is approximately 3N requests. The narinfo lookups
are serialized by the dependency graph — a path's references must be resolved
before the client knows what else to fetch.

## The solution

The client computes the full closure locally (nix already does this during
evaluation) and sends it to the server in a single request:

```
Client → gRPC Negotiate({want, have}) → Server      (×1)
Server → NegotiateResponse {manifest, download_plan} → Client  (×1)
Client → GET /nar/{hash}.nar.zst → Server (×M, parallel, M ≤ N)
```

The `want` set contains store path hashes the client needs but does not have
locally. The `have` set contains hashes already in the client's local store.
The server responds with narinfo-equivalent metadata for every path it can
provide, plus a topologically-sorted download plan. The client then downloads
only the missing NARs, in parallel, following the plan's batch ordering.

This reduces the discovery phase from O(N) round trips to O(1). The download
phase remains O(M) requests, but M is typically much smaller than N because
the client's `have` set excludes paths already present locally.

## Wire format: protobuf over gRPC

The negotiation protocol uses protobuf served over gRPC. This choice is driven
by three concerns:

**Schema evolution.** Protobuf's field numbering provides backward and forward
compatibility without explicit version negotiation. A client compiled against
an older `.proto` ignores unknown fields. A server compiled against a newer
`.proto` treats missing optional fields as defaults. This is critical for a
protocol where clients and servers will be updated independently.

**Cross-language codegen.** The `.proto` files are the canonical protocol
specification. Anyone building a compatible client or server in Go, Python, or
another language generates types from the same source. This lowers the barrier
for third-party tooling and avoids the documentation-drift problem of informal
JSON schemas.

**Streaming.** gRPC server-side streaming provides a natural extension point
for future work such as multiplexed NAR delivery over a single connection.

The standard nix binary cache compatibility endpoints remain plain HTTP. The
server multiplexes gRPC and HTTP on the same port using content-type routing.

## Download plan

The server constructs the download plan by topologically sorting the
dependency graph derived from each path's references:

- **Batch 0**: paths whose references are all in the client's `have` set.
- **Batch 1**: paths whose references are all in batch 0 or `have`.
- **Batch N**: paths whose references are all in prior batches or `have`.

Within each batch, paths are sorted by size ascending so the client sees
progress quickly. The client can pipeline: begin importing batch N into the
nix store while downloading batch N+1.

## Compression negotiation

The client includes preferred compression algorithms in the request. The
server selects the best available format for each path. If a NAR is available
in multiple compressions, the server returns the URL matching the client's
preference. This avoids the current situation where a cache operator must
choose a single compression format for all paths.

## Fallback

The client detects server capability via gRPC health check or a simple HTTP
probe. If the server does not support the negotiation protocol, the client
falls back to delegating directly to `nix build` with `--substituters`. This
means the client works against any nix binary cache, not just
`ekapkgs-serve`.

# Signing

## Standard nix signing (always enabled)

The server always produces standard nix ed25519 narinfo signatures — the
`Sig:` field in `.narinfo` files. This is non-optional. It ensures that:

- `nix build --substituters http://your-server` works with `trusted-public-keys`
- Existing nix tooling (`nix copy`, `nix-store`, cachix, etc.) interoperates
  without changes
- The negotiation protocol includes these signatures in the manifest

A server configured with only a standard nix signing key and no CA certificate
is fully functional. Certificate-based signing is an additional layer, not a
replacement.

## Certificate-based signing (opt-in)

The standard nix trust model pins exact public keys on every client. Key
rotation requires updating `nix.conf` on every machine that uses the cache.
For organizations operating their own caches, this creates a coordination
problem: either keys are never rotated (a security risk), or every rotation
becomes a distributed configuration update.

The certificate model introduces a simple delegation chain:

```
Root CA key (offline, in HSM or cold storage)
  ├── signs → Signing Certificate 2025 (deployed to server, valid 1 year)
  └── signs → Signing Certificate 2026 (deployed after rotation)
```

The client trusts the root CA's public key, configured once. The server
includes its certificate chain in every negotiate response. The client
verifies:

1. The signing certificate's issuer matches a trusted root CA.
2. The issuer's signature over the certificate is valid.
3. The certificate is within its validity period.
4. Each path's signature is valid against the certificate's key, over the
   standard nix fingerprint format (`1;{path};{nar_hash};{nar_size};{refs}`).

Key rotation means deploying a new certificate signed by the same root CA.
No client configuration changes are needed.

# Client

The `ekapkgs` binary wraps the nix CLI. It delegates evaluation, building, and
store operations to nix and handles substitution through the negotiation
protocol. The goal is that `ekapkgs run ekapkgs#openssh` feels identical to
`nix run ekapkgs#openssh` but with faster substitution and better signing.

## Command flow

1. Parse CLI arguments.
2. Evaluate the installable via `nix build --dry-run --json` to get output
   paths, then `nix derivation show -r` to discover the full closure.
3. Query the local nix store to partition closure paths into `have` and `want`.
4. Send `Negotiate({want, have})` to the configured cache server.
5. Download missing NARs in parallel, following the server's download plan.
6. Stage downloaded NARs as a local binary cache and import via `nix copy`.
7. For paths the server did not have, fall back to standard nix substitution.
8. Execute the result (for `run`) or report the output path (for `build`).

## User experience

The CLI follows the patterns established by `nh`: colored log prefixes,
progress bars during downloads, spinners during evaluation and negotiation,
dry-run mode, and confirmation prompts before large operations. The user
should see what is happening at each stage without being overwhelmed by
nix's raw build output.

# Server

The `ekapkgs-serve` binary serves both the negotiation protocol and the
standard nix binary cache protocol on the same port.

## Storage backends

The server abstracts storage behind a trait. Three backends are planned:

**Nix store.** Serves directly from the local `/nix/store` via the nix daemon.
This is the zero-setup option: any machine with a nix store can become a cache
server by running `ekapkgs-serve --storage nix-store`. Path metadata is
obtained from `nix path-info --json`. NARs are streamed via `nix-store --dump`,
piped through compression. Signing is performed on-the-fly with the server's
key, since the local store may contain paths signed by different keys or
unsigned paths.

**Filesystem.** Reads from a pre-populated cache directory containing
`.narinfo` files and compressed NAR archives — the same layout produced by
`nix copy --to file:///path`.

**Object storage.** S3, R2, or compatible backends for cloud deployments.

# Unresolved questions

- Whether the client should support acting as a nix substituter plugin
  (registered via `nix.conf`) rather than requiring the `ekapkgs` wrapper
  binary. This would allow `nix build` to use the negotiation protocol
  directly, but nix's substituter interface is limited and not designed for
  batch queries.
- Whether the download plan should account for the critical execution path.
  For `ekapkgs run`, the target binary and its direct dynamic library
  dependencies are needed before execution; leaf dependencies of other
  closure paths are not. Prioritizing the critical path could reduce
  time-to-first-run for large closures.
- Whether to support push-based cache population (clients uploading build
  results) in the initial scope or defer it. Push support would make
  `ekapkgs-serve` a full replacement for cachix-style services.
- The exact integration path for snix CAS. The protocol includes a
  content-addressed field per path, but snix's chunked blob storage model
  would require additional RPC methods for chunk-level negotiation.

# Future work

- **snix CAS integration.** Serve content-addressed chunks instead of full
  NARs, enabling sub-file deduplication across the store.
- **gRPC NAR streaming.** A `StreamNars` RPC that multiplexes NAR data for
  multiple paths over a single gRPC stream, eliminating separate HTTP
  downloads entirely.
- **Delta transfers.** Binary diffs between NAR versions for packages that
  change frequently but have small deltas.
- **Threshold signing.** Require k-of-n independent signatures for
  high-security deployments.
- **Cache pre-warming.** A command that computes the closure diff between two
  `flake.lock` versions and pre-downloads the delta, for CI/CD pipelines.
- **Metrics.** Server-side metrics for negotiate request counts, cache hit
  rates, and bytes served.
- **Resumable downloads.** HTTP Range header support for partial NAR fetches
  after interrupted transfers.

# Changes

N/A
