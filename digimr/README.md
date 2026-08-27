[Turkce](README.tr.md)

# DigiMR

Turkish electronic signature platform. Self-contained REST API for CAdES, PAdES, XAdES, JAdES, ASiC-E signing and verification.

**Latest release:** [v2.3.2](https://github.com/moreum-tech/MBox/releases/tag/digimr-v2.3.2)

> ⚠️ **v2.3.2 introduces breaking changes** vs v1.x — the string-path API was removed in favor of a single `byte[]`-based surface. See `CHANGELOG.md` inside `digimr-docs.tar.gz` for the migration guide.

---

## Documentation

Full product documentation lives in [docs/](docs/README.md) — **nothing needs to be installed to read it**. It is the same text that ships inside `digimr-docs.tar.gz`.

| Document | What it covers |
|----------|----------------|
| [Overview](docs/en/OVERVIEW.md) | What DigiMR is and what it can sign |
| [Deployment guide](docs/tr/KURULUM_REHBERI.md) | Binaries, systemd, Docker, TLS, sizing *(TR)* |
| [REST API reference](docs/tr/API_REFERANS.md) | Every endpoint, schema and error code *(TR)* |
| [.NET SDK reference](docs/tr/SDK_REFERANS.md) | Full type and method reference *(TR)* |
| [Examples](docs/tr/ORNEKLER.md) | 17 worked scenarios (C# + curl) *(TR)* |
| [Signature formats](docs/tr/IMZA_FORMATLARI.md) | CAdES / XAdES / PAdES / ASiC, levels B-B … B-LTA *(TR)* |
| [All documents →](docs/README.md) | Agent setup, providers, KEP, TSA, mobile signature, legislation |

## Downloads

| File | Description |
|------|-------------|
| [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-linux-amd64.tar.gz) | API server binary (Linux x86_64) |
| [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-docker.tar.gz) | Docker image (`docker load`) |
| [digimr-sdk-1.0.0-all.jar](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-sdk-1.0.0-all.jar) | Java SDK fat JAR |

```bash
# Download all
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-docker.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-sdk-1.0.0-all.jar
```

### Documentation & examples (release assets)

| File | Description |
|------|-------------|
| [digimr-docs.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-docs.tar.gz) | Developer documentation (API reference, setup, signature formats, CHANGELOG) |
| [digimr-dotnet-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-dotnet-examples.tar.gz) | .NET example projects |
| [digimr-java-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-java-examples.tar.gz) | Java example programs |
| [docker-compose.demo.yml](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/docker-compose.demo.yml) | Ready-to-run Docker compose config |

## Components

| Component | Description | Details |
|-----------|-------------|---------|
| [API Server](api/README.md) | REST API on port 7701 | Installation, configuration, endpoints |
| [Token Agent](agent/README.md) | Local PKCS#11 bridge (port 5555) | Hardware token access |
| [Docker](docker/README.md) | Container image and compose | Quick start |

## Features

- Signature formats — CAdES, PAdES, XAdES, JAdES, ASiC-E/S
- Signature levels — BES, T, LT, LTA (AdES baseline)
- EYP — 1.3 and 2.0 archive packages
- KEP — Registered electronic mail packages
- TSA — KamuSM free/paid tier, configurable
- Mobile signature — Turkcell, Vodafone, Turk Telekom
- Token support — PKCS#11 via local agent (AKiS, SafeNet, Gemalto)
- HSM support — multi-slot session pool
- Batch signing — sign multiple documents in one call
- Verification — signature validation with CRL/OCSP checks
- Swagger UI — interactive API documentation at `/swagger`

## Port

| Port | Protocol | Purpose |
|------|----------|---------|
| 7701 | HTTP | REST API and Swagger UI |

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
