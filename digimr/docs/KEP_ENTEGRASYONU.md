# DigiMR KEP Entegrasyon Rehberi

Bu dokuman, DigiMR'nin KEP (Kayitli Elektronik Posta) yeteneklerinin dis sistemler (ornegin terradocs correspondence modulu) tarafindan nasil kullanilacagini anlatir.

## Genel Bakis

DigiMR KEP islemlerinde **kriptografik katmani** saglar:
- KEP zarfi (ZIP paketi) olusturma
- CAdES-T imza atma (zaman damgali)
- Zarf dogrulama (imza + yapi + zaman damgasi)
- Zarf icerigini cikarma (metadata + ekler)

KEP saglayiciya ozgu islemler (SMTP/API baglantisi, polling, delil takibi) cagiran tarafin sorumluluGundadir.

## Entegrasyon Yontemleri

### Yontem 1: REST API (Onerilen)

DigiMR API servisi calisirken HTTP uzerinden erisim.

**Base URL:** `http://localhost:5000/api/v1/kep`

---

### 1. KEP Paketi Olusturma

```
POST /api/v1/kep/create
Content-Type: application/json
```

**Request:**
```json
{
  "senderAddress": "kurum@kephs.gov.tr",
  "recipientAddresses": [
    "alici1@kephs.gov.tr",
    "alici2@kephs.gov.tr"
  ],
  "subject": "Resmi Yazi",
  "body": "Bu bir KEP mesajidir.",
  "type": "Icerik",
  "serviceProvider": "PTT KEP",
  "tsaUrl": "http://zd.kamusm.gov.tr/ts",
  "attachments": [
    {
      "fileName": "yazi.pdf",
      "contentBase64": "JVBERi0xLjQ..."
    },
    {
      "fileName": "ek-tablo.xlsx",
      "contentBase64": "UEsDBBQA..."
    }
  ],
  "provider": {
    "type": "software",
    "certificateBase64": "MIIKYgIBAz...",
    "certificatePassword": "sifre123"
  }
}
```

**Response (200):**
```json
{
  "success": true,
  "kepPackageBase64": "UEsDBBQACAgI...",
  "kepId": "a1b2c3d4-...",
  "sizeBytes": 15234,
  "attachmentCount": 2
}
```

**`type` degerleri:**
| Deger | Aciklama |
|-------|----------|
| `Icerik` | Icerik mesaji (varsayilan) - mesaj + ekler |
| `Gonderi` | Gonderi delili - gonderen KEPHS → alici KEPHS |
| `Kabul` | Kabul delili - alici KEPHS → gonderen KEPHS |
| `Alindi` | Alindi delili - alicinin mesaji okudugu |

---

### 2. KEP Paketi Dogrulama

```
POST /api/v1/kep/verify
Content-Type: application/json
```

**Request:**
```json
{
  "kepPackageBase64": "UEsDBBQACAgI..."
}
```

**Response (200):**
```json
{
  "isValid": true,
  "isSignatureValid": true,
  "isTimestampValid": true,
  "hasRequiredFields": true,
  "signerSubject": "CN=Kurum Imza, O=Kurum Adi, C=TR",
  "signingTime": "2026-02-20T14:30:00Z",
  "timestampTime": "2026-02-20T14:30:01Z",
  "errors": [],
  "warnings": []
}
```

---

### 3. KEP Paketi Icerik Cikarma

```
POST /api/v1/kep/extract
Content-Type: application/json
```

**Request:**
```json
{
  "kepPackageBase64": "UEsDBBQACAgI..."
}
```

**Response (200):**
```json
{
  "kepId": "a1b2c3d4-...",
  "senderAddress": "kurum@kephs.gov.tr",
  "recipientAddresses": ["alici1@kephs.gov.tr"],
  "subject": "Resmi Yazi",
  "body": "Bu bir KEP mesajidir.",
  "sendTime": "2026-02-20T14:30:00Z",
  "type": "Icerik",
  "serviceProvider": "PTT KEP",
  "hasSignature": true,
  "attachments": [
    {
      "fileName": "yazi.pdf",
      "contentBase64": "JVBERi0xLjQ...",
      "sizeBytes": 10240
    }
  ]
}
```

---

## Provider Tipleri

`provider` alaninda 3 farkli imzalama yontemi kullanilabilir:

### Software (PKCS#12 sertifika dosyasi)
```json
{
  "type": "software",
  "certificateBase64": "<pfx dosyasi base64>",
  "certificatePassword": "sifre"
}
```

### Token (Lokal USB/akilli kart)
```json
{
  "type": "token",
  "pin": "123456",
  "pkcs11LibraryPath": "C:/Windows/System32/akisp11.dll",
  "slotId": 0
}
```

### Remote Token (Uzak Agent uzerinden)
```json
{
  "type": "remote-token",
  "agentUrl": "http://192.168.1.100:5555",
  "agentApiKey": "api-key-123",
  "pin": "123456",
  "slotId": 0,
  "certificateIndex": 0
}
```

---

## Yontem 2: NuGet/Proje Referansi (Direkt SDK)

DigiMR SDK'yi dogrudan .NET projesinde kullanmak icin:

**Proje referansi:**
```xml
<ProjectReference Include="..\..\DigiMR\src\DigitalSignature.SDK\DigitalSignature.SDK.csproj" />
```

**Kullanim:**
```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;
using DigitalSignature.Core.Providers;
using DigitalSignature.Xml.Models;

// SDK olustur
var sdk = new DigitalSignatureSDK();

// Provider ile authenticate ol
var provider = new SoftwareCertificateProvider();
await provider.AuthenticateAsync(new AuthenticationContext
{
    CertificateData = pfxBytes,
    CertificatePassword = "sifre"
});

// KEP paketi olustur
var package = new KepPackage
{
    SenderAddress = "kurum@kephs.gov.tr",
    RecipientAddresses = { "alici@kephs.gov.tr" },
    Subject = "Resmi Yazi",
    Body = "Mesaj metni",
    Type = KepType.Icerik,
    ServiceProvider = "PTT KEP",
    Attachments =
    {
        ["belge.pdf"] = pdfBytes,
        ["ek.xlsx"] = excelBytes
    }
};

byte[] kepZip = await sdk.CreateKepPackageAsync(package, provider, "http://zd.kamusm.gov.tr/ts");

// Dogrulama
KepVerificationResult result = sdk.VerifyKepPackage(kepZip);
if (result.IsValid)
{
    Console.WriteLine($"Imzalayan: {result.SignerSubject}");
    Console.WriteLine($"Zaman: {result.TimestampTime}");
}

// Icerik cikarma
KepPackage extracted = sdk.ExtractKepPackage(kepZip);
foreach (var (fileName, data) in extracted.Attachments)
{
    File.WriteAllBytes(fileName, data);
}

provider.Dispose();
```

---

## Ek API Endpoint'leri

Temel create/verify/extract disinda sunulan gelismis endpoint'ler:

### 4. Imzala + KEP Zarfla (Tek Adim)

```
POST /api/v1/kep/sign-and-create
```
Belgeyi once imzalar, sonra KEP zarfina koyar. Iki asamayi tek API cagrisinda birlestirir.

### 5. Imzala + EYP Olustur + KEP Zarfla

```
POST /api/v1/kep/sign-eyp-and-create
```
Belgeyi imzalar, EYP paketine koyar ve KEP zarfiyla sarar. Belge yonetim sistemleri icin ideal.

### 6. Delil Dogrulama

```
POST /api/v1/kep/verify-delil
```
Kabul/Alindi delil paketlerini dogrular. Standart dogrulamaya ek olarak gonderim/teslimat zamani farki, alici eslesmesi ve delil zinciri butunlugu kontrol edilir.

### 7. Toplu KEP Zarfi

```
POST /api/v1/kep/batch-create
```
Ayni belgeleri birden fazla aliciya ayri ayri KEP zarflariyla gonderir. Her alici icin bagimsiz KEP paketi olusturulur.

---

## Hata Durumlari

| HTTP | Durum | Aciklama |
|------|-------|----------|
| 400 | `SenderAddress is required` | Gonderen adresi bos |
| 400 | `At least one recipient address is required` | Alici yok |
| 400 | `TsaUrl is required` | Zaman damgasi URL'si yok |
| 400 | `Invalid base64 format` | Gecersiz base64 verisi |
| 400 | `Document size exceeds limit` | Dosya boyutu limiti asildi |
| 500 | `KEP package creation failed: ...` | Imzalama veya paketleme hatasi |
| 500 | `Provider authentication failed: ...` | Sertifika/PIN hatasi |

---

## Guvenlik Notlari

- Tum eklerin boyutu `Security:MaxDocumentSizeBytes` ile sinirlidir (varsayilan: 50MB)
- Base64 uzunlugu decode oncesinde kontrol edilir (DoS koruması)
- CAdES-T imza zaman damgasi iceriri (7201 sayili Kanun geregi zorunlu)
- Sertifika sifreleri/PIN'ler log'a yazilmaz
- Provider dispose edildikten sonra PIN bellekten silinir

---

## TSA (Zaman Damgasi) Sunuculari

Turkiye'deki yetkilendirilmis ZD sunuculari:
- KamuSM: `http://zd.kamusm.gov.tr/ts`
- E-Guven: `http://zd.e-guven.com/TSS/HttpTspServer`

`tsaUrl` alanini bos birakirsaniz, API konfigurasyonundaki `DefaultTsaUrl` kullanilir.

---

## Test Ortami

DigiMR unit testleri self-signed sertifika ile calismaktadir.
Test ortaminda `http://tsa.test.invalid/ts` URL kullanilabilir;
bu durumda zaman damgasi atlanir ama CAdES-BES imza olusturulur.

Gercek ortamda mutlaka gecerli TSA URL kullanin.
