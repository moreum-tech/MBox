# DigiMR Ornek Kodlar — Senaryo Bazli

Her senaryo icin C# SDK kodu ve REST API curl ornegi verilmistir.

---

## Senaryo 1: PDF'i Token ile Imzala (B-T)

**Amac:** USB token'daki sertifika ile PDF'e zaman damgali imza at.

### SDK (C#)

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;

var sdk = new DigitalSignatureSDK();

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = "123456",
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll",
        SlotId = 0
    });

var result = await sdk.SignPdfWithProviderAsync("sozlesme.pdf", provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    Reason = "Sozlesme onayi",
    Location = "Ankara"
});

if (result.Success)
    File.WriteAllBytes("sozlesme-imzali.pdf", result.SignedData!);
else
    Console.WriteLine($"Hata: {result.Error}");
```

### REST API (curl)

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i sozlesme.pdf)'",
    "format": "PAdES",
    "provider": {
      "type": "token",
      "pin": "123456",
      "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll",
      "slotId": 0
    },
    "parameters": {
      "level": "B_T",
      "tsaProviderKey": "kamusm",
      "reason": "Sozlesme onayi",
      "location": "Ankara"
    }
  }'
```

**Beklenen Sonuc:** `{ "success": true, "signedDocumentBase64": "..." }`
DSS dogrulama: `PAdES-BASELINE-T`

---

## Senaryo 2: Toplu PDF Imzalama (Batch)

**Amac:** 50 fatura PDF'ini tek seferde imzala. Tek PIN girisi, tum belgeler sirayla imzalanir.

### SDK (C#)

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext { Pin = "123456", TokenFilter = "SafeNet" });

var parameters = new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
};

var files = Directory.GetFiles("faturalar/", "*.pdf");
int basarili = 0, basarisiz = 0;

foreach (var file in files)
{
    var result = await sdk.SignPdfWithProviderAsync(file, provider, parameters);
    if (result.Success)
    {
        File.WriteAllBytes($"imzali/{Path.GetFileName(file)}", result.SignedData!);
        basarili++;
    }
    else
    {
        Console.WriteLine($"HATA [{Path.GetFileName(file)}]: {result.Error}");
        basarisiz++;
    }
}

Console.WriteLine($"Toplam: {files.Length}, Basarili: {basarili}, Basarisiz: {basarisiz}");
```

### REST API (curl)

```bash
curl -X POST http://localhost:7701/api/v1/sign/batch \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [
      { "id": "fatura-001", "documentBase64": "'$(base64 -i fatura1.pdf)'" },
      { "id": "fatura-002", "documentBase64": "'$(base64 -i fatura2.pdf)'" },
      { "id": "fatura-003", "documentBase64": "'$(base64 -i fatura3.pdf)'" }
    ],
    "format": "PAdES",
    "provider": {
      "type": "token",
      "pin": "123456",
      "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll"
    },
    "parameters": {
      "level": "B_T",
      "tsaProviderKey": "kamusm"
    }
  }'
```

**Beklenen Sonuc:**
```json
{
  "results": [
    { "id": "fatura-001", "success": true, "signedDocumentBase64": "..." },
    { "id": "fatura-002", "success": true, "signedDocumentBase64": "..." },
    { "id": "fatura-003", "success": true, "signedDocumentBase64": "..." }
  ],
  "summary": { "total": 3, "succeeded": 3, "failed": 0, "elapsedMs": 3200 }
}
```

**Limitler:** Maks 100 belge, toplam 500 MB.

---

## Senaryo 3: Imza Dogrulama

**Amac:** Imzali bir belgeyi dogrula, format ve seviye otomatik algilansin.

### SDK (C#)

```csharp
var result = await sdk.ValidateDocumentAsync("imzali-belge.pdf");

Console.WriteLine($"Format: {result.Format}");           // PAdES
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Butunluk: {result.IsDocumentIntact}");

foreach (var sig in result.Signatures)
{
    Console.WriteLine($"  [{sig.Index}] {sig.SignerSubject}");
    Console.WriteLine($"       Seviye: {sig.Level}");
    Console.WriteLine($"       Rol: {sig.Role}");
    Console.WriteLine($"       Tarih: {sig.SignatureTime}");
    Console.WriteLine($"       Gecerli: {sig.IsValid}");

    foreach (var err in sig.Errors)
        Console.WriteLine($"       HATA: {err}");
}
```

### REST API — Tam dogrulama

```bash
curl -X POST http://localhost:7701/api/v1/verify \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i imzali.pdf)'"}'
```

### REST API — Hedefli kontrol ("PAdES B-T var mi?")

```bash
curl -X POST http://localhost:7701/api/v1/verify/check \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i imzali.pdf)'",
    "format": "PAdES",
    "level": "B_T"
  }'
```

### REST API — Tam envanter

```bash
curl -X POST http://localhost:7701/api/v1/verify/inspect \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i imzali.pdf)'"}'
```

**Beklenen Sonuc (inspect):**
```json
{
  "detectedFormat": "PAdES",
  "validSignatureTypes": ["PAdES-BASELINE-B", "PAdES-BASELINE-T"],
  "signatureInventory": [
    { "index": 0, "signerSubject": "CN=Ahmet Yilmaz", "level": "B_T", "hasTimestamp": true }
  ],
  "summary": "1 gecerli imza bulundu. En yuksek seviye: B-T"
}
```

---

## Senaryo 4: EYP 2.0 Paketi Olustur

**Amac:** Resmi yazi + ekler iceren EYP paketi olustur, imza + e-muhur ekle.

### SDK (C#)

```csharp
// Imzaci (kisisel sertifika)
using var signer = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificatePath = "imzalayici.pfx",
        CertificatePassword = "sifre123"
    });

// Muhur (kurumsal HSM)
using var sealer = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.HSM,
    new AuthenticationContext
    {
        Pin = "hsm-pin",
        Pkcs11LibraryPath = @"C:\hsm\pkcs11.dll"
    });

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
        new() { FileName = "ek-1.pdf", Content = File.ReadAllBytes("ek-1.pdf"),
                MimeType = "application/pdf", EkId = "EK-1" }
    },

    Ustveri = new EypUstveri
    {
        BelgeNo = "2026-001",
        Konu = "Proje Onay Yazisi",
        Tarih = DateTime.UtcNow,
        GuvenlikKodu = "TSD",
        Dil = "tur",
        Olusturan = new EypKurum { Ad = "BT Mudurlugu", KKK = "12345678" }
    },

    NihaiUstveri = new EypNihaiUstveri
    {
        Tarih = DateTime.UtcNow,
        BelgeNo = "2026-001",
        BelgeImzalar = new List<EypImzaBilgi>
        {
            new()
            {
                Imzalayan = new EypImzalayan
                {
                    IlkAdi = "Ahmet", Soyadi = "Yilmaz",
                    Unvan = "Mudur", TCKN = "12345678901"
                },
                Amac = "Onay",
                Tarih = DateTime.UtcNow
            }
        }
    }
};

var result = await sdk.CreateEypPackageV2Async(options, signer, sealer);

if (result.Success)
    File.WriteAllBytes("belge.eyp", result.PackageData!);
```

### REST API (curl)

```bash
curl -X POST http://localhost:7701/api/v1/eyp/create \
  -H "Content-Type: application/json" \
  -d '{
    "coverDocument": {
      "fileName": "ustyazi.pdf",
      "contentBase64": "'$(base64 -i ustyazi.pdf)'",
      "mimeType": "application/pdf"
    },
    "documents": [
      {
        "fileName": "ek-1.pdf",
        "contentBase64": "'$(base64 -i ek-1.pdf)'",
        "mimeType": "application/pdf"
      }
    ],
    "signerProvider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "sifre123"
    },
    "signatureLevel": "B-LT",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

**Beklenen Sonuc:** `{ "success": true, "eypPackageBase64": "..." }`

---

## Senaryo 5: KEP Paketi Olustur ve Dogrula

**Amac:** Kayitli elektronik posta paketi olustur ve dogrula.

### SDK (C#)

```csharp
// Olusturma
var package = new KepPackage
{
    SenderAddress = "kurum@kephs.gov.tr",
    RecipientAddresses = { "alici@kephs.gov.tr" },
    Subject = "Resmi Yazi",
    Body = "Ekteki sozlesme onayiniza sunulmustur.",
    Type = KepType.Icerik,
    ServiceProvider = "PTT KEP",
    Attachments = { ["sozlesme.pdf"] = File.ReadAllBytes("sozlesme.pdf") }
};

byte[] kepZip = await sdk.CreateKepPackageAsync(package, provider, "http://tzd.kamusm.gov.tr");
File.WriteAllBytes("kep-paket.zip", kepZip);

// Dogrulama
KepVerificationResult result = sdk.VerifyKepPackage(kepZip);
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Imzaci: {result.SignerSubject}");
Console.WriteLine($"Zaman: {result.TimestampTime}");

// Icerik cikarma
KepPackage extracted = sdk.ExtractKepPackage(kepZip);
foreach (var (fileName, data) in extracted.Attachments)
    File.WriteAllBytes(fileName, data);
```

### REST API (curl)

```bash
# Olusturma
curl -X POST http://localhost:7701/api/v1/kep/create \
  -H "Content-Type: application/json" \
  -d '{
    "senderAddress": "kurum@kephs.gov.tr",
    "recipientAddresses": ["alici@kephs.gov.tr"],
    "subject": "Resmi Yazi",
    "body": "Ekteki sozlesme.",
    "type": "Icerik",
    "tsaUrl": "http://tzd.kamusm.gov.tr",
    "attachments": [
      { "fileName": "sozlesme.pdf", "contentBase64": "'$(base64 -i sozlesme.pdf)'" }
    ],
    "provider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "sifre"
    }
  }'

# Dogrulama
curl -X POST http://localhost:7701/api/v1/kep/verify \
  -H "Content-Type: application/json" \
  -d '{"kepPackageBase64": "'$(base64 -i kep-paket.zip)'"}'

# Icerik cikarma
curl -X POST http://localhost:7701/api/v1/kep/extract \
  -H "Content-Type: application/json" \
  -d '{"kepPackageBase64": "'$(base64 -i kep-paket.zip)'"}'
```

---

## Senaryo 6: Imza Seviye Yukseltme (B-B -> B-LTA)

**Amac:** Mevcut B-B imzayi adim adim en yuksek seviyeye yukselt.

### SDK (C#)

```csharp
var signedData = File.ReadAllBytes("imzali-bb.pdf");

// B-B -> B-T (zaman damgasi ekle)
var btData = await sdk.UpgradeSignatureAsync(signedData, SignatureLevel.B_T,
    "http://tzd.kamusm.gov.tr");

// B-T -> B-LT (CRL/OCSP gomulu)
var bltData = await sdk.UpgradeSignatureAsync(btData, SignatureLevel.B_LT,
    "http://tzd.kamusm.gov.tr");

// B-LT -> B-LTA (arsiv zaman damgasi)
var bltaData = await sdk.UpgradeSignatureAsync(bltData, SignatureLevel.B_LTA,
    "http://tzd.kamusm.gov.tr");

File.WriteAllBytes("imzali-blta.pdf", bltaData);
```

### REST API (curl)

```bash
# B-B -> B-T
curl -X POST http://localhost:7701/api/v1/upgrade \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i imzali-bb.pdf)'",
    "targetLevel": "B_T",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
  }'
# Yanit: { "success": true, "upgradedDocumentBase64": "...", "previousLevel": "B_B", "newLevel": "B_T" }

# B-T -> B-LT
# ... ayni endpoint, targetLevel: "B_LT"

# B-LT -> B-LTA
# ... ayni endpoint, targetLevel: "B_LTA"
```

---

## Senaryo 7: Iki Asamali Imza (Prepare/Finalize)

**Amac:** PIN sunucuya gitmeden, hash disarida (token, HSM, wallet) imzalansin.

### SDK + Agent Akisi

```
1. Sunucu: prepare -> hash doner
2. Agent/Client: hash'i token'da imzalar
3. Sunucu: finalize -> imzali belge tamamlanir
```

### REST API (curl)

```bash
# Adim 1: Hazirlama
PREPARE=$(curl -s -X POST http://localhost:7701/api/v1/sign/prepare \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "format": "PAdES",
    "certificateBase64": "'$(base64 -i sertifika.cer)'",
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
  }')

JOB_ID=$(echo $PREPARE | jq -r '.jobId')
HASH=$(echo $PREPARE | jq -r '.hashBase64')

echo "Job: $JOB_ID"
echo "Hash: $HASH"

# Adim 2: Hash'i disarida imzala (ornegin OpenSSL ile)
echo -n "$HASH" | base64 -d | openssl rsautl -sign -inkey private.key | base64 > signature.b64

# Adim 3: Tamamlama (5 dakika icinde)
curl -X POST http://localhost:7701/api/v1/sign/finalize \
  -H "Content-Type: application/json" \
  -d '{
    "jobId": "'$JOB_ID'",
    "rawSignatureBase64": "'$(cat signature.b64)'"
  }'
```

**Beklenen Sonuc:** `{ "success": true, "signedDocumentBase64": "..." }`

> `jobId` 5 dakika gecerlidir. Suresi dolunca istek reddedilir.

---

## Senaryo 8: Mobil Imza (Turkcell)

**Amac:** Kullanicinin cep telefonundan SIM kart ile imza onayi al.

### REST API (curl)

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "format": "PAdES",
    "provider": {
      "type": "mobile",
      "msisdn": "905321234567",
      "mobileOperator": "Turkcell",
      "displayText": "Kira sozlesmesi imzalaniyor"
    },
    "parameters": {
      "level": "B_T",
      "tsaProviderKey": "kamusm",
      "reason": "Sozlesme onayi"
    }
  }'
```

### SDK (C#)

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905321234567",
        Metadata = new Dictionary<string, string>
        {
            ["Operator"] = "Turkcell",
            ["DisplayText"] = "Kira sozlesmesi imzalaniyor"
        }
    });

var result = await sdk.SignPdfWithProviderAsync("belge.pdf", provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
```

**Akis:** API istegi -> Operator MSSP'ye SOAP cagrisi -> Kullanici SIM'de PIN girer (maks 120 sn) -> Imzali belge doner.

**Desteklenen operatorler:** `Turkcell`, `Vodafone`, `TurkTelekom`

---

## Senaryo 9: Uzak Token ile Imza (Agent Hub)

**Amac:** Baska bir makinedeki fiziksel token ile imzalama. Ozel anahtar token'dan cikmaz.

### REST API (curl)

```bash
# Yontem 1: remote-token provider ile dogrudan
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "format": "PAdES",
    "provider": {
      "type": "remote-token",
      "agentUrl": "https://uzak-makine:5555",
      "agentApiKey": "gizli-anahtar",
      "pin": "1234",
      "certificateIndex": 0
    },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
  }'

# Yontem 2: Agent Hub uzerinden (SignalR)
# Agent merkezi API'ye baglidir
curl http://localhost:7701/api/v1/admin/agents  # Bagli agent'lari listele

curl -X POST http://localhost:7701/api/v1/admin/agents/{agentId}/sign-hash \
  -H "Content-Type: application/json" \
  -d '{"hashBase64": "...", "hashAlgorithm": "SHA256"}'
```

### SDK (C#)

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.RemoteToken,
    new AuthenticationContext
    {
        AgentUrl = "https://uzak-makine:5555",
        AgentApiKey = "gizli-anahtar",
        Pin = "1234",
        CertificateIndex = 0
    });

var result = await sdk.SignPdfWithProviderAsync("belge.pdf", provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_LT,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
```

---

## Senaryo 10: JAdES Imza Olustur ve Dogrula

**Amac:** JSON tabanlı (JWS) imza olustur ve dogrula.

### REST API (curl)

```bash
# Imzalama
curl -X POST http://localhost:7701/api/v1/jades/sign \
  -H "Content-Type: application/json" \
  -d '{
    "dataBase64": "'$(base64 -i belge.json)'",
    "level": "B_T",
    "tsaUrl": "http://tzd.kamusm.gov.tr",
    "hashAlgorithm": "SHA256",
    "detached": false,
    "provider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "sifre"
    }
  }'

# Dogrulama
curl -X POST http://localhost:7701/api/v1/jades/verify \
  -H "Content-Type: application/json" \
  -d '{"jadesData": "<jws_string>"}'

# Seviye yukseltme (B-B -> B-T)
curl -X POST http://localhost:7701/api/v1/jades/upgrade \
  -H "Content-Type: application/json" \
  -d '{
    "jadesData": "<jws_string>",
    "targetLevel": "B_T",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

---

## Senaryo 11: ASiC Konteyner Olustur

**Amac:** Birden fazla belgeyi tek bir imzali konteyner icinde paketle.

### REST API (curl)

```bash
# ASiC-S (tek belge)
curl -X POST http://localhost:7701/api/v1/asic/create-s \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "fileName": "belge.pdf",
    "provider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "sifre"
    },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
  }'

# ASiC-E (coklu belge)
curl -X POST http://localhost:7701/api/v1/asic/create-e \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [
      { "fileName": "belge1.pdf", "contentBase64": "'$(base64 -i belge1.pdf)'" },
      { "fileName": "belge2.xml", "contentBase64": "'$(base64 -i belge2.xml)'" }
    ],
    "provider": {
      "type": "software",
      "certificateBase64": "'$(base64 -i sertifika.p12)'",
      "certificatePassword": "sifre"
    },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
  }'

# Dogrulama
curl -X POST http://localhost:7701/api/v1/asic/verify \
  -H "Content-Type: application/json" \
  -d '{"packageBase64": "'$(base64 -i belge.asice)'"}'

# Icerik cikarma
curl -X POST http://localhost:7701/api/v1/asic/extract \
  -H "Content-Type: application/json" \
  -d '{"packageBase64": "'$(base64 -i belge.asice)'"}'
```

---

## Senaryo 12: Arsiv Zaman Damgasi Yenileme (Preservation)

**Amac:** B-LTA imzali belgenin arsiv zaman damgasini yenileyerek uzun vadeli gecerliligi uzat.

### REST API (curl)

```bash
# Adim 1: Durum kontrolu
curl -X POST http://localhost:7701/api/v1/preservation/check \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i arsiv-belge.pdf)'"}'
```

**Yanit:**
```json
{
  "needsRenewal": true,
  "oldestArchiveTimestamp": "2021-03-15T10:00:00Z",
  "newestArchiveTimestamp": "2021-03-15T10:00:00Z",
  "currentHashAlgorithm": "SHA256",
  "isHashAlgorithmWeak": false,
  "archiveTimestampCount": 1,
  "recommendation": "Arsiv zaman damgasi 5 yasindan buyuk, yenileme onerilir"
}
```

```bash
# Adim 2: Yenileme
curl -X POST http://localhost:7701/api/v1/preservation/renew \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i arsiv-belge.pdf)'",
    "tsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

**Yanit:** `{ "success": true, "renewedDocumentBase64": "...", "sizeBytes": 145000 }`

```bash
# Adim 3: Kanit zinciri goruntuleme
curl -X POST http://localhost:7701/api/v1/preservation/evidence \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i arsiv-belge.pdf)'"}'
```

**Yanit:**
```json
{
  "chainLength": 2,
  "firstTimestamp": "2021-03-15T10:00:00Z",
  "lastTimestamp": "2026-03-19T10:00:00Z",
  "timestamps": [
    { "index": 0, "genTime": "2021-03-15T10:00:00Z", "hashAlgorithm": "SHA256", "tsaName": "KAMUSM", "isValid": true },
    { "index": 1, "genTime": "2026-03-19T10:00:00Z", "hashAlgorithm": "SHA256", "tsaName": "KAMUSM", "isValid": true }
  ]
}
```

**Oneri:** B-LTA imzali belgelerde her 5 yilda bir arsiv zaman damgasi yenileyin.

---

## Ek: Python ve JavaScript Ornekleri

### Python — PDF Imzalama

```python
import requests, base64

with open('belge.pdf', 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode()

with open('sertifika.p12', 'rb') as f:
    cert_b64 = base64.b64encode(f.read()).decode()

response = requests.post('http://localhost:7701/api/v1/sign', json={
    'documentBase64': pdf_b64,
    'format': 'PAdES',
    'provider': {
        'type': 'software',
        'certificateBase64': cert_b64,
        'certificatePassword': 'sifre'
    },
    'parameters': {
        'level': 'B_T',
        'tsaProviderKey': 'kamusm'
    }
})

result = response.json()
if result['success']:
    with open('imzali.pdf', 'wb') as f:
        f.write(base64.b64decode(result['signedDocumentBase64']))
```

### JavaScript (Node.js) — Dogrulama

```javascript
const fs = require('fs');

const pdfB64 = fs.readFileSync('imzali.pdf').toString('base64');

const response = await fetch('http://localhost:7701/api/v1/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ documentBase64: pdfB64 })
});

const result = await response.json();
console.log(`Gecerli: ${result.isValid}`);
console.log(`Format: ${result.format}`);
for (const sig of result.signatures) {
    console.log(`  Imzaci: ${sig.signerSubject}, Seviye: ${sig.level}`);
}
```

---

*Daha fazla ornek: [EXAMPLES.md](EXAMPLES.md)*
*Tam API referansi: [API_DOCUMENTATION.md](API_DOCUMENTATION.md)*

---

*Copyright 2026 Moreum Tech*
