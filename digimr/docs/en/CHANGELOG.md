# DigiMR Changelog (English)

> **Turkish version is authoritative:** [../CHANGELOG.md](../tr/CHANGELOG.md)

## v2.3.2 — 2026-08-27

> This section covers the release-asset change only; code changes between v2.2.0 and
> v2.3.2 are not yet written up here.

### The gRPC Java client SDK is back

- **`digimr-sdk-1.0.0-all.jar` ships again.** The gRPC Java client SDK removed in v2.0.1
  was rebuilt against the v2.3.2 protos and restored to the release. The jar contains only
  gRPC stubs plus the bundled grpc/netty runtime; the signing engine is **not** inside it —
  requests go to the API server (gRPC port 7702).
- **`digimr-java-examples.tar.gz`** — 10 runnable example programs (sign, verify, KEP,
  timestamp, batch sign, provider listing, level upgrade, benchmark) and a usage guide.
- **Why it came back:** v2.0.1 said a pure-Java in-process SDK would replace it, but
  `sdk/java-native` has not shipped yet, leaving Java callers without a ready client in
  the meantime.
- **Root cause fixed:** the SDK's generated gRPC stubs were checked into the repository
  and were not regenerated when the protos changed; adding the EYP endpoints left the
  module silently uncompilable. Stubs are no longer checked in — they are generated from
  the API's canonical protos on every build.

Work on `sdk/java-native` (pure-Java, in-process signing) continues and will ship in a
separate release.

---

## v2.1.0 — 2026-06-18

### İmzager conformance (all formats × both SDKs)
- **XAdES-A** valid & warning-free in İmzager on both .NET API and Java native:
  - **.NET c14n line-ending fix** — CRLF `\r` was escaped to `&#xD;`, diverging from Santuario (the core signature appeared invalid). LF normalization before c14n; content reference now uses inclusive-c14n (not raw `OuterXml`).
  - **"Unused source in revocation references" warning removed** (both SDKs) — the signature-timestamp's TSA revocation is kept only in `RevocationValues`, not in the signer's `CompleteRevocationRefs`.
  - `SigningProcessor` level mapping fixed (B_LTA→A): the API now produces XAdES-A.
- **CAdES B-LTA .NET API** conformance generator added; samples are **attached** CMS (detached fails İmzager content/archive digest checks).
- **All format × platform combinations pass İmzager**: CAdES, PAdES, XAdES-A, EYP 1.3/2.1, MA3-compat CAdES (.NET + Java native). Only exception: the EYP 2.0 seal is flagged untrusted with the software sample e-seal (expected).
- MA3-compat delegates to the native SDK, so it inherits these fixes automatically.

### Documentation
- Developer docs aligned with code: EYP method names `CreateEypPackageV20Async`/`V21Async`, gRPC `ValidateDocument`/`ValidateBatch`, timestamp verify fields, PAdES/XAdES upgrade APIs, XAdES 7 levels.

---

## v2.0.1 — 2026-06-18

- **Obfuscation**: the .NET signing engines + SDK are now shipped obfuscated (reverse-engineering resistant); the unprotected v2.0.0 was removed. Docker / Linux / Windows binaries are all obfuscated.
- **Old gRPC Java client SDK removed** (`digimr-sdk-*-all.jar`). It is being replaced by a **pure-Java in-process signing SDK** (`sdk/java-native`, BouncyCastle/PDFBox) that mirrors the .NET SDK; it will ship in a separate release. The gRPC/REST API (ports 7701/7702) is unchanged — any language, Java included, can still call the API.
  **Update: this SDK was restored in v2.3.2 — see the v2.3.2 section above.**

## v2.0.0 — 2026-04-17

### Breaking Changes

All file-path overloads for signing/validation methods have been removed. Only `byte[]` versions remain. Callers must read the file themselves:

```csharp
// v1.x (removed):
var result = await sdk.SignPdfWithProviderAsync("document.pdf", provider, parameters);

// v2.0:
var data = File.ReadAllBytes("document.pdf");
var result = await sdk.SignDataWithProviderAsync(data, provider, parameters);
```

**Removed methods:**

| Removed | Replacement |
|---|---|
| `SignPdfAsync(string path, ...)` | `SignDataWithProviderAsync(byte[] data, ...)` |
| `SignPdfWithProviderAsync(string path, ...)` | Same |
| `SignXmlAsync(string path, ...)` | `SignDataWithProviderAsync(byte[] data, ...)` |
| `SignXmlWithProviderAsync(string path, ...)` | Same |
| `ValidateDocumentAsync(string path)` | `ValidateDocumentAsync(byte[] data)` |
| `CreateEypAsync(params string[] paths)` | `CreateEypPackageV20Async / V21Async(EypCreateOptions)` |
| `sdk.GetCertificateInfo(cert)` | Use `ICertificateManager` directly |
| `sdk.ListSigners(...)` | `InspectSignaturesAsync` |
| `sdk.GetPrivateKey(...)` | Removed (security) |
| `sdk.GetMimeType(...)` | Removed (out of SDK scope) |
| `SignOoxmlAsync(...)` | `SignDataWithProviderAsync` + `Format = SignatureFormat.OOXML` |
| `VerifyOoxml(...)` | `ValidateDocumentAsync` (format auto-detected) |

### New Features

**6 new SDK methods:**
- `CheckReadinessAsync()` — Async readiness probe (license, TSA, trust store, PKCS#11 libs)
- `CheckSignatureTypeAsync(data, expectedFormat, expectedLevel?)` — Fast format/level check without full validation
- `InspectSignaturesAsync(data)` — Detailed signature inventory (signer subject, timestamp, level, role)
- `ListTsaProviders()` — Enumerate configured TSA providers
- `VerifyTokenPin(libraryPath, pin, slotId)` — Verify PIN correctness without signing
- `ListSlotCertificates(libraryPath, slotId)` — List certificates on token without PIN

**16 new Java gRPC methods** (full REST parity): `prepareSignature`, `finalizeSignature`, `listFormats`, `listValidationProfiles`, `enqueueSign`, `getSignJob`, `checkSignatureType`, `inspectSignatures`, `verifyKep`, `verifyEypDetail`, `checkReady`, `signAndCreateKep`, `signEypAndCreateKep`, `verifyKepDelil`, `batchCreateKep`, `getTokenSession`

**5 new REST endpoints:**
- `POST /verify/check`
- `POST /verify/inspect`
- `GET /timestamp/providers`
- `POST /token/pin/verify`
- `GET /token/slots/{slotId}/certificates`
- `GET /health/ready`

### Improvements

- **OOXML cross-platform:** removed `net10.0-windows` TFM; now uses BouncyCastle + manual XML-DSig on all platforms
- **Unified API surface:** 11 duplicate methods removed, single entry point (`SignDataWithProviderAsync`)
- **REST/gRPC parity:** all REST endpoints now have equivalent gRPC RPCs on port 7702
- **Test coverage:** 1587+ tests, 0 failures

### Security

- Documentation cleaned up: removed hardcoded passwords, all password examples use `Environment.GetEnvironmentVariable(...)` pattern
- Production `appsettings.json` ships with empty `Password` fields; must be set via environment variables or secrets store

### Migration Guide

1. **Replace file-path SDK calls** — see table above
2. **Update enum references:**
   - `SigningProviderType.Certificate` → `SigningProviderType.Software`
   - `SigningProviderType.Hsm` → `SigningProviderType.HSM`
3. **Update model property names:**
   - `AuthenticationContext.Pkcs12Data` → `.CertificateData`
   - `AuthenticationContext.Password` → `.CertificatePassword`
   - `ASiCDocument.Data` → `.Content`
4. **TSA factory signature changed:**
   - Old: `TsaProviderConfig.KamuSM(url, port, password)`
   - New: `TsaProviderConfig.KamuSM(url, userId, password)` *(second parameter is now int `userId`, not `port`)*

---

## Previous Versions

See Turkish [../CHANGELOG.md](../tr/CHANGELOG.md) for pre-v2.0 history.
