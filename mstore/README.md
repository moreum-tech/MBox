[Turkce](README.tr.md)

# MStore

High-performance, S3-compatible object storage written in Rust.

**Latest release:** [v0.3.1](https://github.com/moreum-tech/MBox/releases/tag/mstore-v0.3.1)

---

## Documentation

Full product documentation lives in [docs/](docs/) — **nothing needs to be installed to read it**.

| Document | What it covers |
|----------|----------------|
| [Installation](docs/installation.md) | Binaries, systemd, zero-config mode, first bucket |
| [Configuration](docs/configuration.md) | Every `config.toml` field, CLI flag and env var |
| [S3 API](docs/s3-api.md) | S3 HTTP API, presigned URLs, admin API, AWS SDK examples |
| [gRPC API](docs/grpc-api.md) | Native gRPC API — 18 RPCs, streaming, batch, error model |
| [SDK Reference](docs/sdk-reference.md) | Rust, Python, Go, Java, .NET, JS/TS clients and the CLI |
| [Docker deployment](docs/docker-deployment.md) | Container image, compose, volumes |
| [All documents →](docs/README.md) | Multi-drive, multi-node, deployment scenarios, operations |

## Downloads

| File | Platform | Description |
|------|----------|-------------|
| [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-linux-amd64.tar.gz) | Linux x86_64 | Server + CLI binary (`mstore-server`, `mstore`) |
| [mstore-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-windows-amd64.zip) | Windows x86_64 | Server + CLI binary (`mstore-server.exe`, `mstore.exe`) |
| [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-docker.tar.gz) | Docker / Podman | OCI image (`docker load` / `podman load`) |

```bash
# Download (Linux)
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-docker.tar.gz
```

## Components

| Component | Description | Details |
|-----------|-------------|---------|
| [Server](server/) | S3 HTTP + gRPC endpoints | Installation, configuration |
| [Docker](docker/) | Container image and compose | Quick start |

## Features

- S3 API compatible — works with AWS SDKs, CLI, and all S3 tools
- Erasure coding — Reed-Solomon data/parity shards
- Encryption — AES-256-GCM, ChaCha20-Poly1305
- Object versioning with delete markers
- Object lock — WORM compliance (SEC 17a-4)
- Full-text search — per-bucket Tantivy indexes
- IAM — user/group policies, LDAP, OIDC
- Site replication — async/sync via gRPC
- SFTP gateway — SSH file transfer access
- Web console — browser-based management UI
- Batch operations — bulk delete/copy/tag
- S3 Select — CSV and JSON query

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9010 | HTTP | S3-compatible API |
| 9011 | gRPC | SDK, CLI, replication |

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
