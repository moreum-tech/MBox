[Turkce](README.tr.md)

# DigiMR

Turkish electronic signature platform. Self-contained REST API for CAdES, PAdES, XAdES, JAdES, ASiC-E signing and verification.

---

## Downloads

| Component | Type | Description | Latest |
|-----------|------|-------------|--------|
| [API Server](api/) | Service | REST API server binary | v0.1.0 |
| [Token Agent](agent/) | Tool | Local PKCS#11 bridge for hardware tokens | v0.1.0 |
| [Docker](docker/) | Container | Docker image and compose | v0.1.0 |
| [C# SDK](sdk/csharp/) | SDK | .NET client library | v0.1.0 |

---

## Features

- Signature formats — CAdES, PAdES, XAdES, JAdES, ASiC-E/S
- Signature levels — BES, T, LT, LTA (AdES baseline)
- EYP — 1.3 and 2.0 archive packages
- KEP — Registered electronic mail packages
- TSA — KamuSM free/paid tier, configurable
- Mobile signature — Turkcell, Vodafone, Turk Telekom
- Token support — PKCS#11 via local agent (AKiS, SafeNet, Gemalto)
- Batch signing — sign multiple documents in one call
- Verification — signature validation with CRL/OCSP checks
- Swagger UI — interactive API documentation at `/swagger`

## Port

| Port | Protocol | Purpose |
|------|----------|---------|
| 7701 | HTTP | REST API and Swagger UI |

---

info@moreum.com | [www.moreum.com](https://www.moreum.com)
