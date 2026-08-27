# DigiMR Documentation (English)

**SDK Version:** 2.0 | **Last Updated:** 2026-04-17 | **Status:** Stable

DigiMR is a comprehensive SDK + REST/gRPC API system for Turkey's qualified electronic signature infrastructure. This documentation targets customer integration teams.

> **Note:** Turkish documentation is the primary reference and always up-to-date. English pages are translated from Turkish; if you notice a discrepancy, the Turkish version is authoritative.

---

## Quick Start (30 seconds)

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;

var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("certificate.pfx"),
        CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
    });

var pdfData = File.ReadAllBytes("document.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_B
});

File.WriteAllBytes("document-signed.pdf", result.SignedData!);
```

Details: [Examples — Beginner](../tr/ORNEKLER.md#baslangic) *(translation pending)*

---

## Quick Navigation

| Your goal | Document |
|-------|---------|
| **Write code now** | [EXAMPLES.md](../tr/ORNEKLER.md) — 17 scenarios (C# + curl) *(translation pending)* |
| **Learn concepts** | [USER_GUIDE.md](../tr/KULLANIM_KILAVUZU.md) — Tutorial walkthrough *(translation pending)* |
| **Decision guide** | [SCENARIO_GUIDE.md](../tr/SENARYO_REHBERI.md) — Decision diagrams *(translation pending)* |
| **SDK reference** (.NET) | [SDK_REFERENCE.md](../tr/SDK_REFERANS.md) — 23 sections + 4 Appendices *(translation pending)* |
| **REST API reference** | [API_REFERENCE.md](../tr/API_REFERANS.md) — 20 sections + 3 Appendices *(translation pending)* |

> **Translation status:** Turkish documentation is complete and authoritative. English translations are in progress. For now, please use the Turkish documents in [docs/tr/](../tr/) — they are the primary source of truth. You can also refer to code examples and C# identifiers which are already in English.

---

## What's New in v2.0

**6 new SDK methods:**
- `CheckReadinessAsync()` — Is the server ready (license, TSA, trust store)?
- `CheckSignatureTypeAsync()` — Quick format/level verification
- `InspectSignaturesAsync()` — Signature inventory (certificates, timestamps, roles)
- `ListTsaProviders()` — Configured TSA provider list
- `VerifyTokenPin()` — Verify PIN without signing
- `ListSlotCertificates()` — Token certificate enumeration

**5 new REST endpoints:**
- `POST /verify/check`, `POST /verify/inspect`
- `GET /timestamp/providers`
- `POST /token/pin/verify`, `GET /token/slots/{id}/certificates`
- `GET /health/ready`

**Breaking changes (migration required):**
- String-path overloads removed — only `byte[]` versions remain
- Full list: [SDK_REFERENCE.md Appendix D](../tr/SDK_REFERANS.md) *(Turkish: [SDK_REFERANS.md](../tr/SDK_REFERANS.md))*
- Migration guide: [CHANGELOG.md](CHANGELOG.md)

---

## Supported Formats and Levels

| Format | Standard | Supported Levels |
|--------|----------|-----------|
| **PAdES** | ETSI EN 319 142 | B-B, B-T, B-LT, B-LTA |
| **CAdES** | ETSI EN 319 122 | B-B, B-T, B-C, B-X, B-LT, B-LTA |
| **XAdES** | ETSI EN 319 132 | B-B, B-T, B-LT, B-LTA |
| **JAdES** | ETSI TS 119 182 | B-B, B-T |
| **ASiC-S / ASiC-E** | ETSI EN 319 162 | B-T, B-LT, B-LTA |
| **OOXML** | ISO/IEC 29500 | B-T |
| **EYP** | Turkish CBDDO V2.0 / V1.3 | OPC + CAdES (seal: B-LTA) |
| **KEP** | Turkish Law #7201 | CAdES-T + ZIP |
| **UETS** | PTT UETS | CAdES-T + ZIP |

---

## Architecture

Two usage models:

```
In-process SDK (local signing)          API server (remote / other languages)
  ├─ .NET SDK  (DigitalSignature.SDK)     ├─ REST API  (:7701)
  └─ Java SDK  (pure Java, java-native)   └─ gRPC API  (:7702)
```

- **.NET SDK**: in-process core library (`IDigitalSignatureSDK`) — produces signatures locally.
- **Java SDK**: in-process **pure-Java** library (`sdk/java-native`, BouncyCastle/PDFBox) — same model as the .NET SDK, signs locally; module services (CadesSigner, XAdesSigner, PadesSigner, EYP, …). _The pure-Java SDK release is in preparation._
- **Java gRPC client**: thin remote client for the API (`digimr-sdk-1.0.0-all.jar`) — signing happens on the server. Shipping again as of v2.3.1; the ready-made way to reach the API from Java until `sdk/java-native` lands.
- **REST API**: HTTP/JSON API (port 7701) — HTTP reflection of all SDK methods (requires the server).
- **gRPC API**: Binary protocol (port 7702) — high performance, streaming (requires the server).

Details: [ARCHITECTURE.md](../tr/ARCHITECTURE.md) (diagrams)

---

## Support

- **GitHub Issues** (customer repo): `moreum-tech/MBox`
- **Email**: `info@moreum.com`
- **Docs bundle**: MBox release assets include `digimr-docs.tar.gz`

---

## Translation Contribution

If you would like to contribute English translations of the remaining documents ([USER_GUIDE.md](../tr/KULLANIM_KILAVUZU.md), [SDK_REFERENCE.md](../tr/SDK_REFERANS.md), etc.), please contact `info@moreum.com`. All translations must be reviewed against the authoritative Turkish source.
