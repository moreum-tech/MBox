[Turkce](README.tr.md)

# MStore

High-performance, S3-compatible object storage written in Rust.

---

## Downloads

| Component | Type | Description | Latest |
|-----------|------|-------------|--------|
| [Server](server/) | Service | MStore server binary | v0.1.0 |
| [CLI](cli/) | Tool | Command-line tool (mstore-ctl) | v0.1.0 |
| [Docker](docker/) | Container | Docker image and compose | v0.1.0 |
| [Rust SDK](sdk/rust/) | SDK | Rust client library | v0.1.0 |
| [C# SDK](sdk/csharp/) | SDK | .NET client library | v0.1.0 |
| [Python SDK](sdk/python/) | SDK | Python client library | v0.1.0 |
| [Go SDK](sdk/go/) | SDK | Go client library | v0.1.0 |
| [Java SDK](sdk/java/) | SDK | Java client library | v0.1.0 |
| [JS/TS SDK](sdk/js/) | SDK | JavaScript/TypeScript library | v0.1.0 |

---

## Features

- S3 API compatible - works with AWS SDKs, CLI, and all S3 tools
- Native SDKs - Rust, C#, Python, Go, Java, JS/TS
- Encryption - SSE-S3 (AES-256-GCM), SSE-KMS, SSE-C
- Erasure coding - Reed-Solomon data/parity shards
- Zero-copy streaming - for large objects (>4 MiB)
- Versioning - full object versioning with delete markers
- Object lock - WORM compliance (SEC 17a-4)
- Web console - browser-based management UI
- Full-text search - per-bucket Tantivy indexes
- Event system - Webhook, Kafka, AMQP, NATS, SQS, SNS
- Site replication - async/sync via gRPC
- SFTP gateway - SSH file transfer access
- IAM - LDAP, OIDC integration

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9010 | HTTP | S3-compatible API |
| 9011 | gRPC | MStore SDK, CLI, replication |

---

info@moreum.com | [www.moreum.com](https://www.moreum.com)
