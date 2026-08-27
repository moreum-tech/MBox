# DigiMR REST API Referansi

**API Versiyon:** v1
**Son Guncelleme:** 2026-04-17
**Durum:** Stabil
**Base URL:** `http://localhost:7701/api/v1`
**gRPC Karsiligi:** Port 7702 (bkz. Appendix C)

---

## Icindekiler

1. Genel Bakis ve Authentication
2. Endpoint Ozet Tablosu
3. Imzalama Endpoint'leri
4. Dogrulama Endpoint'leri
5. Seviye Yukseltme
6. Zaman Damgasi
7. EYP
8. KEP
9. UETS
10. ASiC
11. JAdES
12. PDF Encryption
13. Token / PKCS#11 Yonetimi
14. Oturum Imzalama (SigningSession)
15. Provider Bilgi
16. Preservation
17. Export
18. Health
19. Ek Controller'lar (Admin, Log, Config, Setup, CRL, TrustedList, CSC, EUDI, Biometric)

## Appendix

- A: HTTP Hata Kodlari
- B: Ortak Request/Response Modelleri
- C: gRPC Yansimasi (port 7702)

---

## 1. Genel Bakis ve Authentication

DigiMR REST API, `IDigitalSignatureSDK` .NET interface'inin HTTP yansimasidir. Tum endpoint'ler JSON request/response kullanir; ikili veriler (PDF, sertifika bytes vb.) base64 string olarak kodlanir.

### Base URL

- Development: `http://localhost:7701/api/v1`
- Docker/Production: Reverse proxy arkasinda (HTTPS) — `https://<host>/api/v1`

### Authentication

API-level authentication orta katmanda (gateway/reverse proxy) yapilir. SDK kendi icinde:
- Lisans kontrolu (30 gun deneme modu disinda `.lic` dosyasi gerekli)
- Token PIN / PKCS#12 parolasi request body'de gelir (TLS ile korunmali)
- Agent API key header'da (`X-Agent-API-Key`)

### gRPC Alternatifi

Tum REST endpoint'lerinin gRPC karsiligi port **7702**'de calisir. gRPC tercih durumlari:
- Yuksek performans (protobuf + HTTP/2)
- Buyuk belge transferleri
- Streaming (batch, progress)

Detay: Appendix C.

---

## 2. Endpoint Ozet Tablosu

Asagidaki tablo tum REST endpoint'lerini gruplara ayrilmis halde listeler.
`[YENI 2.0]` etiketi kaynak kodda dogrulanmis, ancak bu surumde eklenen endpoint'leri isaretler.
`[YENI 2.0 — planlaniyor]` etiketi henuz controller'da bulunmayan, planlanan endpoint'leri isaretler.

**Not:** Tablo kaynaktaki controller'lara gore uretilmistir. Path farkliligi varsa kaynak dogrudur, bu dokuman guncellenmeli.

| Grup | HTTP | Path | SDK / Servis Karsiligi | Bolum |
|---|---|---|---|---|
| Imzalama | POST | `/sign` | `SignDataWithProviderAsync` | 3.1 |
| Imzalama | POST | `/sign/batch` | `SignBatchAsync` | 3.2 |
| Imzalama | POST | `/sign/enqueue` | `SigningQueueService` (API katmani) | 3.3 |
| Imzalama | GET | `/sign/jobs/{jobId}` | `IAsyncSignJobStore` (API katmani) | 3.4 |
| Imzalama | GET | `/sign/formats` | Statik enum listesi | 3.5 |
| Imzalama | POST | `/sign/prepare` | `PrepareForExternalSign` | 3.6 |
| Imzalama | POST | `/sign/finalize` | `FinalizeExternalSign` | 3.7 |
| Dogrulama | GET | `/verify/profiles` | Statik profil listesi | 4.1 |
| Dogrulama | POST | `/verify` | `ValidateDocumentAsync` | 4.2 |
| Dogrulama | POST | `/verify/batch` | `ValidateBatchAsync` | 4.3 |
| Dogrulama | POST | `/verify/check` [YENI 2.0] | `CheckSignatureTypeAsync` | 4.4 |
| Dogrulama | POST | `/verify/inspect` [YENI 2.0] | `InspectSignaturesAsync` | 4.5 |
| Dogrulama | POST | `/verify/kep` | `VerifyKepPackage` | 4.6 |
| Dogrulama | POST | `/verify/eyp` | `VerifyEyp` | 4.7 |
| Seviye | POST | `/upgrade` | `UpgradeSignatureAsync` | 5.1 |
| Zaman Damgasi | POST | `/timestamp/token` | `GetTimestampTokenAsync` | 6.1 |
| Zaman Damgasi | POST | `/timestamp/verify` | `VerifyTimestamp` | 6.2 |
| Zaman Damgasi | GET | `/timestamp/providers` [YENI 2.0] | `ListTsaProviders` | 6.3 |
| EYP | POST | `/eyp/create` | `CreateEypPackageV20Async/V21Async` (Version) | 7.1 |
| EYP | POST | `/eyp/create-v13` | `CreateEypPackageV13Async` | 7.2 |
| EYP | POST | `/eyp/verify` | `VerifyEyp` | 7.3 |
| EYP | POST | `/eyp/extract` | `ExtractEyp` | 7.4 |
| EYP | POST | `/eyp/sign-and-create` | `SignAndCreateEypAsync` | 7.5 |
| EYP | POST | `/eyp/encrypt` | `EncryptEypPackage` | 7.6 |
| EYP | POST | `/eyp/decrypt` | `DecryptEypPackage` | 7.7 |
| EYP | POST | `/eyp/guncelleme/create` | `CreateEypGuncellemePaketiAsync` | 7.8 |
| EYP | POST | `/eyp/guncelleme/verify` | `VerifyEypGuncellemePaketi` | 7.9 |
| KEP | POST | `/kep/create` | `CreateKepPackageAsync` | 8.1 |
| KEP | POST | `/kep/verify` | `VerifyKepPackage` | 8.2 |
| KEP | POST | `/kep/extract` | `ExtractKepPackage` | 8.3 |
| KEP | POST | `/kep/sign-and-create` | `SignAndCreateKepAsync` | 8.4 |
| KEP | POST | `/kep/sign-eyp-and-create` | `SignEypAndCreateKepAsync` | 8.5 |
| KEP | POST | `/kep/verify-delil` | `VerifyKepDelil` | 8.6 |
| KEP | POST | `/kep/batch-create` | `BatchCreateKepAsync` | 8.7 |
| UETS | POST | `/uets/create` | `CreateUetsPackageAsync` | 9.1 |
| UETS | POST | `/uets/verify` | `VerifyUetsPackage` | 9.2 |
| UETS | POST | `/uets/extract` | `ExtractUetsPackage` | 9.3 |
| ASiC | POST | `/asic/create-s` | `CreateAsicSAsync` | 10.1 |
| ASiC | POST | `/asic/create-e` | `CreateAsicEAsync` | 10.2 |
| ASiC | POST | `/asic/verify` | `VerifyAsic` | 10.3 |
| ASiC | POST | `/asic/extract` | `ExtractAsic` | 10.4 |
| JAdES | POST | `/jades/sign` | `SignJadesAsync` | 11.1 |
| JAdES | POST | `/jades/verify` | `VerifyJades` | 11.2 |
| JAdES | POST | `/jades/upgrade` | `UpgradeJadesAsync` | 11.3 |
| PDF | POST | `/pdf/encrypt` | `EncryptPdf` | 12.1 |
| PDF | POST | `/pdf/decrypt` | `DecryptPdf` | 12.2 |
| PDF | POST | `/pdf/is-encrypted` | `IsPdfEncrypted` | 12.3 |
| Token | GET | `/token/slots` | `ListTokens` (PKCS#11 slot bilgisi) | 13.1 |
| Token | GET | `/token/slots/{slotId}/certificates` | `ListSlotCertificates` | 13.2 |
| Token | POST | `/token/sessions` | `CreateTokenSessionAsync` | 13.3 |
| Token | GET | `/token/sessions` | `ListTokenSessions` | 13.4 |
| Token | GET | `/token/sessions/{sessionId}` | `GetTokenSession` | 13.5 |
| Token | GET | `/token/sessions/{sessionId}/certificates` | `ListSessionCertificates` | 13.6 |
| Token | DELETE | `/token/sessions/{sessionId}` | `CloseTokenSessionAsync` | 13.7 |
| Token | POST | `/token/pin/verify` [YENI 2.0] | `VerifyTokenPin` | 13.8 |
| Token | GET | `/token/pin/status` | `GetTokenPinStatus` | 13.9 |
| Token | GET | `/token/status` | Token baglanti durumu | 13.10 |
| SigningSession | POST | `/signing-sessions` | Oturum olustur | 14.1 |
| SigningSession | GET | `/signing-sessions/{sessionId}` | Oturum getir | 14.2 |
| SigningSession | GET | `/signing-sessions/{sessionId}/documents/{docIndex}` | Belge getir | 14.3 |
| SigningSession | GET | `/signing-sessions/{sessionId}/documents/{docIndex}/signed` | Imzali belge getir | 14.4 |
| SigningSession | PATCH | `/signing-sessions/{sessionId}/documents/{docIndex}/placement` | Imza konumu guncelle | 14.5 |
| SigningSession | POST | `/signing-sessions/{sessionId}/otp/send` | OTP gonder | 14.6 |
| SigningSession | POST | `/signing-sessions/{sessionId}/otp/verify` | OTP dogrula | 14.7 |
| SigningSession | POST | `/signing-sessions/{sessionId}/token-sign` | Token ile imzala | 14.8 |
| SigningSession | POST | `/signing-sessions/{sessionId}/mobile-sign` | Mobil imza | 14.9 |
| SigningSession | POST | `/signing-sessions/{sessionId}/handwritten-sign` | El yazisi imza | 14.10 |
| SigningSession | POST | `/signing-sessions/{sessionId}/nfc-verified` | NFC dogrulama bildir | 14.11 |
| SigningSession | POST | `/signing-sessions/{sessionId}/send-callback` | Callback gonder | 14.12 |
| SigningSession | POST | `/signing-sessions/{sessionId}/send-to-url` | URL'ye gonder | 14.13 |
| SigningSession | GET | `/signing-sessions/agent/tokens` | Agent token listesi | 14.14 |
| SigningSession | POST | `/signing-sessions/agent/session` | Agent oturumu olustur | 14.15 |
| SigningSession | GET | `/signing-sessions/agent/tokens/{slotId}/certificates` | Agent slot sertifikalari | 14.16 |
| SigningSession | GET | `/signing-sessions/agent/pin-status` | Agent PIN durumu | 14.17 |
| SigningSession | DELETE | `/signing-sessions/agent/session/{agentSessionId}` | Agent oturumu kapat | 14.18 |
| Provider | GET | `/providers` | `GetProviders` (desteklenen tipler) | 15.1 |
| Provider | GET | `/providers/tokens` | `ListTokens` (lokal) | 15.2 |
| Provider | POST | `/providers/remote/tokens` | `QueryRemoteTokensAsync` | 15.3 |
| Provider | POST | `/providers/authenticate` [YENI 2.0 — planlaniyor] | `CreateAndAuthenticateProviderAsync` | 15.4 |
| Provider | POST | `/providers/verify-pin` [YENI 2.0 — planlaniyor] | `VerifyTokenPin` (bkz. `/token/pin/verify`) | 15.5 |
| Provider | POST | `/providers/slot-certs` [YENI 2.0 — planlaniyor] | `ListSlotCertificates` (bkz. `/token/slots/{id}/certificates`) | 15.6 |
| Preservation | POST | `/preservation/check` | `CheckPreservationStatus` | 16.1 |
| Preservation | POST | `/preservation/renew` | `RenewArchiveTimestampAsync` | 16.2 |
| Preservation | POST | `/preservation/evidence` | `GetEvidenceRecord` | 16.3 |
| Export | POST | `/export` | `ExportSignature` | 17.1 |
| Health | GET | `/health` | `CheckHealth` | 18.1 |
| Health | GET | `/health/ready` | `CheckReadinessAsync` | 18.2 |

### Planlanan 2.0 Endpoint'leri (henuz kaynakta yok)

Asagidaki endpoint'ler planlanan API 2.0 revizyon kapsamindadir. Karsilik gelen islevler mevcut controller'larda farkli path altinda calisir.

| Grup | HTTP | Path | Not |
|---|---|---|---|
| Provider | POST | `/providers/authenticate` | Simdilik icsel (`CreateAndAuthenticateProviderAsync`) |
| Provider | POST | `/providers/verify-pin` | Simdilik `/token/pin/verify` |
| Provider | POST | `/providers/slot-certs` | Simdilik `/token/slots/{slotId}/certificates` |

### Ek Controller'lar (19. Bolum Ozeti)

| Grup | HTTP | Path |
|---|---|---|
| Admin | POST | `/admin/shutdown` |
| Admin | GET | `/admin/agents` |
| Admin | GET | `/admin/agents/{agentId}` |
| Admin | POST | `/admin/agents/{agentId}/sign-hash` |
| Admin | GET | `/admin/tsa-credit` |
| Log | GET | `/logs` |
| Log | GET | `/logs/range` |
| Log | DELETE | `/logs` |
| Log | GET | `/logs/daily` |
| Log | GET | `/logs/auth/status` |
| Log | POST | `/logs/auth/setup` |
| Log | POST | `/logs/auth/login` |
| Log | POST | `/logs/auth/logout` |
| Config | GET | `/config/sign-app` |
| Setup | GET | `/setup/status` |
| Setup | GET | `/setup/test` |
| Setup | POST | `/setup/save` |
| Setup | GET | `/setup/machine-id` |
| Setup | POST | `/setup/activate` |
| Setup | GET | `/setup/biometric/status` |
| Setup | POST | `/setup/biometric/generate-keys` |
| CRL | GET | `/crl/status` |
| CRL | POST | `/crl/refresh` |
| CRL | POST | `/crl/refresh/url` |
| CRL | POST | `/crl/urls` |
| CRL | GET | `/crl/urls` |
| TrustedList | POST | `/trustedlist/load` |
| TrustedList | POST | `/trustedlist/check` |
| TrustedList | GET | `/trustedlist/providers` |
| TrustedList | GET | `/trustedlist/status` |
| CSC | POST | `/csc/info` |
| CSC | POST | `/csc/credentials/list` |
| CSC | POST | `/csc/credentials/info` |
| CSC | POST | `/csc/credentials/authorize` |
| CSC | POST | `/csc/signatures/signHash` |
| EUDI | POST | `/eudi/request-signing` |
| EUDI | POST | `/eudi/callback` |
| EUDI | GET | `/eudi/status/{requestId}` |
| Biometric | POST | `/biometric/generate-keys` |
| Biometric | POST | `/biometric/generate-keys/force` |
| Biometric | GET | `/biometric/status` |
| Biometric | POST | `/biometric/extract` |

---

## 3. Imzalama Endpoint'leri

Imzalama islemleri `/api/v1/sign` altinda toplanir. SDK'nin `SignDataWithProviderAsync` ve `SignBatchAsync` metotlarinin HTTP yansimasi.

### 3.1 POST /sign

Tek belge imzalama. Format `PAdES`, `CAdES`, `XAdES`, `OOXML` veya `JAdES` olabilir; belirtilmezse format icerige gore otomatik secilir.

**Request:**
```json
{
    "documentBase64": "<base64 belge>",
    "format": "PAdES",
    "parameters": {
        "level": "B_T",
        "tsaUrl": "http://tzd.kamusm.gov.tr",
        "hashAlgorithm": "SHA256",
        "reason": "Onay",
        "location": "Ankara"
    },
    "provider": {
        "type": "token",
        "pin": "******",
        "pkcs11LibraryPath": "C:\\Windows\\System32\\eTPKCS11.dll",
        "slotId": 0
    }
}
```

**Response 200:**
```json
{
    "success": true,
    "signedDocumentBase64": "<base64 imzali belge>",
    "message": "Signing successful"
}
```

**Response 400 (gecersiz parametre):**
```json
{ "error": "TsaUrl is required for signature level B_T" }
```

**curl:**
```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d @sign-request.json
```

**SDK Karsilgi:** `sdk.SignDataWithProviderAsync` / `SigningProcessor.ExecuteAsync` (Bolum 6.1 SDK_REFERANS)

**Hatalar:** 400 (gecersiz data/params), 500 (signer/TSA hatasi)

### 3.2 POST /sign/batch

Toplu imzalama — ayni provider ile coklu belge. PIN yalnizca bir kez kullanilir.

**Request:**
```json
{
    "documents": [
        { "id": "doc1", "documentBase64": "<base64>" },
        { "id": "doc2", "documentBase64": "<base64>" }
    ],
    "format": "PAdES",
    "parameters": { "level": "B_B" },
    "provider": { "type": "token", "pin": "******" }
}
```

**Response 200:**
```json
{
    "results": [
        { "id": "doc1", "success": true, "signedDocumentBase64": "<base64>" },
        { "id": "doc2", "success": false, "error": "Invalid PDF" }
    ],
    "summary": {
        "total": 2,
        "succeeded": 1,
        "failed": 1,
        "elapsedMs": 1234.5
    }
}
```

**SDK Karsilgi:** `sdk.SignBatchAsync(batchItems, provider, parameters)` (Bolum 6.2)

### 3.3 POST /sign/enqueue

Async kuyruk tabanli imzalama. Uzun suren islemler icin; aninda `jobId` doner, is arka planda islenir.

**Request:** `SignRequest` (3.1 ile ayni)

**Response 202:**
```json
{
    "jobId": "a1b2c3...",
    "status": "Pending",
    "pollUrl": "/api/v1/sign/jobs/a1b2c3...",
    "message": "Is kuyruğa eklendi. Durum icin pollUrl'i sorgulayın."
}
```

### 3.4 GET /sign/jobs/{jobId}

Async is durumunu sorgular. Is tamamlandiginda sonucu dondurur ve store'dan siler (one-time read).

**Response 200:**
```json
{
    "jobId": "a1b2c3...",
    "status": "Completed",
    "createdAt": "2026-04-17T10:00:00Z",
    "startedAt": "2026-04-17T10:00:01Z",
    "completedAt": "2026-04-17T10:00:02Z",
    "signedDocumentBase64": "<base64 imzali belge>",
    "error": null
}
```

`status` degerleri: `Pending`, `Running`, `Completed`, `Failed`

### 3.5 GET /sign/formats

Desteklenen format + level kombinasyonlarini, gerekli parametreleri ve yasal notlari listeler.

**Response 200 (kisaltilmis):**
```json
{
    "supportedFormats": [
        {
            "format": "PAdES",
            "levels": [
                { "level": "B_B", "requiresTsa": false },
                { "level": "B_T", "requiresTsa": true },
                { "level": "B_LT", "requiresTsa": true },
                { "level": "B_LTA", "requiresTsa": true }
            ],
            "requiredParameters": ["documentBase64", "provider"],
            "legalNote": "5070 sayili Elektronik Imza Kanunu kapsaminda gecerli"
        }
    ],
    "hashAlgorithms": ["SHA256", "SHA384", "SHA512"],
    "defaultHashAlgorithm": "SHA256"
}
```

### 3.6 POST /sign/prepare

Iki asamali imzalamanin birinci asamasi. External signer (HSM, remote agent) kullanimi icin hash hazirlar.

**Akis:**
1. `GET /api/v1/token/slots/{slotId}/certificates` ile sertifika al
2. `POST /sign/prepare` ile hash hazirla → `{ jobId, hashBase64, expiresIn: 300 }`
3. Agent ile hash'i imzala → `{ signatureBase64 }`
4. `POST /sign/finalize` ile belgeye entegre et

**Request:**
```json
{
    "documentBase64": "<base64 belge>",
    "format": "PAdES",
    "parameters": { "level": "B_B" },
    "certBase64": "<base64 DER sertifika>",
    "chainBase64": ["<base64 DER ara sertifika>"]
}
```

**Response 200:**
```json
{
    "jobId": "a1b2c3...",
    "hashBase64": "<base64 hash-to-sign>",
    "hashAlgorithm": "SHA256",
    "certBase64": "<sertifika tekrari>",
    "expiresIn": 300,
    "format": "PAdES",
    "message": "Hash hazır. Token ile imzalayın, ardından /finalize çağırın."
}
```

**SDK Karsilgi:** `sdk.PrepareSignatureAsync(documentData, parameters, certBytes, chainBytes)` (Bolum 20.1)

### 3.7 POST /sign/finalize

External signer'dan alinan raw imzayi belgeye entegre eder.

**Request:**
```json
{
    "jobId": "a1b2c3...",
    "rawSignatureBase64": "<base64 raw imza (DER veya PKCS#1)>"
}
```

**Response 200:**
```json
{
    "success": true,
    "signedDocumentBase64": "<base64 imzali belge>",
    "message": null
}
```

**SDK Karsilgi:** `sdk.FinalizeSignatureAsync(prepareResult, rawSignatureBytes)` (Bolum 20.2)

---

## 4. Dogrulama Endpoint'leri

Dogrulama islemleri `/api/v1/verify` altinda toplanir. Format otomatik tespit edilir (PAdES/CAdES/XAdES/EYP/ASiC).

### 4.1 GET /verify/profiles

Yapilandirilmis dogrulama profillerini listeler. Profiller `appsettings.json` icinde tanimlanir.

**Response 200:**
```json
{
    "profiles": [
        {
            "key": "standard",
            "displayName": "Standart",
            "description": "Genel amaçli dogrulama",
            "minLevel": "B_B",
            "allowedFormats": ["PAdES", "CAdES", "XAdES"],
            "revocationCheck": true,
            "gracePeriodHours": 24
        }
    ]
}
```

### 4.2 POST /verify

Tek belge dogrulama. Format otomatik algilir.

**Request:**
```json
{
    "documentBase64": "<base64 imzali belge>"
}
```

MA3 uyumlulugu: `BaseVerifyRequest` alanlarini da kabul eder (`trustSigningTimeAttribute`, `initialCertificatesBase64`, `initialCrlsBase64`, `initialOcspResponsesBase64`, `gracePeriodSeconds`, `profile`). MA3 alias'lari: `P_TRUST_SIGNINGTIMEATTR`, `P_INITIAL_CERTIFICATES`, `P_INITIAL_CRLS`, `P_INITIAL_OCSP`, `P_GRACE_PERIOD`, `P_VALIDATION_POLICY`.

**Response 200:**
```json
{
    "isValid": true,
    "format": "PAdES",
    "isDocumentIntact": true,
    "signatures": [
        {
            "signerSubject": "CN=Test, O=Org, C=TR",
            "certificateSerial": "12:34:56",
            "signatureTime": "2026-04-17T10:30:00Z",
            "isValid": true,
            "level": "B_T",
            "role": "Document",
            "hashAlgorithm": "SHA256",
            "errors": [],
            "metadata": {}
        }
    ],
    "validationState": "MATURE",
    "isPremature": false,
    "errors": [],
    "warnings": []
}
```

`validationState` degerleri: `"PREMATURE"` | `"MATURE"` | `null`

**SDK Karsilgi:** `sdk.ValidateDocumentAsync(documentData)` (Bolum 7.1)

### 4.3 POST /verify/batch

Coklu belge dogrulama. Max belge sayisi `Security:MaxBatchDocuments` (varsayilan 100).

**Request:**
```json
{
    "documents": [
        { "id": "doc1", "documentBase64": "<base64>" },
        { "id": "doc2", "documentBase64": "<base64>" }
    ]
}
```

**Response 200:**
```json
{
    "results": [
        { "id": "doc1", "isValid": true, "format": "PAdES", "signatureCount": 1, "errors": [] },
        { "id": "doc2", "isValid": false, "format": "Unknown", "signatureCount": 0, "errors": ["..."] }
    ],
    "total": 2
}
```

**SDK Karsilgi:** `sdk.ValidateDocumentAsync` (her belge icin ayri) (Bolum 7.2)

### 4.4 POST /verify/check  [YENI 2.0]

Belgede belirli format/level'da imza var mi kontrol eder. Profil kurallarini da uygular. Kaynakta mevcut (`VerifyController.cs`, `[HttpPost("check")]`).

**Request:**
```json
{
    "documentBase64": "<base64 imzali belge>",
    "format": "PAdES",
    "level": "B_T"
}
```

`format` zorunlu: `PAdES`, `CAdES`, `XAdES`, `EYP`. `level` opsiyonel: `B_B`, `B_T`, `B_LT`, `B_LTA`.

**Response 200:**
```json
{
    "hasMatch": true,
    "detectedFormat": "PAdES",
    "targetFormat": "PAdES",
    "targetLevel": "B_T",
    "appliedProfile": "Standart",
    "matchReason": "1 adet PAdES B_T imzasi bulundu",
    "matchedSignatures": [
        {
            "index": 0,
            "signerSubject": "CN=Test, O=Org, C=TR",
            "signatureTime": "2026-04-17T10:30:00Z",
            "level": "B_T",
            "isValid": true
        }
    ],
    "deficiencies": []
}
```

**SDK Karsilgi:** `sdk.ValidateDocumentAsync` + format/level filtresi (Bolum 7.3)

### 4.5 POST /verify/inspect  [YENI 2.0]

Belgede bulunan tum imzalarin envanterini cikarir; eksiklikleri ve yukseltme onerilerini raporlar. Kaynakta mevcut (`VerifyController.cs`, `[HttpPost("inspect")]`).

**Request:**
```json
{
    "documentBase64": "<base64 imzali belge>"
}
```

**Response 200:**
```json
{
    "detectedFormat": "PAdES",
    "isDocumentIntact": true,
    "validationState": "MATURE",
    "isPremature": false,
    "signatureCount": 2,
    "appliedProfile": "Standart",
    "profileViolations": [],
    "signatureInventory": [
        {
            "index": 0,
            "role": "Document",
            "format": "PAdES",
            "level": "B_T",
            "signatureType": "PAdES-B-T",
            "isValid": true,
            "signerSubject": "CN=Signer1",
            "certificateSerial": "12:34:56",
            "signatureTime": "2026-04-17T10:30:00Z",
            "errors": [],
            "metadata": {}
        }
    ],
    "validSignatureTypes": ["PAdES-B-T"],
    "deficiencyReport": [
        {
            "signatureIndex": 1,
            "signatureType": "PAdES-B-B",
            "deficiencies": ["Imza zaman damgasi (RFC 3161) eksik — B-T seviyesi gerektirir"],
            "recommendation": "Zaman damgasi ekleyerek PAdES-B-T seviyesine yukseltilebilir"
        }
    ],
    "packageNote": null,
    "overallErrors": [],
    "warnings": [],
    "summary": "2 imza/muhur bulundu. Gecerli imza turleri: PAdES-B-T | 1 imzada eksiklik"
}
```

**SDK Karsilgi:** `sdk.ValidateDocumentAsync` + envanter analizi (Bolum 7.4)

### 4.6 POST /verify/kep

KEP paketini ayrintili dogrular: ZIP yapisi, kep-bilgi.xml, CAdES imza ve RFC 3161 zaman damgasi.

**Request:**
```json
{ "kepPackageBase64": "<base64 KEP ZIP>" }
```

**Response 200:**
```json
{
    "isValid": true,
    "isSignatureValid": true,
    "isTimestampValid": true,
    "hasRequiredFields": true,
    "signerSubject": "CN=KEPHS",
    "signingTime": "2026-04-17T10:30:00Z",
    "timestampTime": "2026-04-17T10:30:01Z",
    "senderAddress": "gonderen@kep.gov.tr",
    "recipientAddresses": ["alici@kep.gov.tr"],
    "subject": "Resmi Yazisma",
    "attachmentCount": 2,
    "deficiencies": [],
    "errors": [],
    "warnings": []
}
```

### 4.7 POST /verify/eyp

EYP paketini ayrintili dogrular: OPC yapisi, PaketOzeti hash'leri, CAdES imza ve E-Muhur.

**Request:**
```json
{ "eypPackageBase64": "<base64 EYP ZIP>" }
```

**Response 200:**
```json
{
    "isValid": true,
    "isPackageStructureValid": true,
    "isPaketOzetiValid": true,
    "isNihaiOzetValid": true,
    "signatureInfo": { "isValid": true, "signerSubject": "CN=Imzaci", "level": "B_LT" },
    "sealInfo": { "isValid": true, "sealSubject": "CN=Muhur", "level": "B_LTA" },
    "deficiencies": [],
    "errors": [],
    "warnings": []
}
```

---

## 5. Seviye Yukseltme

Seviye yukseltme islemleri mevcut imzali belgelerin seviyesini artirmak icin kullanilir (B-B → B-T → B-LT → B-LTA). Format otomatik algilir.

### 5.1 POST /upgrade

CAdES/PAdES/XAdES imzalarini daha yuksek seviyeye cikarir. B-T ve uzeri seviyelerde TSA URL gereklidir.

**Request:**
```json
{
    "documentBase64": "<base64 imzali belge>",
    "targetLevel": "B-T",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
}
```

`targetLevel` kabul edilen degerler: `B-T`, `B-C`, `B-X`, `B-LT`, `B-LTA` (tire veya alt cizgi ile).

**Response 200:**
```json
{
    "success": true,
    "upgradedDocumentBase64": "<base64 yukseltilmis belge>",
    "detectedFormat": "PAdES",
    "previousLevel": "B_B",
    "newLevel": "B_T",
    "sizeBytes": 45678
}
```

**Response 400 (basarisiz):**
```json
{
    "success": false,
    "error": "TsaUrl is required for B-T, B-X, B-LT, B-LTA levels",
    "detectedFormat": "PAdES"
}
```

**SDK Karsilgi:** `sdk.UpgradeSignatureAsync(documentData, targetLevel, tsaUrl)` (Bolum 8.1)

**Not:** JAdES yukseltme icin bkz. Bolum 11.3 (`POST /jades/upgrade`).

### 5.2 POST /jades/upgrade

JAdES icin ayri endpoint. Compact veya JSON Serialization formatindaki JWS'yi yukseltir.

**Request:**
```json
{
    "jadesData": "<jws compact string veya JSON>",
    "targetLevel": "B_T",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
}
```

`targetLevel` B_T veya uzeri olmalidir.

**Response 200:**
```json
{
    "success": true,
    "jadesData": "<yukseltilmis jws>",
    "level": "B_T",
    "sizeBytes": 2048
}
```

**SDK Karsilgi:** `sdk.UpgradeJadesAsync(jadesData, targetLevel, tsaUrl)` (Bolum 8.2)

**Path:** `/jades/upgrade` (NOT `/upgrade/jades`) — ozet tablodaki yonlendirme Bolum 11.3'e isaret eder.

---

## 6. Zaman Damgasi Endpoint'leri

RFC 3161 zaman damgasi token alma ve dogrulama. Birden fazla TSA saglayicisi desteklenir (`appsettings.json → TsaProviders`).

### 6.1 POST /timestamp/token

RFC 3161 zaman damgasi token'i alir. Ham veri (DataBase64) veya onceden hesaplanmis hash (HashBase64) gonderin.

**Request (ham veri ile):**
```json
{
  "dataBase64": "<base64 belge/veri>",
  "hashAlgorithm": "SHA256",
  "provider": "KAMUSM"
}
```

**Request (hash ile):**
```json
{
  "hashBase64": "<base64 SHA256 hash, 32 byte>",
  "hashAlgorithm": "SHA256",
  "provider": "KAMUSM"
}
```

Alan aciklamalari:
- `dataBase64` — Zaman damgalanacak ham veri (base64). `hashBase64` ile kullanilmaz.
- `hashBase64` — Onceden hesaplanmis hash (base64). `dataBase64` yerine gecebilir.
- `hashAlgorithm` — `SHA256` (varsayilan), `SHA384`, `SHA512`
- `provider` — TSA saglayici anahtari (`TsaProviders` icindeki key). Belirtilmezse varsayilan saglayici kullanilir.

**Response 200:**
```json
{
  "success": true,
  "timestampTokenBase64": "<base64 RFC-3161 TSTInfo DER>",
  "provider": "KAMUSM",
  "providerName": "Kamu Sertifikasyon Merkezi",
  "hashAlgorithm": "SHA256",
  "tokenSizeBytes": 2048,
  "genTime": "2026-04-17T10:30:00Z",
  "serialNumber": "1234567890"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/timestamp/token \
  -H "Content-Type: application/json" \
  -d '{"dataBase64":"SGVsbG8gV29ybGQ=","hashAlgorithm":"SHA256"}' | jq .
```

**SDK Karsiligi:** `sdk.GetTimestampTokenAsync(data, providerKey)` ve `sdk.GetTimestampTokenFromHashAsync(hash, algorithm, providerKey)` (SDK_REFERANS Bolum 9.1)

---

### 6.2 POST /timestamp/verify

RFC 3161 timestamp token'ini dogrular. Token'in icindeki hash'in verilen veriyle eslesip eslesmedigini kontrol eder.

**Request:**
```json
{
  "timestampTokenBase64": "<base64 RFC-3161 token>",
  "dataBase64": "<base64 orijinal veri>"
}
```

Alan aciklamalari:
- `timestampTokenBase64` — Dogrulanacak RFC 3161 token (zorunlu)
- `dataBase64` — Orijinal veri. Hash karsilastirmasi icin gerekli. `hashBase64` ile alternatif olarak kullanilabilir.
- `hashBase64` — Onceden hesaplanmis hash (dataBase64 yerine)

**Response 200:**
```json
{
  "isValid": true,
  "genTime": "2026-04-17T10:30:00Z",
  "serialNumber": "1234567890",
  "hashAlgorithm": "SHA256",
  "hashMatches": true
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/timestamp/verify \
  -H "Content-Type: application/json" \
  -d '{"timestampTokenBase64":"<token>","dataBase64":"<veri>"}' | jq .
```

**SDK Karsiligi:** `sdk.VerifyTimestamp(tokenBytes, originalData)` (SDK_REFERANS Bolum 9.2)

---

### 6.3 GET /timestamp/providers  [YENI 2.0]

Yapilandirilmis ve etkin TSA saglayicilarinin listesini doner. Kimlik bilgileri (sifre, userId) asla dondurmez.

**Request:** HTTP GET, body yok.

**Response 200:**
```json
{
  "defaultProvider": "KAMUSM",
  "providers": [
    {
      "key": "KAMUSM",
      "name": "Kamu Sertifikasyon Merkezi",
      "url": "http://zd.kamusm.gov.tr",
      "authType": "UsernamePassword",
      "isDefault": true
    },
    {
      "key": "TURKTRUST",
      "name": "TURKTRUST",
      "url": "http://zd.turktrust.com.tr",
      "authType": "None",
      "isDefault": false
    }
  ]
}
```

**curl:**
```bash
curl -s http://localhost:7701/api/v1/timestamp/providers | jq .
```

**SDK Karsiligi:** Dogrudan `IConfiguration["TsaProviders"]` okur; SDK metodu yoktur. TSA listesi `appsettings.json → TsaProviders` bolumunde yapilandirilir.

---

## 7. EYP Endpoint'leri

EYP (Elektronik Yazisma Paketi) — CBDDO EYP 2.0/2.1 ve 1.3 standartlari. OPC/ZIP tabanli yapiyi imzali belge + muhur + metadata ile olusturur.

### 7.1 POST /eyp/create

EYP 2.0/2.1 paketi olusturur. OPC yapisinda: ust yazi + ekler + CAdES imza + muhur (opsiyonel).

**Request:**
```json
{
  "coverDocument": {
    "fileName": "UstYazi.pdf",
    "contentBase64": "<base64 PDF>",
    "mimeType": "application/pdf"
  },
  "documents": [
    {
      "fileName": "Ek1.pdf",
      "contentBase64": "<base64 PDF>",
      "mimeType": "application/pdf",
      "ekTur": "DED",
      "ekAd": "Birinci Ek",
      "ekImzaliMi": true,
      "ekSiraNo": 1
    }
  ],
  "version": "V2_0",
  "withSeal": false,
  "signatureLevel": "B_LT",
  "sealLevel": "B_LTA",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "ustveri": {
    "belgeId": "550e8400-e29b-41d4-a716-446655440000",
    "belgeNo": "2026/001",
    "tarih": "2026-04-17T10:00:00",
    "konu": "Test Yazismasi",
    "guvenlikKodu": "YOK",
    "dil": "tur",
    "olusturan": {
      "ad": "Moreum Teknoloji A.S.",
      "kkk": "1234567890"
    }
  },
  "nihaiUstveri": {
    "belgeNo": "2026/001",
    "belgeImzalar": [
      {
        "imzalayan": {
          "ilkAdi": "Ahmet",
          "soyadi": "Yilmaz",
          "tckn": "12345678901"
        },
        "amac": "Onay"
      }
    ]
  },
  "signerProvider": {
    "type": "token",
    "pin": "******"
  },
  "sealProvider": null
}
```

Alan aciklamalari:
- `coverDocument` — Ust yazi belgesi (opsiyonel). `FileName`, `ContentBase64`, `MimeType` iceren nesne.
- `documents` — En az bir ek belge (zorunlu). Her belgede `FileName` ve `ContentBase64` zorunlu.
- `version` — `V2_0` (varsayilan) veya `V2_1`
- `withSeal` — Muhur eklensin mi (varsayilan: false). `true` ise `sealProvider` zorunlu.
- `signatureLevel` / `sealLevel` — `B_B`, `B_T`, `B_LT`, `B_LTA`
- `ustveri.guvenlikKodu` — `YOK`, `GZL`, `HZN`, `COK_GZL`
- `signerProvider` — Imzalama provider yapilandirmasi (zorunlu)

**Response 200:**
```json
{
  "success": true,
  "eypPackageBase64": "<base64 ZIP/OPC>",
  "sizeBytes": 45678,
  "message": "EYP paketi basariyla olusturuldu"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/eyp/create \
  -H "Content-Type: application/json" \
  -d @eyp-create.json | jq '{success,sizeBytes}'
```

**SDK Karsiligi:** `sdk.CreateEypPackageV21Async(options, signerProvider, sealProvider)` (SDK_REFERANS Bolum 7.1)

---

### 7.2 POST /eyp/create-v13

EYP 1.3 paketi olusturur. Tek hash (SHA-256), opsiyonel muhur, opsiyonel zaman damgasi.

**Request:**
```json
{
  "coverDocument": {
    "fileName": "UstYazi.pdf",
    "contentBase64": "<base64 PDF>",
    "mimeType": "application/pdf"
  },
  "documents": [
    {
      "fileName": "Ek1.pdf",
      "contentBase64": "<base64 PDF>",
      "mimeType": "application/pdf"
    }
  ],
  "withSeal": false,
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "ustveri": {
    "belgeId": "550e8400-e29b-41d4-a716-446655440000",
    "belgeNo": "2026/001",
    "konu": "V1.3 Test Yazismasi",
    "guvenlikKodu": "YOK",
    "dil": "tur"
  },
  "signerProvider": {
    "type": "software",
    "certificateBase64": "<base64 PKCS12>",
    "certificatePassword": "<sertifika sifresi>"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "eypPackageBase64": "<base64 ZIP>",
  "sizeBytes": 32456,
  "version": "V1_3",
  "withSeal": false,
  "withTimestamp": true
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/eyp/create-v13 \
  -H "Content-Type: application/json" \
  -d @eyp-v13.json | jq '{success,version,withTimestamp}'
```

**SDK Karsiligi:** `sdk.CreateEypPackageV13Async(ustveri, ustYazi, ekler, signerProvider, sealProvider, tsaUrl)` (SDK_REFERANS Bolum 7.2)

---

### 7.3 POST /eyp/verify

EYP paketini dogrular. Yapi, hash, imza ve muhur butunlugunu kontrol eder.

**Request:**
```json
{
  "eypPackageBase64": "<base64 EYP ZIP>"
}
```

**Response 200:**
```json
{
  "isValid": true,
  "isPackageStructureValid": true,
  "isPaketOzetiValid": true,
  "isNihaiOzetValid": true,
  "isImzaValid": true,
  "isMuhurValid": true,
  "areHashesValid": true,
  "imzaLevel": "B_LT",
  "muhurLevel": "B_LTA",
  "imzaciSubject": "CN=Ahmet Yilmaz, O=Moreum, C=TR",
  "muhurSubject": "CN=Moreum E-Muhur, O=Moreum, C=TR",
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/eyp/verify \
  -H "Content-Type: application/json" \
  -d '{"eypPackageBase64":"<base64>"}' | jq '{isValid,imzaLevel}'
```

**SDK Karsiligi:** `sdk.VerifyEyp(eypData)` (SDK_REFERANS Bolum 7.3)

---

### 7.4 POST /eyp/extract

EYP paketinden belgeleri ve metadatay cikartir.

**Request:**
```json
{
  "eypPackageBase64": "<base64 EYP ZIP>"
}
```

**Response 200:**
```json
{
  "packageId": "550e8400-e29b-41d4-a716-446655440000",
  "createdAt": "2026-04-17T10:00:00Z",
  "coverDocument": {
    "fileName": "UstYazi.pdf",
    "mimeType": "application/pdf",
    "sizeBytes": 12345,
    "contentBase64": "<base64 PDF>"
  },
  "attachments": [
    {
      "fileName": "Ek1.pdf",
      "mimeType": "application/pdf",
      "sizeBytes": 5678,
      "contentBase64": "<base64 PDF>"
    }
  ],
  "metadata": {
    "belgeId": "550e8400-e29b-41d4-a716-446655440000",
    "belgeNo": "2026/001",
    "konu": "Test Yazismasi",
    "tarih": "2026-04-17T10:00:00Z",
    "dil": "tur"
  }
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/eyp/extract \
  -H "Content-Type: application/json" \
  -d '{"eypPackageBase64":"<base64>"}' | jq '{packageId,metadata}'
```

**SDK Karsiligi:** `sdk.ExtractEyp(eypData)` (SDK_REFERANS Bolum 7.4)

---

### 7.5 POST /eyp/sign-and-create

Belgeleri once PAdES/CAdES ile imzalar, ardindan imzali belgeleri EYP paketine koyar. Tek cagirida imzalama + EYP paketleme.

**Request:**
```json
{
  "documents": [
    {
      "fileName": "UstYazi.pdf",
      "contentBase64": "<base64 imzasiz PDF>",
      "mimeType": "application/pdf"
    },
    {
      "fileName": "Ek1.pdf",
      "contentBase64": "<base64 imzasiz PDF>",
      "mimeType": "application/pdf"
    }
  ],
  "signatureFormat": "PAdES",
  "signatureLevel": "B_T",
  "version": "V2_0",
  "withSeal": false,
  "eypSignatureLevel": "B_LT",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "ustveri": {
    "belgeNo": "2026/001",
    "konu": "Imzala ve Paketle"
  },
  "signerProvider": {
    "type": "token",
    "pin": "******"
  }
}
```

Alan aciklamalari:
- `documents` — Ilk belge ust yazi, geri kalanlar ek olarak eklenir.
- `signatureFormat` — Belge imza formati: `PAdES` (varsayilan), `CAdES`
- `eypSignatureLevel` — EYP paket imzasinin seviyesi (belge imzasindan bagimsiz)

**Response 200:**
```json
{
  "success": true,
  "eypPackageBase64": "<base64 ZIP/OPC>",
  "sizeBytes": 67890,
  "documentCount": 2,
  "eypVersion": "V2_0",
  "message": "EYP paketi basariyla olusturuldu"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/eyp/sign-and-create \
  -H "Content-Type: application/json" \
  -d @eyp-sign-create.json | jq '{success,documentCount,sizeBytes}'
```

**SDK Karsiligi:** `sdk.SignAndCreateEypAsync(options, signerProvider, docSignerProvider, sealProvider)` (SDK_REFERANS Bolum 7.5)

---

### 7.6 POST /eyp/encrypt

EYP paketini alici sertifikalari icin sifreler (CMS EnvelopedData; RSAES-OAEP + AES-256).

Istek: `{ "eypPackageBase64": "...", "recipientCertificatesBase64": ["<DER base64>", ...] }`
Yanit: `{ "success": true, "encryptedPackageBase64": "...", "sizeBytes": N }`

---

### 7.7 POST /eyp/decrypt

Sifreli EYP paketini alici PFX (sertifika + RSA ozel anahtar) ile cozer.

Istek: `{ "encryptedPackageBase64": "...", "recipientPfxBase64": "...", "pfxPassword": "..." }`
Yanit: `{ "success": true, "eypPackageBase64": "...", "sizeBytes": N }`

---

### 7.8 POST /eyp/guncelleme/create

EYP guncelleme paketi (dis OPC kabi) olusturur — TS 13298 V2.1 §6.2.4 (guvenlik kodu degisikligi vb.).
Guncellemeyi kurum e-muhru imzalar (`sealProvider`).

Istek: `{ "origEypBase64": "...", "sealProvider": { ... }, "tsaUrl": "...",
  "guncellemeler": [ { "guncellemeTuru": "GuvenlikKoduDegisikligi",
    "degisiklikler": [ { "yeniGizlilikDerecesi": "...", "degistirmeTarihi": "...",
      "aciklama": "...", "komisyonKarariBelgeNo": "...", "komisyonKarariBelgeId": "..." } ] } ] }`
Yanit: `{ "success": true, "eypPackageBase64": "...", "sizeBytes": N }`

---

### 7.9 POST /eyp/guncelleme/verify

EYP guncelleme paketini dogrular (dis OPC + ic orijinal paket).

Istek: `{ "eypPackageBase64": "..." }`
Yanit: `{ "success": true, "isValid": bool, "innerPackageIsValid": bool,
  "errors": [...], "innerPackageErrors": [...] }`

---

## 8. KEP Endpoint'leri

KEP (Kayitli Elektronik Posta) — 7201 sayili Kanun kapsaminda KEP paketi olusturma, dogrulama ve icerik cikartma.

### 8.1 POST /kep/create

KEP paketi olusturur. ZIP yapisi: kep-bilgi.xml + CAdES-T imza + ekler.

**Request:**
```json
{
  "senderAddress": "gonderen@hs01.kep.tr",
  "recipientAddresses": ["alici@hs02.kep.tr"],
  "subject": "Resmi Yazisma",
  "body": "Ekteki belgeleri bilginize sunarim.",
  "type": "Gonderi",
  "serviceProvider": "PTT KEP",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "attachments": [
    {
      "fileName": "belge.pdf",
      "contentBase64": "<base64 PDF>"
    }
  ],
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

Alan aciklamalari:
- `senderAddress` — Gonderen KEP adresi (zorunlu)
- `recipientAddresses` — En az bir alici KEP adresi (zorunlu)
- `type` — `Gonderi` (varsayilan), `Kabul`, `Alindi`, `Icerik`
- `tsaUrl` — TSA sunucusu URL'si (zorunlu; ya request'te ya `DefaultTsaUrl` konfigurasyonunda)
- `attachments` — Her ekte `fileName` ve `contentBase64` zorunlu

**Response 200:**
```json
{
  "success": true,
  "kepPackageBase64": "<base64 ZIP>",
  "kepId": "kep-uuid-1234",
  "sizeBytes": 23456,
  "attachmentCount": 1
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/create \
  -H "Content-Type: application/json" \
  -d @kep-create.json | jq '{success,kepId,sizeBytes}'
```

**SDK Karsiligi:** `sdk.CreateKepPackageAsync(package, provider, tsaUrl)` (SDK_REFERANS Bolum 8.1)

---

### 8.2 POST /kep/verify

KEP paketini dogrular. Imza, zaman damgasi ve yapisal butunlugu kontrol eder.

**Request:**
```json
{
  "kepPackageBase64": "<base64 KEP ZIP>"
}
```

**Response 200:**
```json
{
  "isValid": true,
  "isSignatureValid": true,
  "isTimestampValid": true,
  "hasRequiredFields": true,
  "signerSubject": "CN=KEP Imzacisi, O=PTT, C=TR",
  "signingTime": "2026-04-17T10:30:00Z",
  "timestampTime": "2026-04-17T10:30:05Z",
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/verify \
  -H "Content-Type: application/json" \
  -d '{"kepPackageBase64":"<base64>"}' | jq '{isValid,isTimestampValid}'
```

**SDK Karsiligi:** `sdk.VerifyKepPackage(kepData)` (SDK_REFERANS Bolum 8.2)

---

### 8.3 POST /kep/extract

KEP paketinden icerik cikartir. Metadata, ekler ve imza bilgilerini doner.

**Request:**
```json
{
  "kepPackageBase64": "<base64 KEP ZIP>"
}
```

**Response 200:**
```json
{
  "kepId": "kep-uuid-1234",
  "senderAddress": "gonderen@hs01.kep.tr",
  "recipientAddresses": ["alici@hs02.kep.tr"],
  "subject": "Resmi Yazisma",
  "body": "Ekteki belgeleri bilginize sunarim.",
  "sendTime": "2026-04-17T10:30:00Z",
  "type": "Gonderi",
  "serviceProvider": "PTT KEP",
  "hasSignature": true,
  "attachments": [
    {
      "fileName": "belge.pdf",
      "sizeBytes": 12345,
      "contentBase64": "<base64 PDF>"
    }
  ]
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/extract \
  -H "Content-Type: application/json" \
  -d '{"kepPackageBase64":"<base64>"}' | jq '{kepId,senderAddress,subject}'
```

**SDK Karsiligi:** `sdk.ExtractKepPackage(kepData)` (SDK_REFERANS Bolum 8.3)

---

### 8.4 POST /kep/sign-and-create

Belgeleri once PAdES/CAdES ile imzalar, ardindan imzali belgeleri KEP zarfina koyar.

**Request:**
```json
{
  "senderAddress": "gonderen@hs01.kep.tr",
  "recipientAddresses": ["alici@hs02.kep.tr"],
  "subject": "Imzali Belge Gonderi",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "documents": [
    {
      "fileName": "belge.pdf",
      "contentBase64": "<base64 imzasiz PDF>"
    }
  ],
  "signatureFormat": "PAdES",
  "signatureLevel": "B_T",
  "addTimestamp": true,
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "kepPackageBase64": "<base64 ZIP>",
  "kepId": "kep-uuid-5678",
  "sizeBytes": 34567,
  "documentCount": 1,
  "signatureFormat": "PAdES",
  "signatureLevel": "B_T"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/sign-and-create \
  -H "Content-Type: application/json" \
  -d @kep-sign-create.json | jq '{success,kepId,signatureFormat}'
```

**SDK Karsiligi:** `sdk.SignDataWithProviderAsync()` + `sdk.CreateKepPackageAsync()` (SDK_REFERANS Bolum 8.4)

---

### 8.5 POST /kep/sign-eyp-and-create

Belgeleri imzalar, EYP paketi olusturur ve EYP paketini KEP zarfina koyar. Tek cagirida: Imzalama -> EYP paketleme -> KEP zarfi.

**Request:**
```json
{
  "senderAddress": "gonderen@hs01.kep.tr",
  "recipientAddresses": ["alici@hs02.kep.tr"],
  "subject": "EYP Paketi Gonderi",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "documents": [
    {
      "fileName": "UstYazi.pdf",
      "contentBase64": "<base64 PDF>"
    }
  ],
  "signatureFormat": "PAdES",
  "signatureLevel": "B_T",
  "eypVersion": "V2_0",
  "withSeal": false,
  "eypSignatureLevel": "B_LT",
  "ustveri": {
    "belgeNo": "2026/001",
    "konu": "EYP Yazismasi"
  },
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "kepPackageBase64": "<base64 ZIP>",
  "sizeBytes": 56789,
  "eypSizeBytes": 45678,
  "documentCount": 1,
  "signatureFormat": "PAdES",
  "signatureLevel": "B_T",
  "eypVersion": "V2_0"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/sign-eyp-and-create \
  -H "Content-Type: application/json" \
  -d @kep-eyp.json | jq '{success,sizeBytes,eypSizeBytes}'
```

**SDK Karsiligi:** `sdk.SignAndCreateEypAsync()` + `sdk.CreateKepPackageAsync()` (SDK_REFERANS Bolum 8.5)

---

### 8.6 POST /kep/verify-delil

KEP delil paketini dogrular (Gonderi, Kabul, Alindi). Standart dogrulama + delil turune ozel kontroller.

**Request:**
```json
{
  "delilPackageBase64": "<base64 delil KEP ZIP>",
  "expectedType": "Kabul",
  "expectedRecipient": "alici@hs02.kep.tr",
  "originalKepId": "kep-uuid-1234"
}
```

Alan aciklamalari:
- `delilPackageBase64` — Dogrulanacak delil paketi (zorunlu)
- `expectedType` — Beklenen delil turu: `Gonderi`, `Kabul`, `Alindi` (opsiyonel kontrol)
- `expectedRecipient` — Beklenen alici adresi (opsiyonel kontrol)
- `originalKepId` — Orijinal KEP ID (delil eslestirme icin, opsiyonel)

**Response 200:**
```json
{
  "delilType": "Kabul",
  "isValid": true,
  "isSignatureValid": true,
  "isTimestampValid": true,
  "signerSubject": "CN=KEP Imzacisi, O=PTT, C=TR",
  "sendTime": "2026-04-17T10:30:00Z",
  "timestampTime": "2026-04-17T10:30:05Z",
  "timeDifference": "00:00:05",
  "sender": "gonderen@hs01.kep.tr",
  "recipients": ["alici@hs02.kep.tr"],
  "subject": "Resmi Yazisma",
  "originalKepId": "kep-uuid-1234",
  "delilKepId": "kep-uuid-1234",
  "attachmentCount": 1,
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/verify-delil \
  -H "Content-Type: application/json" \
  -d '{"delilPackageBase64":"<base64>","expectedType":"Kabul"}' | jq '{delilType,isValid}'
```

**SDK Karsiligi:** `sdk.VerifyKepPackage()` + `sdk.ExtractKepPackage()` birlesimi (SDK_REFERANS Bolum 8.6)

---

### 8.7 POST /kep/batch-create

Ayni belgeleri birden fazla alici grubuna ayri KEP zarflariyla gonderir. Her alici grubu icin bagimsiz paket olusturulur.

**Request:**
```json
{
  "senderAddress": "gonderen@hs01.kep.tr",
  "recipientGroups": [
    {
      "addresses": ["alici1@hs01.kep.tr"],
      "subject": "Grup 1 Ozel Konu"
    },
    {
      "addresses": ["alici2@hs02.kep.tr", "alici3@hs02.kep.tr"],
      "subject": null
    }
  ],
  "subject": "Varsayilan Konu",
  "type": "Gonderi",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "attachments": [
    {
      "fileName": "belge.pdf",
      "contentBase64": "<base64 PDF>"
    }
  ],
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

**Response 200:**
```json
{
  "totalGroups": 2,
  "successCount": 2,
  "failCount": 0,
  "results": [
    {
      "recipients": ["alici1@hs01.kep.tr"],
      "success": true,
      "kepPackageBase64": "<base64 ZIP>",
      "kepId": "kep-uuid-aaaa",
      "sizeBytes": 23456
    },
    {
      "recipients": ["alici2@hs02.kep.tr", "alici3@hs02.kep.tr"],
      "success": true,
      "kepPackageBase64": "<base64 ZIP>",
      "kepId": "kep-uuid-bbbb",
      "sizeBytes": 23460
    }
  ]
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/kep/batch-create \
  -H "Content-Type: application/json" \
  -d @kep-batch.json | jq '{totalGroups,successCount}'
```

**SDK Karsiligi:** Her grup icin `sdk.CreateKepPackageAsync()` (SDK_REFERANS Bolum 8.7)

---

## 9. UETS Endpoint'leri

UETS (Ulusal Elektronik Tebligat Sistemi) — 7201 sayili Tebligat Kanunu e-tebligat hukumleri kapsaminda tebligat paketi olusturma, dogrulama ve icerik cikartma. Zaman damgasi zorunludur (CAdES-T).

### 9.1 POST /uets/create

UETS tebligat paketi olusturur. ZIP: META-INF/uets-bilgi.xml + META-INF/imza.p7s + ekler.

**Request:**
```json
{
  "gonderen": {
    "uetsAdresi": "gonderen@uets.gov.tr",
    "adSoyad": "Ahmet Yilmaz",
    "tcKimlikNo": "12345678901",
    "muhatapTuru": "GercekKisi"
  },
  "muhatap": {
    "uetsAdresi": "alici@uets.gov.tr",
    "adSoyad": "Mehmet Kaya",
    "tcKimlikNo": "98765432109",
    "muhatapTuru": "GercekKisi"
  },
  "konu": "Tebligat Konusu",
  "aciklama": "Ekteki belge dikkatinize sunulur.",
  "tebligatTuru": "Gonderim",
  "birimKodu": "BRM-001",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "attachments": [
    {
      "fileName": "tebligat.pdf",
      "contentBase64": "<base64 PDF>"
    }
  ],
  "provider": {
    "type": "eseal",
    "certificateBase64": "<base64 PKCS12>",
    "certificatePassword": "<sertifika sifresi>"
  }
}
```

Alan aciklamalari:
- `gonderen.uetsAdresi` — Gonderen UETS adresi (zorunlu)
- `muhatap.uetsAdresi` — Alici UETS adresi (zorunlu)
- `muhatapTuru` — `GercekKisi` (varsayilan) veya `TuzelKisi`
- `tebligatTuru` — `Gonderim` (varsayilan), `Iade`, `Bildirim`, `Dogrulama`
- `tsaUrl` — TSA URL zorunlu (CAdES-T icin)

**Response 200:**
```json
{
  "success": true,
  "uetsPackageBase64": "<base64 ZIP>",
  "tebligatId": "tebligat-uuid-1234",
  "sizeBytes": 34567,
  "attachmentCount": 1
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/uets/create \
  -H "Content-Type: application/json" \
  -d @uets-create.json | jq '{success,tebligatId,sizeBytes}'
```

**SDK Karsiligi:** `sdk.CreateUetsPackageAsync(message, provider, tsaUrl)` (SDK_REFERANS Bolum 9.1)

---

### 9.2 POST /uets/verify

UETS paketini dogrular. Imza, zaman damgasi ve zorunlu alan butunlugunu kontrol eder.

**Request:**
```json
{
  "uetsPackageBase64": "<base64 UETS ZIP>"
}
```

**Response 200:**
```json
{
  "isValid": true,
  "isSignatureValid": true,
  "isTimestampValid": true,
  "hasRequiredFields": true,
  "signerSubject": "CN=Moreum E-Muhur, O=Moreum, C=TR",
  "signingTime": "2026-04-17T10:30:00Z",
  "timestampTime": "2026-04-17T10:30:05Z",
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/uets/verify \
  -H "Content-Type: application/json" \
  -d '{"uetsPackageBase64":"<base64>"}' | jq '{isValid,isTimestampValid}'
```

**SDK Karsiligi:** `sdk.VerifyUetsPackage(uetsData)` (SDK_REFERANS Bolum 9.2)

---

### 9.3 POST /uets/extract

UETS paketinden icerik cikartir. Tebligat metadata, taraf bilgileri ve ekleri doner.

**Request:**
```json
{
  "uetsPackageBase64": "<base64 UETS ZIP>"
}
```

**Response 200:**
```json
{
  "tebligatId": "tebligat-uuid-1234",
  "tebligatTuru": "Gonderim",
  "birimKodu": "BRM-001",
  "gonderimZamani": "2026-04-17T10:30:00Z",
  "teslimZamani": null,
  "gonderen": {
    "uetsAdresi": "gonderen@uets.gov.tr",
    "adSoyad": "Ahmet Yilmaz",
    "muhatapTuru": "GercekKisi",
    "tcKimlikNo": "12345678901",
    "vkn": null
  },
  "muhatap": {
    "uetsAdresi": "alici@uets.gov.tr",
    "adSoyad": "Mehmet Kaya",
    "muhatapTuru": "GercekKisi",
    "tcKimlikNo": "98765432109",
    "vkn": null
  },
  "konu": "Tebligat Konusu",
  "aciklama": "Ekteki belge dikkatinize sunulur.",
  "hasSignature": true,
  "attachments": [
    {
      "fileName": "tebligat.pdf",
      "sizeBytes": 12345,
      "contentBase64": "<base64 PDF>"
    }
  ]
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/uets/extract \
  -H "Content-Type: application/json" \
  -d '{"uetsPackageBase64":"<base64>"}' | jq '{tebligatId,konu,gonderen}'
```

**SDK Karsiligi:** `sdk.ExtractUetsPackage(uetsData)` (SDK_REFERANS Bolum 9.3)

---

## 10. ASiC Endpoint'leri

ASiC (Associated Signature Containers) — ETSI EN 319 162. ASiC-S tek belge, ASiC-E coklu belge konteyneri.

### 10.1 POST /asic/create-s

ASiC-S konteyner olusturur (tek belge + CAdES detached imza). ZIP yapisi: `mimetype` + belge + `META-INF/signature.p7s`.

**Request:**
```json
{
  "documentBase64": "<base64 belge>",
  "fileName": "sozlesme.pdf",
  "level": "B_T",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

Alan aciklamalari:
- `documentBase64` — Imzalanacak belge icerik (zorunlu)
- `fileName` — Belge dosya adi, uzanti dahil (zorunlu)
- `level` — `B_B` (varsayilan), `B_T`, `B_LT`
- `tsaUrl` — B_T ve uzeri icin gerekli

**Response 200:**
```json
{
  "success": true,
  "asicContainerBase64": "<base64 ZIP .asics>",
  "containerType": "ASiC-S",
  "fileName": "sozlesme.asics",
  "sizeBytes": 23456
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/asic/create-s \
  -H "Content-Type: application/json" \
  -d '{"documentBase64":"<base64>","fileName":"belge.pdf","level":"B_B","provider":{"type":"software","certificateBase64":"<p12>","certificatePassword":"pass"}}' | jq '{success,containerType,sizeBytes}'
```

**SDK Karsiligi:** `sdk.CreateAsicSAsync(docData, fileName, provider, parameters)` (SDK_REFERANS Bolum 10.1)

---

### 10.2 POST /asic/create-e

ASiC-E konteyner olusturur (coklu belge + manifest + CAdES imza). ZIP: `mimetype` + belgeler + `META-INF/ASiCManifest001.xml` + `META-INF/signatures001.p7s`.

**Request:**
```json
{
  "documents": [
    {
      "fileName": "sozlesme.pdf",
      "contentBase64": "<base64 PDF>",
      "mimeType": "application/pdf"
    },
    {
      "fileName": "ek.docx",
      "contentBase64": "<base64 DOCX>",
      "mimeType": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    }
  ],
  "level": "B_T",
  "tsaUrl": "http://zd.kamusm.gov.tr",
  "provider": {
    "type": "token",
    "pin": "******"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "asicContainerBase64": "<base64 ZIP .asice>",
  "containerType": "ASiC-E",
  "fileName": "asic-container.asice",
  "documentCount": 2,
  "sizeBytes": 45678
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/asic/create-e \
  -H "Content-Type: application/json" \
  -d @asic-e.json | jq '{success,containerType,documentCount}'
```

**SDK Karsiligi:** `sdk.CreateAsicEAsync(documents, provider, parameters)` (SDK_REFERANS Bolum 10.2)

---

### 10.3 POST /asic/verify

ASiC konteyneri dogrular (yapi, mimetype, imza, zaman damgasi).

**Request:**
```json
{
  "asicContainerBase64": "<base64 ASiC ZIP>"
}
```

**Response 200:**
```json
{
  "isValid": true,
  "containerType": "ASiC_S",
  "documentCount": 1,
  "signatureCount": 1,
  "detectedLevel": "B_T",
  "signerSubject": "CN=Ahmet Yilmaz, O=Moreum, C=TR",
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/asic/verify \
  -H "Content-Type: application/json" \
  -d '{"asicContainerBase64":"<base64>"}' | jq '{isValid,containerType,detectedLevel}'
```

**SDK Karsiligi:** `sdk.VerifyAsic(asicData)` (SDK_REFERANS Bolum 10.3)

---

### 10.4 POST /asic/extract

ASiC konteynerinden belgeleri ve imzalari cikartir.

**Request:**
```json
{
  "asicContainerBase64": "<base64 ASiC ZIP>"
}
```

**Response 200:**
```json
{
  "containerType": "ASiC_S",
  "mimeType": "application/vnd.etsi.asic-s+zip",
  "documents": [
    {
      "fileName": "sozlesme.pdf",
      "mimeType": "application/pdf",
      "sizeBytes": 12345,
      "contentBase64": "<base64 PDF>"
    }
  ],
  "signatures": [
    {
      "fileName": "META-INF/signature.p7s",
      "sizeBytes": 4096
    }
  ],
  "manifests": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/asic/extract \
  -H "Content-Type: application/json" \
  -d '{"asicContainerBase64":"<base64>"}' | jq '{containerType,documents}'
```

**SDK Karsiligi:** `sdk.ExtractAsic(asicData)` (SDK_REFERANS Bolum 10.4)

---

## 11. JAdES Endpoint'leri

JAdES (JSON Advanced Electronic Signatures) — ETSI TS 119 182-1. JWS Compact veya JSON Serialization formatinda imza.

### 11.1 POST /jades/sign

JAdES formatinda imza olusturur.

**Request:**
```json
{
  "dataBase64": "<base64 imzalanacak veri>",
  "level": "B_B",
  "hashAlgorithm": "SHA256",
  "tsaUrl": null,
  "detached": false,
  "provider": {
    "type": "software",
    "certificateBase64": "<base64 PKCS12>",
    "certificatePassword": "<sertifika sifresi>"
  }
}
```

Alan aciklamalari:
- `dataBase64` — Imzalanacak veri (zorunlu)
- `level` — `B_B` (varsayilan), `B_T`. B_T icin `tsaUrl` gerekli.
- `hashAlgorithm` — `SHA256` (varsayilan), `SHA384`, `SHA512`
- `detached` — `true` ise payload bos JWS (varsayilan: false)

**Response 200:**
```json
{
  "success": true,
  "jadesData": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "format": "JAdES",
  "level": "B_B",
  "detached": false,
  "sizeBytes": 4096
}
```

Not: `jadesData` JSON string olarak donmektedir (base64 degil — JWS Compact string veya JSON Serialization).

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/jades/sign \
  -H "Content-Type: application/json" \
  -d '{"dataBase64":"SGVsbG8gV29ybGQ=","level":"B_B","provider":{"type":"software","certificateBase64":"<p12>","certificatePassword":"pass"}}' | jq '{success,level,sizeBytes}'
```

**SDK Karsiligi:** `sdk.SignJadesAsync(data, provider, parameters)` (SDK_REFERANS Bolum 11.1)

---

### 11.2 POST /jades/verify

JAdES imzasini dogrular (Compact veya JSON Serialization).

**Request:**
```json
{
  "jadesData": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response 200:**
```json
{
  "isValid": true,
  "signerSubject": "CN=Ahmet Yilmaz, O=Moreum, C=TR",
  "signingTime": "2026-04-17T10:30:00Z",
  "level": "B_B",
  "errors": [],
  "warnings": []
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/jades/verify \
  -H "Content-Type: application/json" \
  -d '{"jadesData":"<JWS string>"}' | jq '{isValid,signerSubject}'
```

**SDK Karsiligi:** `sdk.VerifyJades(jadesData)` (SDK_REFERANS Bolum 11.2)

---

### 11.3 POST /jades/upgrade

JAdES imza seviyesini yukseltir (B_B -> B_T). Zaman damgasi ekler.

**Request:**
```json
{
  "jadesData": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "targetLevel": "B_T",
  "tsaUrl": "http://zd.kamusm.gov.tr"
}
```

Alan aciklamalari:
- `jadesData` — Yukseltilecek JAdES verisi (zorunlu)
- `targetLevel` — Hedef seviye: `B_T` veya uzeri (zorunlu)
- `tsaUrl` — TSA URL (zorunlu)

**Response 200:**
```json
{
  "success": true,
  "jadesData": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "level": "B_T",
  "sizeBytes": 5120
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/jades/upgrade \
  -H "Content-Type: application/json" \
  -d '{"jadesData":"<JWS>","targetLevel":"B_T","tsaUrl":"http://zd.kamusm.gov.tr"}' | jq '{success,level}'
```

**SDK Karsiligi:** `sdk.UpgradeJadesAsync(jadesData, targetLevel, tsaUrl)` (SDK_REFERANS Bolum 11.3)

---

## 11a. OOXML Imzalama

OOXML (Office Open XML) imzalama ayri bir controller'a sahip degildir. Standart `POST /api/v1/sign` endpoint'i `format=OOXML` parametresiyle kullanilir.

**Desteklenen dosya turleri:** `.docx`, `.xlsx`, `.pptx` (ECMA-376 / ISO 29500)

**Sinirlamalar:**
- Yalnizca `B_B` seviyesi desteklenir (XML-DSig, zaman damgasi eklenmez)
- Yalnizca RSA sertifikalari desteklenir (ECMA-376 gereksinimleri)

**Request (POST /api/v1/sign):**
```json
{
  "documentBase64": "<base64 .docx/.xlsx/.pptx>",
  "format": "OOXML",
  "parameters": {
    "level": "B_B"
  },
  "provider": {
    "type": "software",
    "certificateBase64": "<base64 PKCS12>",
    "certificatePassword": "<sertifika sifresi>"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "signedDocumentBase64": "<base64 imzali .docx/.xlsx/.pptx>",
  "message": "Signing successful"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{"documentBase64":"<base64 docx>","format":"OOXML","parameters":{"level":"B_B"},"provider":{"type":"software","certificateBase64":"<p12>","certificatePassword":"pass"}}' | jq '{success}'
```

**SDK Karsiligi:** `sdk.SignDocumentAsync(data, parameters, provider)` ile `format=OOXML` (SDK_REFERANS Bolum 3.1)

---

## 12. PDF Sifreleme Endpoint'leri

PDF belgelerini sifreleme, sifreni cozme ve sifreleme durumu kontrolu. Desteklenen algoritmalar: AES-256 (varsayilan), AES-128, RC4-128.

### 12.1 POST /pdf/encrypt

PDF belgesini sifreler.

**Request:**
```json
{
  "documentBase64": "<base64 PDF>",
  "ownerPassword": "<sahip sifresi>",
  "userPassword": "<kullanici sifresi>",
  "algorithm": "AES-256",
  "permissions": ["Print", "Copy"]
}
```

Alan aciklamalari:
- `documentBase64` — Sifrelencek PDF (zorunlu)
- `ownerPassword` — Sahip sifresi. Sifreleme izinleri ve sifreyi kaldir.
- `userPassword` — Kullanici sifresi. Belgeyi ac.
- En az biri zorunlu: `ownerPassword` veya `userPassword`
- `algorithm` — `AES-256` (varsayilan), `AES-128`, `RC4-128`
- `permissions` — Izin listesi (opsiyonel): `Print`, `Copy`, `Modify`, `Annotate`, `FillForms`, `ExtractForAccessibility`, `Assemble`, `PrintHighQuality`

**Response 200:**
```json
{
  "documentBase64": "<base64 sifreli PDF>",
  "algorithm": "AES-256"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/pdf/encrypt \
  -H "Content-Type: application/json" \
  -d '{"documentBase64":"<base64>","ownerPassword":"sahip123","algorithm":"AES-256"}' | jq '{algorithm}'
```

**SDK Karsiligi:** `sdk.EncryptPdf(pdfData, ownerPassword, userPassword)` (SDK_REFERANS Bolum 12.1)

---

### 12.2 POST /pdf/decrypt

Sifreli PDF belgesinin sifresini cozer.

**Request:**
```json
{
  "documentBase64": "<base64 sifreli PDF>",
  "password": "<sahip veya kullanici sifresi>"
}
```

Alan aciklamalari:
- `documentBase64` — Sifreli PDF (zorunlu)
- `password` — Sahip veya kullanici sifresi (zorunlu)

**Response 200:**
```json
{
  "documentBase64": "<base64 sifresiz PDF>"
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/pdf/decrypt \
  -H "Content-Type: application/json" \
  -d '{"documentBase64":"<base64 sifreli>","password":"sahip123"}' | jq '{documentBase64}' | head -c 200
```

**SDK Karsiligi:** `sdk.DecryptPdf(pdfData, password)` (SDK_REFERANS Bolum 12.2)

---

### 12.3 POST /pdf/is-encrypted

PDF belgesinin sifreli olup olmadigini kontrol eder.

**Request:**
```json
{
  "documentBase64": "<base64 PDF>"
}
```

**Response 200:**
```json
{
  "isEncrypted": true
}
```

**curl:**
```bash
curl -s -X POST http://localhost:7701/api/v1/pdf/is-encrypted \
  -H "Content-Type: application/json" \
  -d '{"documentBase64":"<base64>"}' | jq .
```

**SDK Karsiligi:** `sdk.IsPdfEncrypted(pdfData)` (SDK_REFERANS Bolum 12.3)

---

## 13. Token / PKCS#11 Yonetimi

Token ve HSM erisimi icin iki farkli kullanim modeli destekler:

### 13.0 Iki Kullanim Modeli

#### Model A: Her Imza Icin Yeni Authentication

Her `/sign` cagrisinda request body icinde `provider` blogu ile authentication bilgisi (PIN + PKCS#11 path + slot) gecer. API katmani bu bilgi ile HsmSigningProvider olusturur, imzalar, dispose eder.

```json
POST /sign
{
  "documentBase64": "...",
  "provider": {
    "type": "token",
    "pin": "<TOKEN_PIN>",
    "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll",
    "slotId": 0
  }
}
```

- **Kullanim:** Her istek bagimsiz (stateless), microservice mimarisi, tek imza senaryolari
- **Dezavantaj:** PIN her istekte body'de transfer — HTTPS zorunlu, log maskeleme onemli

#### Model B: Session-based (Long-lived)

`POST /token/sessions` ile bir kez authenticate olup session ID al; sonraki imza cagrilarinda `sessionId` kullan; isin bitince `DELETE /token/sessions/{id}`.

```
1. POST /token/sessions  (authentication, token-server session'i acar)
   -> { "sessionId": "abc123", "expiresAt": "2026-04-18T...", "certificateCount": 2 }

2. GET  /token/sessions/{sessionId}/certificates  (PIN tekrar sormadan listele)

3. POST /sign  (sessionId ile imzala — body'de PIN yok)
   {
     "documentBase64": "...",
     "provider": { "type": "session", "sessionId": "abc123" }
   }
   (N kez tekrarlanabilir)

4. DELETE /token/sessions/{sessionId}  (oturumu kapat)
```

- **Kullanim:** Web oturumu (login → N imza → logout), batch signing, tablet/kiosk akisi
- **Avantaj:** PIN tek sefer, network'te bir kez; kullanici deneyimi iyi
- **Dikkat:** Session timeout (varsayilan 5 dk idle) — kapatmayi unutsaniz bile otomatik temizlenir

#### Secim Tablosu

| Durum | Model |
|---|---|
| Backend imza servisi (microservice) | **A** |
| Web UI ile batch (10+ belge) | **B** |
| Kullanici oturumu boyunca yeniden yeniden imza | **B** |
| Tablet kiosk | **B** (+ signing-sessions, bkz. Bolum 14) |
| CI/CD pipeline, tek imza | **A** |

---

### 13.1 GET /token/slots
### 13.2 GET /token/slots/{slotId}/certificates
### 13.3 POST /token/sessions
### 13.4 GET /token/sessions
### 13.5 GET /token/sessions/{sessionId}
### 13.6 GET /token/sessions/{sessionId}/certificates
### 13.7 DELETE /token/sessions/{sessionId}
### 13.8 POST /token/pin/verify [YENI 2.0]
### 13.9 GET /token/pin/status
### 13.10 GET /token/status

---

## 14. Oturum Imzalama (SigningSession)

PDF belgeler icin kullanici arayuzu tabanli imzalama akisi. Token, mobil imza ve el imzasi (tablet cizimi) yontemlerini destekler. Session olusturulur, kullaniciya UI URL'si verilir, imzalama UI'dan gerceklestirilir, tamamlaninca callback ile sonuc iletilir.

**Tipik akis:**
1. `POST /signing-sessions` → `sessionId` + `uiUrl` al
2. `uiUrl`'yi kullaniciya goster (iframe veya redirect)
3. Kullanici UI'da imzalar (token, mobil veya el imzasi)
4. `callbackUrl`'ye imzali PDF POST edilir

**Session timeout:** Idle 30 dakika (her mutating istekte resetlenir) + Absolute 8 saat (CreatedAt'tan itibaren, resetlenmez). Süresi dolan session sonraki istekte HTTP 410 Gone döner.

---

### 14.1 POST /signing-sessions

Yeni imzalama session'i olusturur. Cagiran sistem PDF belgelerini, callback URL'sini ve imzalama tercihlerini gonderir; karsiliginda kullaniciya acilacak UI URL'sini alir.

**Request:**
```json
{
    "documents": [
        {
            "id": "doc-1",
            "documentBase64": "JVBERi0x...",
            "fileName": "sozlesme.pdf",
            "requiresSignature": true,
            "signaturePlacement": {
                "page": 1,
                "x": 400,
                "y": 700,
                "width": 150,
                "height": 50
            },
            "footerNote": "Imzalayan: {user}",
            "footerY": 12.0
        }
    ],
    "callbackUrl": "https://app.example.com/signing-callback",
    "signatureFormat": "PAdES",
    "signatureLevel": "B_T",
    "tsaProviderKey": "default",
    "addTimestamp": true,
    "reason": "Sozlesme onayi",
    "location": "Istanbul",
    "allowedMethods": ["token", "mobile", "handwritten"],
    "handwrittenVerification": "sms-otp",
    "phoneNumber": "905551234567",
    "signerName": "Ahmet Yilmaz",
    "footerTemplate": "{date} tarihinde {user} tarafindan imzalandi",
    "digitalSignatureVisual": "default",
    "allowUserPlacement": true
}
```

**Response (201 Created):**
```json
{
    "sessionId": "a1b2c3d4e5f6...",
    "uiUrl": "http://localhost:7701/signing-ui/?sessionId=a1b2c3d4e5f6...",
    "expiresAt": "2026-04-17T11:30:00Z"
}
```

**Zorunlu alanlar:** `documents` (en az 1 eleman, her birinde `documentBase64`)
**Opsiyonel:** `callbackUrl`, `signatureFormat`, `signatureLevel`, `allowedMethods`, `handwrittenVerification`, `phoneNumber`, `signerName`, `footerTemplate`, `digitalSignatureVisual`, `reason`, `location`, `tsaProviderKey`, `addTimestamp`, `allowUserPlacement`

**Not:** `handwrittenVerification = "sms-otp"` ise `phoneNumber` zorunludur.

#### handwrittenVerification — el imzasi kimlik dogrulamasi

Gonderen uygulama, el imzasi oncesi kimlik dogrulamasinin yapilip yapilmayacagini ve hangi
yontemle yapilacagini bu alanla secer. Bilinmeyen bir deger 400 ile reddedilir.

| Deger | Anlam |
|-------|-------|
| `none` (varsayilan) | Dogrulama yok, yalniz imza cizimi. |
| `sms-otp` | SMS OTP; `phoneNumber` zorunlu. |
| `mrz` | TC Kimlik arka yuz MRZ'si kameradan okunur. |
| `nfc` | TC Kimlik NFC ile okunur; cihazda PC/SC temassiz okuyucu gerekir. |

`mrz` veya `nfc` secildiginde `POST /signing-sessions/{id}/handwritten-sign` kimlik
dogrulanana kadar 400 dondurur; baska bir yontemle dogrulanan kimlik de kabul edilmez.

**Kapsam:** Bu alan yalniz el imzasini etkiler. `token-sign` ve `mobile-sign` uclarinda kimlik
zaten nitelikli sertifikaya bagli oldugu icin kimlik kapisi uygulanmaz.

---

### 14.2 GET /signing-sessions/{sessionId}

Session durumunu sorgular.

**Response (200 OK):**
```json
{
    "sessionId": "a1b2c3d4...",
    "status": "Completed",
    "method": "token",
    "createdAt": "2026-04-17T10:00:00Z",
    "expiresAt": "2026-04-17T10:30:00Z",
    "documentCount": 1,
    "signedCount": 1,
    "callbackStatus": "Sent",
    "error": null,
    "allowedMethods": ["token", "mobile", "handwritten"],
    "signatureFormat": "PAdES",
    "signatureLevel": "B_T",
    "documents": [
        {
            "id": "doc-1",
            "fileName": "sozlesme.pdf",
            "signed": true,
            "requiresSignature": true,
            "signaturePlacement": { "page": 1, "x": 400, "y": 700, "width": 150, "height": 50 }
        }
    ]
}
```

**status degerleri:** `Pending` | `Signing` | `Completed` | `Failed` | `Expired`

**404:** Session bulunamazsa veya suresi dolmussa.

---

### 14.3 GET /signing-sessions/{sessionId}/documents/{docIndex}

Ham PDF bytes doner (PDF.js rendering icin). Response: `application/pdf`

---

### 14.4 GET /signing-sessions/{sessionId}/documents/{docIndex}/signed

Imzali PDF bytes doner (indirme icin). Response: `application/pdf` + `Content-Disposition: attachment`

**404:** Belge henuz imzalanmadiysa.

---

### 14.5 PATCH /signing-sessions/{sessionId}/documents/{docIndex}/placement

Belgenin imza gorsel konumunu gunceller (kullanici PDF uzerinde surukleyerek secer).

**Request:**
```json
{
    "page": 1,
    "x": 350,
    "y": 650,
    "width": 150,
    "height": 50
}
```

**Not:** `allowUserPlacement: false` olan session'larda 400 doner.

---

### 14.6 POST /signing-sessions/{sessionId}/otp/send

SMS OTP gonderir. `handwrittenVerification = "sms-otp"` olan session'larda kullanilir.

**Request:**
```json
{
    "phoneNumber": "905551234567"
}
```

---

### 14.7 POST /signing-sessions/{sessionId}/otp/verify

OTP kodunu dogrular. Basarili olursa el imzasi adimine gecilir.

**Request:**
```json
{
    "code": "123456"
}
```

---

### 14.8 POST /signing-sessions/{sessionId}/token-sign

Token ile imzalar. Token session ID veya direkt PIN gonderilebilir.

**Request:**
```json
{
    "sessionId": "token-session-id",
    "pin": null,
    "libraryPath": null,
    "slotId": 0
}
```

---

### 14.9 POST /signing-sessions/{sessionId}/mobile-sign

Mobil imza baslatir. Kullanicinin telefonuna onay mesaji gider.

**Request:**
```json
{
    "msisdn": "905551234567",
    "mobileOperator": "Turkcell"
}
```

---

### 14.10 POST /signing-sessions/{sessionId}/handwritten-sign

El imzasi gorselini (base64 PNG canvas) PDF'e isler.

**Request:**
```json
{
    "signatureImageBase64": "iVBORw0KGgo..."
}
```

---

### 14.11 POST /signing-sessions/{sessionId}/nfc-verified

NFC TC Kimlik dogrulama sonucunu iletir (DigiMR Agent'tan).

---

### 14.12 POST /signing-sessions/{sessionId}/send-callback

Callback'i manuel tetikler. Otomatik callback basarisizsa kullanilir.

---

### 14.13 POST /signing-sessions/{sessionId}/send-to-url

Imzali PDF'i belirtilen URL'ye POST eder.

**Request:**
```json
{
    "url": "https://app.example.com/receive-signed",
    "headers": { "Authorization": "Bearer ..." }
}
```

---

### 14.14 GET /signing-sessions/agent/tokens

Agent'taki token listesini sorgular.

---

### 14.15 POST /signing-sessions/agent/session

Agent uzerinde token session olusturur.

---

### 14.16 GET /signing-sessions/agent/tokens/{slotId}/certificates

Agent'taki slot sertifikalarini sorgular.

---

### 14.17 GET /signing-sessions/agent/pin-status

Agent uzerindeki token PIN durumunu sorgular (kilitli mi, son deneme mi).

---

### 14.18 DELETE /signing-sessions/agent/session/{agentSessionId}

Agent token session'ini kapatir.

---

## 15. Provider Bilgi

Desteklenen imzalama provider tiplerini kesfetme ve lokal/uzak token listeleme. Kaynak: `ProviderController.cs` (`/api/v1/providers`)

---

### 15.1 GET /providers

Desteklenen provider tiplerini listeler. Donanim veya lisans gerektirmez.

**Response (200 OK):**
```json
{
    "providers": [
        { "type": "Software", "description": "PKCS#12 yazilim sertifikasi ile imzalama" },
        { "type": "Token",    "description": "Lokal PKCS#11 USB token ile imzalama" },
        { "type": "HSM",      "description": "PKCS#11 HSM (Hardware Security Module) ile imzalama" },
        { "type": "RemoteToken", "description": "Uzak Token Agent uzerinden imzalama" },
        { "type": "Mobile",   "description": "Mobil imza (Turkcell/Vodafone/TurkTelekom)" },
        { "type": "ESeal",    "description": "Elektronik muhur (e-Seal) ile imzalama" },
        { "type": "CloudHSM", "description": "Cloud HSM (AWS/Azure/GCP) ile imzalama" },
        { "type": "Biometric","description": "Biyometrik dogrulama ile imzalama" }
    ]
}
```

---

### 15.2 GET /providers/tokens

API sunucusundaki lokal PKCS#11 token'lari listeler. `libraryPath` verilmezse config'deki varsayilan kullanilir.

**Query parametresi:** `?libraryPath=C:/Windows/System32/eTPKCS11.dll` (opsiyonel)

**Response (200 OK):**
```json
[
    {
        "slotId": 0,
        "label": "eToken Pro",
        "manufacturerId": "SafeNet",
        "model": "eToken 5110",
        "serialNumber": "01234567",
        "library": "C:/Windows/System32/eTPKCS11.dll"
    }
]
```

---

### 15.3 POST /providers/remote/tokens

Uzak Token Agent uzerindeki token'lari sorgular.

**Request:**
```json
{
    "agentUrl": "http://192.168.1.100:7703",
    "apiKey": "your-agent-api-key"
}
```

**Response (200 OK):** 15.2 ile ayni format.

**500:** Agent erisilemediyse `{ "error": "Agent baglanti hatasi: ..." }`

---

## 16. Preservation

ETSI TS 119 511/512 uyumlu uzun vadeli koruma (long-term preservation). Arsiv zaman damgasi yonetimi. Kaynak: `PreservationController.cs` (`/api/v1/preservation`)

---

### 16.1 POST /preservation/check

Imzali belgenin arsiv zaman damgasi durumunu kontrol eder. Yenileme gerekip gerekmedigini, hash algoritmasinin guvenligini ve arsiv zaman damgasi sayisini raporlar.

**Request:**
```json
{
    "documentBase64": "JVBERi0x..."
}
```

**Response (200 OK):**
```json
{
    "needsRenewal": false,
    "oldestArchiveTimestamp": "2024-01-15T10:00:00Z",
    "newestArchiveTimestamp": "2025-06-01T12:00:00Z",
    "currentHashAlgorithm": "SHA256",
    "isHashAlgorithmWeak": false,
    "archiveTimestampCount": 2,
    "recommendation": "Arsiv zaman damgasi gecerli, 2027'de yenileme oneriliyor"
}
```

**400:** `documentBase64` eksikse veya gecersiz base64.
**500:** Belge uyumsuz formatta.

---

### 16.2 POST /preservation/renew

Arsiv zaman damgasini yenileyerek belgenin uzun vadeli gecerliligini uzatir. Mevcut imzaya yeni bir `archive-time-stamp-v3` ekler.

**Request:**
```json
{
    "documentBase64": "JVBERi0x...",
    "tsaUrl": "http://tsa.example.com/rfc3161"
}
```

**Notlar:**
- `tsaUrl` verilmezse `DefaultTsaUrl` config degeri kullanilir; ikisi de yoksa 400.
- KamuSM TSA: Wireguard IP whitelist + RFC 3161 + kimlik bilgileri gerektirir. Sadece lokal test.

**Response (200 OK):**
```json
{
    "success": true,
    "renewedDocumentBase64": "JVBERi0x...",
    "sizeBytes": 102400
}
```

---

### 16.3 POST /preservation/evidence

Belgeden tum arsiv zaman damgasi zincirini (Evidence Record) cikarir. Her zaman damgasinin uretim zamani, hash algoritmasi ve TSA bilgisini dondurur.

**Request:**
```json
{
    "documentBase64": "JVBERi0x..."
}
```

**Response (200 OK):**
```json
{
    "chainLength": 2,
    "firstTimestamp": "2024-01-15T10:00:00Z",
    "lastTimestamp": "2025-06-01T12:00:00Z",
    "timestamps": [
        {
            "index": 0,
            "genTime": "2024-01-15T10:00:00Z",
            "hashAlgorithm": "SHA256",
            "tsaName": "TubiTak UEKAE TSA",
            "isValid": true
        },
        {
            "index": 1,
            "genTime": "2025-06-01T12:00:00Z",
            "hashAlgorithm": "SHA256",
            "tsaName": "TubiTak UEKAE TSA",
            "isValid": true
        }
    ]
}
```

---

## 17. Export

Imzali belgeden ham imza bytes ayiklar. Kaynak: `ExportController.cs` (`/api/v1/export`)

---

### 17.1 POST /export

Imzali belgeden imza verisini disari aktarir.

- **CAdES:** Belgenin kendisi imza verisidir (detached).
- **PAdES:** PDF'ten imza baytlari cikarilir.

**Request:**
```json
{
    "signedDocumentBase64": "JVBERi0x..."
}
```

**Response (200 OK):**
```json
{
    "success": true,
    "signatureDataBase64": "MIIHZgYJKo...",
    "sizeBytes": 4096
}
```

**400:** `signedDocumentBase64` eksikse veya gecersiz base64.
**500:** Belge desteklenmeyen formatta veya imza ayiklanamadiysa.

---

## 18. Health

Servis saglik ve hazirlik kontrolleri. Kaynak: `HealthController.cs` (`/api/v1/health`)

---

### 18.1 GET /health

Temel saglik kontrolu. Servis ayakta mi?

**Response (200 OK):**
```json
{
    "status": "healthy",
    "timestamp": "2026-04-17T10:00:00Z",
    "version": "2.0.0.0"
}
```

---

### 18.2 GET /health/ready

Detayli hazirlik kontrolu. Bagimliliklar erisebilir mi? (TSA, SDK)

**Response (200 OK — tum hazir):**
```json
{
    "status": "ready",
    "timestamp": "2026-04-17T10:00:00Z",
    "checks": [
        { "name": "TSA",  "status": "healthy", "details": "URL: http://tsa.example.com, Response: 405" },
        { "name": "SDK",  "status": "healthy",  "details": null }
    ]
}
```

**Response (503 Service Unavailable — degraded):**
```json
{
    "status": "degraded",
    "timestamp": "2026-04-17T10:00:00Z",
    "checks": [
        { "name": "TSA",  "status": "unhealthy", "details": "URL: http://tsa.example.com, Error: Connection refused" },
        { "name": "SDK",  "status": "healthy",   "details": null }
    ]
}
```

**status degerleri:**
- `ready` → 200 (tum kontroller healthy veya unconfigured)
- `degraded` → 503 (en az bir kontrol unhealthy)
- TSA yapilandirilmamissa `"status": "unconfigured"` ile 200 doner

---

## 19. Ek Controller'lar

Asagidaki controller'lar operasyonel / sistem yonetimi icin kullanilir.
Detaylar (Admin, Log, Config, Setup, CRL, TrustedList, CSC, EUDI, Biometric) sonraki task'larda doldurulacak.

---

## Appendix A: HTTP Hata Kodlari

Tum endpoint'lerde kullanilan standart HTTP durum kodlari:

| Kod | Anlam | Tipik Neden |
|-----|-------|-------------|
| 200 | OK | Basarili |
| 201 | Created | Session olusturuldu (`POST /signing-sessions`) |
| 400 | Bad Request | Gecersiz JSON, eksik zorunlu alan, yanlis format, kisitlama ihlali |
| 401 | Unauthorized | Lisans suresi dolmus veya authentication eksik |
| 404 | Not Found | Session ID, job ID veya kaynak bulunamadi |
| 422 | Unprocessable Entity | Validation basarisiz (orn. Level=B-T icin TsaUrl gerekli) |
| 500 | Internal Server Error | Signer/TSA/token iletisim hatasi, beklenmeyen istisna |
| 503 | Service Unavailable | Bagimlilik erisebilir degil (TSA down, trust store bozuk) |

**Hata response formati** (tum 4xx/5xx icin):
```json
{
    "error": "Aciklayici hata mesaji",
    "hint": "Opsiyonel cozum onerisi"
}
```

Bazi endpoint'ler (dogrulama, imzalama) 200 ile hata bilgisi donderebilir — `isValid: false` veya `success: false` alanina bakin.

**PIN kilitleme uyarisi:** Token PIN yanlis girilirse donanim kalici kilitlenebilir. `/token/pin/status` ile once durum sorgulama onerilir.

---

## Appendix B: Ortak Request/Response Modelleri

### ProviderConfig

Tum imzalama endpoint'lerinde (`/sign`, `/sign/batch`, `/sign/prepare`, `/sign/finalize`, vb.) `provider` alani olarak gonderilir.

```json
{
    "type": "software | token | hsm | remote-token | mobile | eseal | cloudhsm | biometric",

    "certificateBase64": "MIIHZg...",
    "certificatePassword": "<PFX_PASSWORD>",

    "pin": "<TOKEN_PIN>",
    "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll",
    "slotId": 0,
    "tokenFilter": "eToken Pro",
    "keyLabel": "imza-key",
    "keyId": "01020304",
    "allowFinalTry": false,
    "sessionId": "token-session-id",
    "pinMode": "session | per_signature",

    "msisdn": "905551234567",
    "mobileOperator": "Turkcell | Vodafone | TurkTelekom",

    "agentUrl": "http://192.168.1.100:7703",
    "agentApiKey": "agent-api-key",
    "agentSessionId": "agent-session-id",
    "certificateIndex": 0
}
```

**Guvenik notlari:**
- `pin`, `certificatePassword`, `agentApiKey` alanlari response'larda `[JsonIgnore]` ile gizlenir — hicbir zaman API yaniti olarak donmez.
- `sessionId` verildiginde `pin` gerekmez; mevcut token session kullanilir.
- `allowFinalTry: true` sadece Development modunda etkindir.

### SignatureParametersDto

`/sign` ve diger imzalama endpoint'lerinde `parameters` alani.

```json
{
    "level": "B_B | B_T | B_LT | B_LTA",
    "tsaUrl": "http://tsa.example.com/rfc3161",
    "tsaProviderKey": "default",
    "reason": "Sozlesme onayi",
    "location": "Istanbul",
    "signerRole": "CEO",
    "signatureAppearance": "default | base64-png",
    "signatureField": "Signature1",
    "signaturePage": 1
}
```

### Token PIN Durum Modeli

`GET /token/pin/status` response:

```json
{
    "isLocked":   false,
    "isFinalTry": false,
    "isCountLow": true,
    "isOk":       false,
    "message":    "Kalan deneme sayisi az"
}
```

### Token Session Modeli

`POST /token/sessions` response:

```json
{
    "sessionId": "abc123",
    "providerType": "token",
    "slotId": 0,
    "expiresIn": 1800,
    "message": "Token session olusturuldu..."
}
```

`GET /token/sessions/{sessionId}` response:

```json
{
    "sessionId": "abc123",
    "providerType": "token",
    "slotId": 0,
    "isAuthenticated": true,
    "createdAt": "2026-04-17T10:00:00Z",
    "lastUsedAt": "2026-04-17T10:05:00Z",
    "expiresAt": "2026-04-17T10:35:00Z"
}
```

---

## Appendix C: gRPC Yansimasi (port 7702)

Tum REST endpoint'lerinin gRPC karsiligi `7702` portunda calisir. Proto dosyasi: `src/DigitalSignature.API/Protos/signature.proto`

**Temel farklar:**
- Protobuf binary format (daha verimli, daha az bandwidth)
- HTTP/2 multiplexing
- Stream destegi (batch operations icin)
- Swagger UI yerine gRPC reflection veya `.proto` dosyasi kullanilir

**Service listesi:**

| Service | Aciklama | Ornek RPC'ler |
|---------|----------|---------------|
| `DigitalSignatureService` | Imzalama, dogrulama, yukseltme, KEP, timestamp, preservation, export | `SignDocument`, `ValidateDocument`, `UpgradeSignature`, `CheckHealth`, `CheckReady` |
| `EypService` | EYP paketi olusturma ve dogrulama | `CreateEypV2`, `CreateEypV13`, `VerifyEyp`, `ExtractEyp` |
| `UetsService` | UETS paketi islemleri | `CreateUets`, `VerifyUets`, `ExtractUets` |
| `AsicService` | ASiC-S / ASiC-E konteyner | `CreateAsicS`, `CreateAsicE`, `VerifyAsic`, `ExtractAsic` |
| `JadesService` | JSON Advanced Electronic Signature | `SignJades`, `VerifyJades`, `UpgradeJades` |
| `PdfService` | PDF sifreleme | `EncryptPdf`, `DecryptPdf`, `IsPdfEncrypted` |
| `ProviderService` | Token/session yonetimi | `GetProviders`, `ListTokens`, `GetTokenPinStatus`, `CreateTokenSession`, `ListTokenSessions`, `CloseTokenSession`, `ListSessionCertificates` |

**C# client ornegi:**

```csharp
using var channel = GrpcChannel.ForAddress("http://localhost:7702");
var client = new DigitalSignatureService.DigitalSignatureServiceClient(channel);

var response = await client.SignDocumentAsync(new SignDocumentRequest
{
    DocumentBase64 = Convert.ToBase64String(pdfBytes),
    Format = "PAdES",
    Provider = new ProviderConfigProto
    {
        Type = "token",
        SessionId = "abc123"
    },
    Parameters = new SignatureParametersProto
    {
        Level = "B_T",
        TsaUrl = "http://tsa.example.com/rfc3161"
    }
});
```

**Java'dan ham gRPC istemci ornegi.** Hazir stub'lar icin `digimr-sdk-1.0.0-all.jar`'i
(v2.3.2 release varligi) classpath'e eklemek yeterlidir; asagidaki tipler o jar'dan gelir.
In-process yerel imza icin saf-Java SDK (`sdk/java-native`, release hazirlaniyor)
kullanilacaktir.

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 7702).usePlaintext().build();
DigitalSignatureServiceGrpc.DigitalSignatureServiceBlockingStub stub =
    DigitalSignatureServiceGrpc.newBlockingStub(channel);
SignDocumentResponse response = stub.signDocument(request);
```

**Notlar:**
- `package` alani degistirilemez — gRPC wire uyumluluğu bozulur. Sadece `java_package` degistirilebilir.
- REST ve gRPC endpoint'leri birebir eslesir; ayni SDK metodlarini cagirir.
- Health check: `DigitalSignatureService.CheckHealth` (basit) ve `CheckReady` (bagimlilik kontrollu) RPC'leri mevcuttur.
