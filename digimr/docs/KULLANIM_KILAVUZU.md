# DigiMR Kullanim Kilavuzu

**Versiyon**: 3.0 | **Tarih**: Mart 2026 | **Base URL**: `http://localhost:7701/api/v1`

---

## Icindekiler

1. [Giris](#1-giris)
2. [Kurulum](#2-kurulum)
3. [SDK Kullanimi (.NET)](#3-sdk-kullanimi-net)
4. [REST API Kullanimi](#4-rest-api-kullanimi)
5. [Agent Kullanimi](#5-agent-kullanimi)
6. [Token/HSM Yonetimi](#6-tokenhsm-yonetimi)
7. [Zaman Damgasi (TSA)](#7-zaman-damgasi-tsa)
8. [Turk Mevzuati Uyumu](#8-turk-mevzuati-uyumu)

---

## 1. Giris

### DigiMR Nedir?

DigiMR, Turkiye mevzuatina tam uyumlu bir dijital imza kutuphanesidir. .NET 8 uzerinde BouncyCastle 2.4.0 ve iTextSharp ile calisir. SDK, REST API, masaustu Agent ve Web UI olarak kullanilabilir.

### Desteklenen Imza Formatlari

| Format | Standart | Kullanim |
|--------|----------|----------|
| **PAdES** | ETSI EN 319 142-1 | PDF imzalama |
| **CAdES** | ETSI EN 319 122-1 | Her tur dosya, detached .p7s |
| **XAdES** | ETSI EN 319 132-1 | XML, e-Defter, e-Fatura |
| **JAdES** | ETSI TS 119 182-1 | JSON tabanlı imza (JWS) |
| **ASiC-S** | ETSI EN 319 162-1 | Basit konteyner (tek belge) |
| **ASiC-E** | ETSI EN 319 162-1 | Genisletilmis konteyner (coklu belge) |
| **EYP** | CBDDO e-Yazisma Paketi | Kamu yazisma (v1.3, v2.0, v2.1) |
| **KEP** | 7201 / ETSI TS 102 640 | Kayitli elektronik posta |
| **UETS** | Ulusal Elektronik Tebligat | Tebligat paketi |

### Desteklenen Imza Seviyeleri

| Seviye | Aciklama | TSA Gerekli? |
|--------|----------|-------------|
| **B-B** | Temel imza (sertifika + imza) | Hayir |
| **B-T** | Zaman damgali imza | Evet |
| **B-C** | CompleteCertificateRefs + CompleteRevocationRefs | Evet |
| **B-X** | B-C + SigAndRefsTimeStamp | Evet |
| **B-LT** | CRL/OCSP gomulu, uzun donemli dogrulama | Evet |
| **B-LTA** | Arsiv zaman damgali (en yuksek seviye) | Evet |

### Mimari

```
+------------------+     +------------------+     +------------------+
|   SDK (NuGet)    |     |  REST API (:7701)|     | Agent (:5555)    |
|   .NET in-proc   |     |  HTTP/JSON       |     | Lokal token      |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         +------------------------+------------------------+
                          |
              +-----------+-----------+
              |   DigitalSignature    |
              |   Core + Pdf + Xml    |
              |   Token + JAdES       |
              +-----------------------+
```

- **SDK**: .NET projelerine dogrudan NuGet referansi
- **REST API**: Tum dil ve platformlardan HTTP ile erisim
- **Agent**: Kullanici bilgisayarinda calisan servis, browser'dan token erisimi
- **Web UI**: Swagger (`/swagger`), KEP UI (`/kep-ui`), Agent UI (`/agent-ui`)

---

## 2. Kurulum

### 2.1 NuGet ile SDK Kullanimi

```bash
dotnet add package DigiMR.SDK
```

```csharp
using DigitalSignature.SDK;

var sdk = new DigitalSignatureSDK();
```

### 2.2 API Sunucusunu Calistirma

```bash
# Kaynak koddan
cd src/DigitalSignature.API
dotnet run -c Release

# Yayinlanmis binary'den
dotnet DigitalSignature.API.dll

# Varsayilan port: 7701
# Swagger UI: http://localhost:7701/swagger
# Health: http://localhost:7701/api/v1/health
```

### 2.3 Docker

```bash
docker run -p 7701:7701 digimr
```

### 2.4 Agent Kurulumu

Agent, kullanicinin bilgisayarinda calisarak USB token/akilli kart erisimi saglar.

```bash
# Windows
curl -L https://github.com/moreum-tech/digimr/releases/latest/download/digimr-agent-win-x64.zip -o agent.zip
unzip agent.zip -d "C:\Program Files\digimr"

# Rust Agent Lite (2.9 MB, browser indirmesi icin)
# Ayni API, .NET Agent ile degistirilebilir
```

Agent varsayilan olarak `http://localhost:5555` adresinde calisir.

### 2.5 Konfigürasyon (appsettings.json)

```json
{
  "Kestrel": {
    "Endpoints": {
      "Local":    { "Url": "http://127.0.0.1:7701" },
      "External": { "Url": "http://0.0.0.0:7701" }
    }
  },

  "DefaultTsaUrl": "http://tzd.kamusm.gov.tr",
  "TimestampServer": {
    "UserId":   7521,
    "Password": "3VilLuA8"
  },

  "TsaProviders": {
    "Default": "kamusm",
    "Providers": {
      "kamusm": {
        "Name": "TUBITAK Kamu SM",
        "Url": "http://tzd.kamusm.gov.tr",
        "AuthType": "kamusm-identity",
        "UserId": 7521,
        "Password": "3VilLuA8",
        "Enabled": true
      },
      "digicert": {
        "Name": "DigiCert",
        "Url": "http://timestamp.digicert.com",
        "AuthType": "none",
        "Enabled": true
      }
    }
  },

  "TokenSession": {
    "IdleTimeoutMinutes": 5,
    "AcquireTimeoutSeconds": 30,
    "CleanupIntervalSeconds": 30
  },

  "Security": {
    "MaxDocumentSizeBytes": 104857600,
    "MaxBatchDocuments": 100,
    "MaxBatchTotalSizeBytes": 524288000,
    "RateLimitPerMinute": 100
  }
}
```

**Ortam Degiskenleri:**
- `TSA_MODE`: `free` (DigiCert), `kamusm` (Kamu SM), `local` (test stub)

---

## 3. SDK Kullanimi (.NET)

### 3.1 Provider Olusturma — `CreateAndAuthenticateProviderAsync`

Imzalama icin once provider olusturulur. Provider, ozel anahtarin nerede oldugunu belirler.

```csharp
// Yazilim sertifikasi (.pfx)
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificatePath = "sertifika.pfx",
        CertificatePassword = "sifre123"
    });

// USB Token (PKCS#11)
using var tokenProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = "123456",
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll",
        SlotId = 0
    });

// Mobil imza
using var mobileProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905321234567",
        Metadata = new Dictionary<string, string> { ["Operator"] = "Turkcell" }
    });

// Uzak token (Agent uzerinden)
using var remoteProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.RemoteToken,
    new AuthenticationContext
    {
        AgentUrl = "https://agent-sunucu:5555",
        AgentApiKey = "api-key",
        Pin = "1234",
        CertificateIndex = 0
    });

// HSM (kurumsal)
using var hsmProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.HSM,
    new AuthenticationContext
    {
        Pin = "hsm-pin",
        Pkcs11LibraryPath = @"C:\hsm\pkcs11.dll",
        Metadata = new Dictionary<string, string> { ["HsmPoolSize"] = "5" }
    });
```

**Desteklenen Provider Tipleri:** `Software`, `Token`, `Mobile`, `RemoteToken`, `HSM`, `ESeal`, `Biometric`

### 3.2 PDF Imzalama — `SignPdfWithProviderAsync`

```csharp
Task<SignatureResult> SignPdfWithProviderAsync(
    string documentPath,
    ISigningProvider provider,
    SignatureParameters parameters);
```

```csharp
var result = await sdk.SignPdfWithProviderAsync("belge.pdf", provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    Reason = "Onayladim",
    Location = "Ankara",
    VisualSignature = new VisualSignatureOptions
    {
        Mode = VisualSignatureMode.StampOnly,
        Page = 1, X = 350, Y = 50, Width = 200, Height = 60,
        StampText = "ONAYLANDI",
        StampStyle = StampStyle.RoundedRectangle,
        StampColor = "#003366"
    }
});

if (result.Success)
    File.WriteAllBytes("imzali.pdf", result.SignedData!);
```

### 3.3 CAdES Imzalama — `SignDataWithProviderAsync`

```csharp
Task<SignatureResult> SignDataWithProviderAsync(
    byte[] data,
    ISigningProvider provider,
    SignatureParameters parameters);
```

```csharp
var data = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(data, provider, new SignatureParameters
{
    Format = SignatureFormat.CAdES,
    Level = SignatureLevel.B_LTA,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});

if (result.Success)
    File.WriteAllBytes("belge.p7s", result.SignedData!);
```

### 3.4 XML Imzalama — `SignXmlWithProviderAsync`

```csharp
Task<SignatureResult> SignXmlWithProviderAsync(
    string documentPath,
    ISigningProvider provider,
    SignatureParameters parameters);
```

```csharp
var result = await sdk.SignXmlWithProviderAsync("fatura.xml", provider, new SignatureParameters
{
    Format = SignatureFormat.XAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    DetachedSignature = false  // enveloped
});
```

### 3.5 JAdES Imzalama — `SignJadesAsync`

JSON Advanced Electronic Signatures (JWS tabanlı).

```csharp
var result = await sdk.SignJadesAsync(data, provider, new SignatureParameters
{
    Format = SignatureFormat.JAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
```

### 3.6 Dogrulama — `ValidateDocumentAsync`

Otomatik format tespiti: PDF, CAdES, XAdES, EYP 1.3/2.0/2.1, ASiC.

```csharp
Task<ValidationResult> ValidateDocumentAsync(byte[] documentData);
Task<ValidationResult> ValidateDocumentAsync(string filePath);
```

```csharp
var result = await sdk.ValidateDocumentAsync("imzali.pdf");

Console.WriteLine($"Format: {result.Format}");         // PAdES
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Butunluk: {result.IsDocumentIntact}");

foreach (var sig in result.Signatures)
{
    Console.WriteLine($"  Imzaci: {sig.SignerSubject}");
    Console.WriteLine($"  Seviye: {sig.Level}");
    Console.WriteLine($"  Tarih: {sig.SignatureTime}");
}
```

### 3.7 EYP Paketi — `CreateEypPackageV2Async`

```csharp
Task<EypCreateResult> CreateEypPackageV2Async(
    EypCreateOptions options,
    ISigningProvider signerProvider,
    ISigningProvider? sealProvider = null);
```

```csharp
var options = new EypCreateOptions
{
    Version = EypVersion.V2_0,
    SealMode = EypSealMode.WithSeal,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    ImzaLevel = SignatureLevel.B_LT,
    MuhurLevel = SignatureLevel.B_LTA,
    UstYazi = new EypDocumentItem
    {
        FileName = "ustyazi.pdf",
        Content = File.ReadAllBytes("ustyazi.pdf"),
        MimeType = "application/pdf"
    },
    Ekler = new List<EypDocumentItem>
    {
        new() { FileName = "ek-1.pdf", Content = File.ReadAllBytes("ek-1.pdf"), MimeType = "application/pdf" }
    },
    Ustveri = new EypUstveri { BelgeNo = "2026-001", Konu = "Onay Yazisi", Dil = "tur" },
    NihaiUstveri = new EypNihaiUstveri { Tarih = DateTime.UtcNow, BelgeNo = "2026-001" }
};

var result = await sdk.CreateEypPackageV2Async(options, signerProvider, sealProvider);
if (result.Success)
    File.WriteAllBytes("paket.eyp", result.PackageData!);
```

Dogrulama: `sdk.VerifyEypPackageV2(eypData)` — `EypVerificationResult` doner.
Cikarma: `sdk.ExtractEypPackageV2(eypData)` — `EypPackageInfo` doner.

### 3.8 KEP Paketi — `CreateKepPackageAsync`

```csharp
var package = new KepPackage
{
    SenderAddress = "gonderen@kep.tr",
    RecipientAddresses = { "alici@kep.tr" },
    Subject = "Resmi Yazi",
    Body = "Mesaj metni",
    Type = KepType.Icerik,
    Attachments = { ["belge.pdf"] = pdfBytes }
};

byte[] kepZip = await sdk.CreateKepPackageAsync(package, provider, "http://tzd.kamusm.gov.tr");
```

### 3.9 ASiC Konteyner — `CreateAsicSAsync` / `CreateAsicEAsync`

```csharp
// ASiC-S: tek belge
var asicS = await sdk.CreateAsicSAsync(data, provider, parameters);

// ASiC-E: coklu belge
var asicE = await sdk.CreateAsicEAsync(documents, provider, parameters);
```

### 3.10 Seviye Yukseltme — `UpgradeSignatureAsync`

```csharp
var upgraded = await sdk.UpgradeSignatureAsync(signedData, SignatureLevel.B_LTA, tsaUrl);
```

---

## 4. REST API Kullanimi

Tum endpointler `http://localhost:7701/api/v1` altindadir. Belgeler base64 kodlanmis olarak gonderilir.

### 4.1 Imzalama — `POST /api/v1/sign`

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "format": "PAdES",
    "provider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "p12_sifresi"
    },
    "parameters": {
      "level": "B_T",
      "tsaProviderKey": "kamusm",
      "reason": "Onayladim",
      "location": "Ankara"
    }
  }'
```

**Yanit:**
```json
{
  "success": true,
  "signedDocumentBase64": "<imzali_belge_base64>",
  "message": "PAdES B_T imza basarili"
}
```

**format Degerleri:** `PAdES`, `CAdES`, `XAdES`, `ASiC-S`, `ASiC-E`, `EYP`, `KEP`

**provider Tipleri:** `software`, `token`, `hsm`, `remote-token`, `mobile`

**parameters Referansi:**

| Alan | Tur | Aciklama |
|------|-----|----------|
| `level` | string | `B_B`, `B_T`, `B_LT`, `B_LTA` |
| `tsaUrl` | string | TSA URL override |
| `tsaProviderKey` | string | `kamusm`, `digicert`, `turktrust` vb. |
| `reason` | string | Imzalama nedeni |
| `location` | string | Imzalama yeri |
| `hashAlgorithm` | string | `SHA256` (varsayilan), `SHA384`, `SHA512` |
| `signaturePolicy` | string | BTK profili: `P1`-`P4` |
| `visualSignature` | object | Gorunur imza ayarlari (sadece PAdES) |
| `signerRole` | object | `{ claimedRoles: ["Mudur"] }` |
| `commitmentType` | string | `ProofOfApproval`, `ProofOfCreation` vb. |
| `productionPlace` | object | `{ city, countryName }` |

### 4.2 Toplu Imzalama — `POST /api/v1/sign/batch`

Tek PIN girisiyle birden fazla belge imzalar. Maks 100 belge, 500 MB.

```bash
curl -X POST http://localhost:7701/api/v1/sign/batch \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [
      { "id": "fatura-001", "documentBase64": "<base64>" },
      { "id": "fatura-002", "documentBase64": "<base64>" }
    ],
    "format": "PAdES",
    "provider": { "type": "token", "pin": "1234", "pkcs11LibraryPath": "/usr/lib/libakis.so" },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
  }'
```

**Yanit:**
```json
{
  "results": [
    { "id": "fatura-001", "success": true, "signedDocumentBase64": "<base64>" },
    { "id": "fatura-002", "success": false, "error": "Belge bozuk" }
  ],
  "summary": { "total": 2, "succeeded": 1, "failed": 1, "elapsedMs": 2500 }
}
```

### 4.3 Dogrulama — `POST /api/v1/verify`

Format otomatik algilanir (PAdES, CAdES, XAdES, EYP, ASiC).

```bash
curl -X POST http://localhost:7701/api/v1/verify \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i imzali.pdf)'"}'
```

**Yanit:**
```json
{
  "isValid": true,
  "format": "PAdES",
  "isDocumentIntact": true,
  "signatures": [
    {
      "signerSubject": "CN=Ahmet Yilmaz, O=Sirket, C=TR",
      "isValid": true,
      "level": "B_T",
      "hasTimestamp": true,
      "signatureTime": "2026-03-04T15:30:00Z"
    }
  ]
}
```

### 4.4 Hedefli Kontrol — `POST /api/v1/verify/check`

"Bu belgede PAdES B-T var mi?" sorusuna cevap verir.

```bash
curl -X POST http://localhost:7701/api/v1/verify/check \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "<base64>", "format": "PAdES", "level": "B_T"}'
```

### 4.5 Tam Envanter — `POST /api/v1/verify/inspect`

Belgede bulunan tum imza tiplerini listeler.

```bash
curl -X POST http://localhost:7701/api/v1/verify/inspect \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "<base64>"}'
```

### 4.6 EYP/KEP Dogrulama — `POST /api/v1/verify/eyp`, `/verify/kep`

EYP ve KEP paketleri icin detayli dogrulama (yapi + hash + imza + muhur).

### 4.7 Token Oturum API

PIN'i bir kez girerek `sessionId` alin. Sonraki tum imzalamalarda sessionId kullanin.

```bash
# Slot listeleme (PIN gerekmez)
curl "http://localhost:7701/api/v1/token/slots?libraryPath=C:/Windows/System32/eTPKCS11.dll"

# Oturum acma
curl -X POST http://localhost:7701/api/v1/token/sessions \
  -H "Content-Type: application/json" \
  -d '{"libraryPath": "C:/Windows/System32/eTPKCS11.dll", "pin": "123456", "slotId": 0}'

# Yanit: { "sessionId": "abc123", "expiresAt": "...", "certificateCount": 1 }

# sessionId ile imzalama
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "<base64>",
    "format": "PAdES",
    "provider": { "type": "token", "sessionId": "abc123" },
    "parameters": { "level": "B_T" }
  }'

# Oturumu kapat
curl -X DELETE http://localhost:7701/api/v1/token/sessions/abc123
```

### 4.8 EYP API

```bash
# Olusturma
POST /api/v1/eyp/create
# Dogrulama
POST /api/v1/eyp/verify
# Cikarma
POST /api/v1/eyp/extract
```

### 4.9 KEP API

```bash
# Olusturma
POST /api/v1/kep/create
# Dogrulama
POST /api/v1/kep/verify
# Cikarma
POST /api/v1/kep/extract
```

### 4.10 ASiC API

```bash
POST /api/v1/asic/create-s    # ASiC-S olustur
POST /api/v1/asic/create-e    # ASiC-E olustur
POST /api/v1/asic/verify      # ASiC dogrula
POST /api/v1/asic/extract     # ASiC icerik cikar
```

### 4.11 UETS API

```bash
POST /api/v1/uets/create      # UETS paketi olustur
POST /api/v1/uets/verify      # UETS dogrula
POST /api/v1/uets/extract     # UETS icerik cikar
```

### 4.12 JAdES API

```bash
POST /api/v1/jades/sign       # JAdES imza olustur
POST /api/v1/jades/verify     # JAdES dogrula
POST /api/v1/jades/upgrade    # Seviye yukselt (B-B -> B-T)
```

### 4.13 CSC API (Cloud Signature Consortium)

ETSI TS 119 432 uyumlu bulut imza API'si.

```bash
POST /api/v1/csc/info                    # Servis bilgisi
POST /api/v1/csc/credentials/list        # Credential listele
POST /api/v1/csc/credentials/info        # Credential detay
POST /api/v1/csc/credentials/authorize   # PIN ile yetkilendir -> SAD
POST /api/v1/csc/signatures/signHash     # Hash imzala
```

### 4.14 Preservation API (Uzun Vadeli Koruma)

ETSI TS 119 511/512 uyumlu arsiv zaman damgasi yonetimi.

```bash
# Durum kontrolu
POST /api/v1/preservation/check
# Yanit: { needsRenewal, oldestArchiveTimestamp, recommendation }

# Arsiv zaman damgasi yenileme
POST /api/v1/preservation/renew
# Yanit: { success, renewedDocumentBase64 }

# Kanit zinciri (Evidence Record) cikarma
POST /api/v1/preservation/evidence
# Yanit: { chainLength, timestamps: [...] }
```

### 4.15 EUDI Wallet API

EU Digital Identity Wallet entegrasyonu (prepare/finalize pattern).

```bash
POST /api/v1/eudi/request-signing    # Imzalama talebi olustur
POST /api/v1/eudi/callback           # Wallet'tan imzali hash'i al
GET  /api/v1/eudi/status/{requestId} # Durumu kontrol et
```

### 4.16 Iki Asamali Imzalama — `POST /api/v1/sign/prepare` + `/finalize`

PIN sunucuya gonderilmeden, hash disarida imzalanir.

```bash
# 1. Hazirlama
curl -X POST http://localhost:7701/api/v1/sign/prepare \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "<base64>",
    "format": "PAdES",
    "certificateBase64": "<sertifika_DER_base64>",
    "parameters": { "level": "B_T" }
  }'
# Yanit: { "jobId": "job-uuid", "hashBase64": "<imzalanacak_hash>", "expiresIn": 300 }

# 2. Hash'i disarida imzala (token, HSM, wallet vb.)

# 3. Tamamlama
curl -X POST http://localhost:7701/api/v1/sign/finalize \
  -H "Content-Type: application/json" \
  -d '{"jobId": "job-uuid", "rawSignatureBase64": "<ham_imza>"}'
# Yanit: { "success": true, "signedDocumentBase64": "<imzali_belge>" }
```

### 4.17 Seviye Yukseltme — `POST /api/v1/upgrade`

```bash
curl -X POST http://localhost:7701/api/v1/upgrade \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i imzali_bb.pdf)'",
    "targetLevel": "B_T",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

### 4.18 Zaman Damgasi — `POST /api/v1/timestamp/token`

```bash
curl -X POST http://localhost:7701/api/v1/timestamp/token \
  -H "Content-Type: application/json" \
  -d '{"dataBase64": "'$(base64 -i dosya.pdf)'"}'
```

### 4.19 Diger Endpointler

| Endpoint | Aciklama |
|----------|----------|
| `GET /api/v1/health` | Servis durumu |
| `GET /api/v1/health/ready` | Bagimlilik kontrolu |
| `GET /api/v1/sign/formats` | Desteklenen formatlar/seviyeler |
| `GET /api/v1/providers` | Provider tipleri |
| `GET /api/v1/providers/tokens` | Lokal PKCS#11 tokenlar |
| `GET /api/v1/timestamp/providers` | TSA saglayicilar |
| `GET /api/v1/admin/agents` | Bagli agent'lar |
| `POST /api/v1/admin/agents/{id}/sign-hash` | Agent uzerinden hash imzala |
| `GET /api/v1/config` | API konfigurasyonu |
| `GET /api/v1/crl/*` | CRL izleme durumu |
| `GET /api/v1/trustedlist/*` | Guvenilir liste |
| `GET /api/v1/logs` | API loglari |

---

## 5. Agent Kullanimi

### 5.1 Agent Nedir?

Desktop Agent, kullanicinin bilgisayarinda localhost:5555'te calisan bir servistir. USB token/akilli kart gibi donanima erisim saglayarak browser'dan imza atilmasini mumkun kilar. Ozel anahtar hicbir zaman agent sunucusunu terk etmez.

Iki versiyon mevcuttur:
- **.NET Agent** (55 MB): Tam ozellikli, gRPC destegi
- **Rust Agent Lite** (2.9 MB): Hafif, browser indirmesi icin ideal

Her iki agent'in API'si aynidir — degistirilebilir.

### 5.2 Agent Baslat

```bash
# .NET Agent
dotnet DigitalSignature.Agent.dll

# Rust Agent Lite
./agent-lite
```

Varsayilan: `http://localhost:5555`

### 5.3 Browser'dan Agent Algilama

```javascript
// Agent varligini kontrol et
fetch('http://localhost:5555/api/agent/status')
  .then(r => r.json())
  .then(data => console.log('Agent aktif:', data.status));
```

### 5.4 Agent UI

`http://localhost:5555/agent-ui` — Web tabanli UI:
- PIN girisi
- Token durumu
- Hub baglanti durumu
- Log izleme

### 5.5 Agent Hub (SignalR)

Agent'lar merkezi API'ye SignalR uzerinden baglanabilir:

```json
// Agent appsettings.json
{
  "CentralApiUrl": "http://sunucu:7701/api/v1/agent-hub"
}
```

`CentralApiUrl` bos ise agent standalone modda calisir.

### 5.6 postMessage Entegrasyonu

Browser icinde iframe uzerinden agent ile iletisim:

```javascript
// imza-app iframe'ine mesaj gonder
iframe.contentWindow.postMessage({
  action: 'sign',
  documentBase64: '...',
  format: 'PAdES',
  level: 'B_T'
}, '*');

// Yanit dinle
window.addEventListener('message', (event) => {
  if (event.data.action === 'signResult') {
    console.log('Imzali belge:', event.data.signedDocumentBase64);
  }
});
```

---

## 6. Token/HSM Yonetimi

### 6.1 PKCS#11 Token Erisimi

DigiMR, PKCS#11 uyumlu tum token ve akilli kartlari destekler: AKiS, SafeNet eToken, Gemalto, SoftHSM2 vb.

```csharp
// Token filter ile otomatik bulma
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = "123456",
        TokenFilter = "SafeNet"   // Label, seri no veya uretici ile eslestir
    });
```

### 6.2 SoftHSM2 Test Ortami

Gelistirme ortami icin yazilim tabanli PKCS#11 token:

```bash
# SoftHSM2 kurulumu
# Windows: choco install softhsm
# Linux: apt install softhsm2

# Token olustur
softhsm2-util --init-token --slot 0 --label "test" --pin 1234 --so-pin 0000

# Kutuphane yolu
# Windows: C:\SoftHSM2\lib\softhsm2-64.dll
# Linux: /usr/lib/softhsm/libsofthsm2.so
```

### 6.3 eToken Yapilandirmasi

```json
{
  "type": "token",
  "pin": "123456",
  "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll",
  "slotId": 0
}
```

### 6.4 PIN Modlari

| PinMode | Davranis |
|---------|----------|
| `SessionStart` | Oturum basinda bir kez giris (varsayilan) |
| `PerSignature` | Her imzada PIN sorulur |
| `Cached` | PIN TTL kadar onbellege alinir |
| `ProtectedPath` | PIN kart okuyucudan girilir |
| `AutoDetect` | Token yeteneklerine gore otomatik |

```json
{
  "type": "token",
  "pin": "1234",
  "metadata": { "PinMode": "Cached", "PinCacheTtlSeconds": "300" }
}
```

### 6.5 Token Oturum API (REST)

```
GET  /api/v1/token/slots?libraryPath=...                  # Slotlari listele
GET  /api/v1/token/slots/{slotId}/certificates?...         # Sertifikalari listele (PIN gerekmez)
POST /api/v1/token/sessions                                # Oturum ac -> sessionId
GET  /api/v1/token/sessions                                # Aktif oturumlar
GET  /api/v1/token/sessions/{sessionId}                    # Oturum detay
GET  /api/v1/token/sessions/{sessionId}/certificates       # Oturum sertifikalari
DELETE /api/v1/token/sessions/{sessionId}                   # Oturumu kapat
POST /api/v1/token/pin/verify                              # PIN dogrula (oturum acmadan)
```

---

## 7. Zaman Damgasi (TSA)

### 7.1 RFC 3161

DigiMR yalnizca RFC 3161 protokolunu destekler (HTTP degil). Zaman damgasi B-T ve uzeri seviyelerde zorunludur.

### 7.2 Kamu SM Entegrasyonu

```json
{
  "TsaProviders": {
    "Providers": {
      "kamusm": {
        "Url": "http://tzd.kamusm.gov.tr",
        "AuthType": "kamusm-identity",
        "UserId": 7521,
        "Password": "3VilLuA8"
      }
    }
  }
}
```

- **Test**: `http://tzd.kamusm.gov.tr` (kredi sinirli)
- **Uretim**: `http://zd.kamusm.gov.tr`
- `kamusm-identity`: PBKDF2+AES ozel protokol

### 7.3 DigiCert (Ucretsiz)

```json
{
  "digicert": {
    "Url": "http://timestamp.digicert.com",
    "AuthType": "none",
    "Enabled": true
  }
}
```

### 7.4 Multi-TSA Provider

Istek bazinda `tsaProviderKey` ile farkli TSA secimi:

```json
{
  "parameters": {
    "level": "B_T",
    "tsaProviderKey": "kamusm"
  }
}
```

### 7.5 TSA AuthType Degerleri

| Deger | Aciklama |
|-------|----------|
| `kamusm-identity` | TUBITAK KAMUSM'e ozel PBKDF2+AES |
| `http-basic` | HTTP Basic Auth |
| `none` | Kimlik dogrulama yok |

### 7.6 Turkiye'de TSA Saglayicilari

| Saglayici | AuthType | Durum |
|-----------|----------|-------|
| TUBITAK Kamu SM | `kamusm-identity` | Aktif |
| TURKTRUST | `http-basic` | Yapilandirma hazir |
| E-Guven | `http-basic` | Yapilandirma hazir |
| DigiCert | `none` | Aktif (ucretsiz) |

---

## 8. Turk Mevzuati Uyumu

### 8.1 BTK P1-P4 Profilleri

| Profil | ETSI Seviye | Policy OID | Revokasyon |
|--------|------------|-----------|-----------|
| **P1** | B-B (BES) | Yok | CRL veya OCSP |
| **P2** | B-T | `2.16.792.1.61.0.1.5070.3.1.1` | CRL zorunlu, 24 saat bekleme |
| **P3** | B-LT | `2.16.792.1.61.0.1.5070.3.2.1` | CRL + gomulu, 24 saat bekleme |
| **P4** | B-LT | `2.16.792.1.61.0.1.5070.3.3.1` | OCSP + gomulu, anlik |

SDK'da kullanim:

```csharp
var p1 = TurkishSignatureProfiles.CreateP1Parameters(SignatureFormat.PAdES);
var p2 = TurkishSignatureProfiles.CreateP2Parameters(policyDigestValue, tsaUrl);
var p3 = TurkishSignatureProfiles.CreateP3Parameters(policyDigestValue);
var p4 = TurkishSignatureProfiles.CreateP4Parameters(policyDigestValue);
```

API'da: `"signaturePolicy": "P2"` seklinde gonderilir.

### 8.2 5070 Sayili Kanun

- Guvenli elektronik imza, el yazisi imza ile esit hukuki gecerliliktedir
- 2021 degisikligi ile elektronik muhur (e-Seal) eklendi
- Nitelikli elektronik sertifika (NES) gerektirir

### 8.3 EYP 2.0/2.1 — CBDDO Standardi

Cumhurbaskanligi Dijital Donusum Ofisi standardi:
- OPC tabanlı paket yapisi
- Cift hash: SHA-384 + SHA-512
- Zorunlu CAdES B-LT imza + CAdES B-LTA muhur
- EYP 2.1: Guncelleme paketi destegi, P3 profili

### 8.4 KEP — 7201 Sayili Kanun

Kayitli Elektronik Posta:
- CAdES-T detached imza (zaman damgali)
- Paket yapisi: ZIP + META-INF/kep-bilgi.xml + imza.p7s
- Tur: Gonderi, Kabul, Alindi, Icerik

### 8.5 DSS Dogrulama Sonucu

Turk CA'lari AATL veya EUTL listesinde yer almadigi icin:
- DSS Demo: `INDETERMINATE` / `NO_CERTIFICATE_CHAIN_FOUND` gosterir
- Bu, imzanin gecersizligi degil, guven listesi eksikligidir
- **5070 sayili kanun kapsaminda imza hukuken tam gecerlidir**

---

## Hata Kodlari

| HTTP | Aciklama |
|------|----------|
| 200 | Basarili |
| 400 | Gecersiz istek |
| 429 | Rate limit asildi (100 istek/dakika/IP) |
| 500 | Sunucu hatasi |
| 503 | Servis hazir degil |

```json
{
  "error": "Saglayici bulunamadi: 'xyz'. Mevcut saglayicilar: kamusm"
}
```

---

*Detayli API referansi icin: [API_DOCUMENTATION.md](API_DOCUMENTATION.md)*
*SDK tip referansi icin: [SDK_REFERENCE.md](SDK_REFERENCE.md)*
*Ornek kodlar icin: [ORNEKLER.md](ORNEKLER.md)*

---

*Copyright 2026 Moreum Tech*
