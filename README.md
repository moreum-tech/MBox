[Turkce](README.tr.md)

# MBox

Official distribution repository for [Moreum](https://www.moreum.com) products.

Pre-built binaries, Docker images, and SDK packages are published as [GitHub Releases](https://github.com/moreum-tech/MBox/releases).

---

## Products

### MStore — Object Storage

High-performance, S3-compatible object storage written in Rust.

| Component | Latest | Download |
|-----------|--------|----------|
| [Server + CLI](mstore/server/README.md) | v0.3.1 | [Linux amd64](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-linux-amd64.tar.gz) · [Windows amd64](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-windows-amd64.zip) |
| [Docker / Podman](mstore/docker/README.md) | v0.3.1 | [Image](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-docker.tar.gz) |

**Documentation:** [mstore/docs/](mstore/docs/README.md) — installation, configuration, S3 & gRPC API, SDK reference. No install needed to read.

[All MStore releases](https://github.com/moreum-tech/MBox/releases?q=mstore)

---

### DigiMR — Electronic Signature

Turkish electronic signature platform — CAdES, PAdES, XAdES, JAdES, ASiC-E. EYP 1.3/2.0, KEP, KamuSM TSA, mobile signature.

| Component | Latest | Download |
|-----------|--------|----------|
| [API Server](digimr/api/README.md) | v2.3.2 | [Linux amd64](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-linux-amd64.tar.gz) |
| [Docker](digimr/docker/README.md) | v2.3.2 | [Image](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-docker.tar.gz) |
| [Java SDK](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-sdk-1.0.0-all.jar) | 1.0.0 | JAR |

**Documentation:** [digimr/docs/](digimr/docs/README.md) — deployment guide, REST API reference, .NET SDK reference, signature formats. No install needed to read.

[All DigiMR releases](https://github.com/moreum-tech/MBox/releases?q=digimr)

---

## About

[Moreum](https://www.moreum.com) is a software and consulting company established in 2004, specializing in document management, workflow automation, and digital signature solutions.

**Istanbul, Turkey** · info@moreum.com · [www.moreum.com](https://www.moreum.com)

## License

All products are distributed under PolyForm Strict License 1.0.0 unless otherwise noted.
