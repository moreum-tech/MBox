[Turkce](README.tr.md)

# DigiMR

Turkish electronic signature platform. Self-contained REST API for CAdES, PAdES, XAdES, JAdES, ASiC-E signing and verification.

**Latest release:** [v1.1.0](https://github.com/moreum-tech/MBox/releases/tag/digimr-v1.1.0)

---

## Downloads

| File | Description |
|------|-------------|
| [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz) | API server binary (Linux x86_64) |
| [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-docker.tar.gz) | Docker image (`docker load`) |
| [digimr-sdk-1.0.0-all.jar](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-sdk-1.0.0-all.jar) | Java SDK fat JAR |

```bash
# Download all
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-docker.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-sdk-1.0.0-all.jar
```

### Documentation (v1.1.0 release assets)

| File | Description |
|------|-------------|
| [REST_API_KILAVUZU.md](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/REST_API_KILAVUZU.md) | REST API reference |
| [GRPC_API_KILAVUZU.md](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/GRPC_API_KILAVUZU.md) | gRPC API reference |
| [JAVA_SDK_KILAVUZU.md](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/JAVA_SDK_KILAVUZU.md) | Java SDK guide |
| [DOTNET_SDK_KILAVUZU.md](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/DOTNET_SDK_KILAVUZU.md) | .NET SDK guide |

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
