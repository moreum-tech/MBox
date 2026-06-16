[Turkce](README.tr.md)

# DigiMR

Turkish electronic signature platform. Self-contained REST API for CAdES, PAdES, XAdES, JAdES, ASiC-E signing and verification.

**Latest release:** [v2.0.0](https://github.com/moreum-tech/MBox/releases/tag/digimr-v2.0.0)

> ⚠️ **v2.0.0 introduces breaking changes** vs v1.x — the string-path API was removed in favor of a single `byte[]`-based surface. See `CHANGELOG.md` inside `digimr-docs.tar.gz` for the migration guide.

---

## Downloads

| File | Description |
|------|-------------|
| [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-linux-amd64.tar.gz) | API server binary (Linux x86_64) |
| [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-docker.tar.gz) | Docker image (`docker load`) |
| [digimr-sdk-1.0.0-all.jar](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-sdk-1.0.0-all.jar) | Java SDK fat JAR |

```bash
# Download all
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-docker.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-sdk-1.0.0-all.jar
```

### Documentation & examples (release assets)

| File | Description |
|------|-------------|
| [digimr-docs.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-docs.tar.gz) | Developer documentation (API reference, setup, signature formats, CHANGELOG) |
| [digimr-dotnet-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-dotnet-examples.tar.gz) | .NET example projects |
| [digimr-java-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/digimr-java-examples.tar.gz) | Java example programs |
| [docker-compose.demo.yml](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.0.0/docker-compose.demo.yml) | Ready-to-run Docker compose config |

## Components

| Component | Description | Details |
|-----------|-------------|---------|
| [API Server](api/) | REST API on port 7701 | Installation, configuration, endpoints |
| [Token Agent](agent/) | Local PKCS#11 bridge (port 5555) | Hardware token access |
| [Docker](docker/) | Container image and compose | Quick start |

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
