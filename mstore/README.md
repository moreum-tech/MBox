[Turkce](README.tr.md)

# MStore

High-performance, S3-compatible object storage written in Rust.

**Latest release:** [v0.2.0](https://github.com/moreum-tech/MBox/releases/tag/mstore-v0.2.0)

---

## Downloads

| File | Description |
|------|-------------|
| [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz) | Server + CLI binary (Linux x86_64) |
| [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz) | Docker image (`docker load`) |

```bash
# Download all
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz
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
