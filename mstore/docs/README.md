[Türkçe](README.tr.md)

# MStore Documentation

Complete product documentation for MStore — S3-compatible object storage, gRPC API and client SDKs.

> **You do not need to install anything to read these.** Every document below
> describes MStore v0.4.0 as shipped, so you can evaluate what it does — API
> surface, configuration knobs, deployment topologies, SDK shape — before
> downloading a single binary.

**Language note:** most documents are bilingual (Turkish section first, English
second). A few are single-language and are marked as such in the tables below.

---

## Start here

| If you want to… | Read |
|---|---|
| Get it running in five minutes | [installation.md](installation.md) — binaries, systemd, first bucket |
| Run it as a container | [docker-deployment.md](docker-deployment.md) — Docker / Podman |
| Decide whether it fits your use case | [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — segments, sizing, topologies *(TR)* |
| Point an existing S3 application at it | [s3-api.md](s3-api.md) — S3 API and AWS SDK examples |
| Write code against the native API | [sdk-reference.md](sdk-reference.md) — client SDKs in six languages |
| Look up a configuration key | [configuration.md](configuration.md) — every TOML field and default |

## Installation & operations

| Document | What it covers |
|---|---|
| [installation.md](installation.md) | Binary install, systemd unit, zero-config mode, first steps |
| [docker-deployment.md](docker-deployment.md) | Container image, compose, volumes, health checks |
| [configuration.md](configuration.md) | Full `config.toml` reference, CLI flags, environment variables |
| [multidrive-setup.md](multidrive-setup.md) | Erasure coding across multiple drives — K+M sizing *(TR)* |
| [multinode-setup.md](multinode-setup.md) | Multi-node cluster, placement groups, peers |
| [deployment-scenarios.tr.md](deployment-scenarios.tr.md) | Who uses MStore and with which topology *(TR)* |
| [orphan-reclamation.md](orphan-reclamation.md) | Async reclamation of data files left by deletes *(EN)* |

## Developer reference

| Document | What it covers |
|---|---|
| [s3-api.md](s3-api.md) | S3 HTTP API — bucket/object operations, sub-resources, presigned URLs, full-text search, admin API, AWS SDK examples |
| [grpc-api.md](grpc-api.md) | Native gRPC API — 18 RPCs, message fields, streaming, batch, `loc_token`, error model |
| [sdk-reference.md](sdk-reference.md) | Client SDKs — Rust, Python, Go, Java, .NET, JS/TS — plus the `mstore` CLI |
| [mstore.proto](mstore.proto) | The protobuf contract itself — generate stubs for any language |

---

## Two APIs, two ports

| Port | Protocol | Authentication | Use it for |
|---|---|---|---|
| 9010 | S3 over HTTP(S) | AWS SigV4, IAM, presigned URLs | Existing S3 applications, browsers, tooling |
| 9011 | gRPC | **None — every request runs as root** | Trusted-network, high-throughput clients |

> Port `9011` performs no per-request authentication. Never expose it directly
> to the internet — see [grpc-api.md](grpc-api.md).

---

## Downloads

See the [MStore product page](../README.md) for the current version and
download links.

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
