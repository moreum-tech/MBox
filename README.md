[Turkce](README.tr.md)

# MBox

Official distribution repository for Moreum products. Pre-built binaries, Docker images, and SDK packages are published here.

> This repository contains no source code - only compiled, downloadable packages.

---

## Products

### MStore - Object Storage

High-performance, S3-compatible object storage written in Rust.

| Component | Description | Latest |
|-----------|-------------|--------|
| [Server](mstore/server/) | MStore server binary | v0.1.0 |
| [CLI](mstore/cli/) | Command-line tool (mstore-ctl) | v0.1.0 |
| [Docker](mstore/docker/) | Docker image and compose | v0.1.0 |
| [SDK - Rust](mstore/sdk/rust/) | Rust client library | v0.1.0 |
| [SDK - C#](mstore/sdk/csharp/) | .NET client library | v0.1.0 |
| [SDK - Python](mstore/sdk/python/) | Python client library | v0.1.0 |
| [SDK - Go](mstore/sdk/go/) | Go client library | v0.1.0 |
| [SDK - Java](mstore/sdk/java/) | Java client library | v0.1.0 |
| [SDK - JS/TS](mstore/sdk/js/) | JavaScript/TypeScript library | v0.1.0 |

[View all MStore downloads](mstore/)

---

### DigiMR - Electronic Signature

Turkish electronic signature platform — CAdES, PAdES, XAdES, JAdES, ASiC-E. EYP 1.3/2.0, KEP, KamuSM TSA, mobile signature.

| Component | Description | Latest |
|-----------|-------------|--------|
| [API Server](digimr/api/) | REST API server binary | v0.1.0 |
| [Token Agent](digimr/agent/) | Local PKCS#11 bridge for hardware tokens | v0.1.0 |
| [Docker](digimr/docker/) | Docker image and compose | v0.1.0 |
| [SDK - C#](digimr/sdk/csharp/) | .NET client library | v0.1.0 |

[View all DigiMR downloads](digimr/)

---

## About Moreum

[Moreum](https://www.moreum.com) is a software and consulting company established in 2004, specializing in document-based technology solutions. The company delivers customer-focused products and services in document management, workflow automation, form processing, and resource planning across European and Turkish markets.

**Istanbul, Turkey** | info@moreum.com | [www.moreum.com](https://www.moreum.com)

## License

All products are distributed under PolyForm Strict License 1.0.0 unless otherwise noted.
