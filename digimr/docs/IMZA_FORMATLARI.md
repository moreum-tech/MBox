# Imza Formatlari (Signature Formats)

DigiMR SDK, Turkiye'deki nitelikli elektronik imza standartlarina uygun olarak
dort temel imza formatini destekler: **CAdES**, **PAdES**, **XAdES** ve **JAdES**.

> **Not**: Tum orneklerde `provider` bir `ISigningProvider` implementasyonudur
> (SoftwareCertificateProvider, TokenSigningProvider, RemoteTokenSigningProvider, MobileSigningProvider).

---

## CAdES (CMS Advanced Electronic Signatures)
- ETSI TS 319 122-1 / EN 319 122-1
- Binary format, her turlu dosya icin uygun
- Attached ve detached mod

### CAdES Seviyeleri
| Seviye | Aciklama | Unsigned Attributes |
|--------|----------|---------------------|
| B-B (BES) | Temel imza | - |
| B-T | Zaman damgali | signatureTimeStampToken |
| B-LT | Uzun sureli (LTV) | certValues, revocationValues |
| B-LTA | Arsiv | archiveTimestampV3 |
| ES-C | Sertifika referanslari | completeCertificateRefs, completeRevocationRefs |
| ES-X | Genisletilmis referanslar | sigAndRefsTimestamp |

### CAdES Kullanim
```csharp
// CAdES-BES (temel)
var parameters = new SignatureParameters
{
    Format = SignatureFormat.CAdES,
    Level = SignatureLevel.B_B
};
var result = await sdk.SignDataWithProviderAsync(data, provider, parameters);

// CAdES-T (zaman damgali)
parameters.Level = SignatureLevel.B_T;
parameters.TsaUrl = "http://tzd.kamusm.gov.tr";
```

### CAdES Dogrulama
```csharp
var verifier = new CAdESVerifier();
var result = verifier.Verify(cmsSignature);
// or detached:
var result = verifier.VerifyDetached(cmsSignature, originalContent);
```

### CAdES Yukseltme (Upgrade)
```csharp
var builder = new CAdESBuilder(timestampService, crlOcspService);
// B-B → B-T
var upgraded = await builder.AddTimestampAsync(cmsSignature, tsaUrl);
// B-T → B-LT
upgraded = await builder.AddValidationDataAsync(upgraded, chain);
// B-LT → B-LTA
upgraded = await builder.AddArchiveTimestampAsync(upgraded, tsaUrl);
```

Yukseltme siralamasi: `B-B --> B-T --> B-LT --> B-LTA` veya `B-T --> ES-C --> ES-X`

---

## PAdES (PDF Advanced Electronic Signatures)
- ETSI TS 319 142 / EN 319 142
- PDF icine gomulu imza
- DSS (Document Security Store) destegi
- Gorunur ve gorunmez imza

### PAdES Seviyeleri
| Seviye | Aciklama | Ek Ozellikler |
|--------|----------|---------------|
| B-B | Temel PDF imza | Imza alani + CMS |
| B-T | Zaman damgali | CMS icerisinde RFC 3161 |
| B-LT | Uzun sureli | DSS (sertifika, OCSP, CRL) |
| B-LTA | Arsiv | Belge zaman damgasi (ETSI.RFC3161) |

### PAdES Kullanim
```csharp
var parameters = new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_LT,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    Reason = "Belge onayi",
    Location = "Ankara"
};

var result = await sdk.SignPdfWithProviderAsync("document.pdf", provider, parameters);
```

### Gorunur Imza (Visual Signature)
4 farkli mod desteklenir:

```csharp
// Metin imza
parameters.VisualSignature = new VisualSignatureOptions
{
    Mode = VisualSignatureMode.TextOnly,
    Page = 1,
    X = 100, Y = 100,
    Width = 200, Height = 80,
    Text = "Onaylayan: {signer}\nTarih: {date}"
};

// Resim imza
parameters.VisualSignature = new VisualSignatureOptions
{
    Mode = VisualSignatureMode.ImageOnly,
    SignatureImage = File.ReadAllBytes("signature.png"),
    Page = 1,
    X = 100, Y = 100, Width = 200, Height = 80
};

// Damga (Stamp)
parameters.VisualSignature = new VisualSignatureOptions
{
    Mode = VisualSignatureMode.StampOnly,
    StampText = "ONAYLANDI",
    StampStyle = StampStyle.RoundedRectangle,
    StampColor = "#2563eb"
};
```

Yer tutuculari: `{signer}` (imzaci CN), `{date}`, `{reason}`, `{location}`

### PAdES Dogrulama
```csharp
var pdfSigner = new PdfSigner(signatureService, timestampService, crlOcspService);
var result = pdfSigner.VerifyPdfSignatureDetailed(pdfData);

foreach (var sig in result.Signatures)
{
    Console.WriteLine($"Alan: {sig.FieldName}");
    Console.WriteLine($"Seviye: {sig.DetectedLevel}");
    Console.WriteLine($"Gecerli: {sig.IntegrityValid}");
}
```

### PAdES Yukseltme
```csharp
var pdfSigner = new PdfSigner(signatureService, timestampService, crlOcspService);
// B-B/B-T --> B-LT (DSS ekleme)
var upgraded = await pdfSigner.UpgradeToBLTAsync(signedPdfData);
// B-LT --> B-LTA (belge zaman damgasi ekleme)
upgraded = await pdfSigner.UpgradeToBLTAAsync(upgraded);
```

---

## XAdES (XML Advanced Electronic Signatures)
- ETSI EN 319 132
- XML belgeleri icin
- Enveloped ve detached mod

### XAdES Seviyeleri
| Seviye | Aciklama | Ek Elementler |
|--------|----------|---------------|
| BES | Temel XML imza | SignedProperties (SigningTime, SigningCertificateV2) |
| T | Zaman damgali | SignatureTimeStamp |
| XL | Uzun sureli | CertificateValues, RevocationValues |
| A | Arsiv | ArchiveTimeStamp (xades141 namespace) |

### XAdES Kullanim
```csharp
var builder = new XAdESBuilder(hashService, timestampService);

// Enveloped imza
var signedXml = await builder.SignEnvelopedAsync(xmlContent, provider);

// Detached imza
var signedXml = await builder.SignDetachedAsync(data, "document.pdf", provider);
```

### XAdES Dogrulama
```csharp
var verifier = new XAdESVerifier(hashService);
var result = verifier.Verify(signedXml);

Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Seviye: {result.DetectedLevel}");
```

### XAdES Yukseltme
```csharp
var upgradeService = new XAdESUpgradeService(hashService, timestampService, crlOcspService);
// BES --> T
signedXml = await upgradeService.UpgradeToTAsync(signedXml, tsaUrl);
// T --> XL
signedXml = await upgradeService.UpgradeToXLAsync(signedXml, chain);
// XL --> A
signedXml = await upgradeService.UpgradeToAAsync(signedXml, tsaUrl);
```

---

## JAdES (JSON Advanced Electronic Signatures)
- ETSI TS 119 182-1
- JSON/JWS tabanlI imza formati
- REST API ve web servisleri icin ideal

### JAdES Seviyeleri
| Seviye | Aciklama |
|--------|----------|
| B-B | Temel JWS imza (JAdES Baseline) |
| B-T | Zaman damgali |

### JAdES Kullanim
```csharp
// JAdES imza olustur
var parameters = new SignatureParameters
{
    Format = SignatureFormat.JAdES,
    Level = SignatureLevel.B_B
};
var result = await sdk.SignJadesAsync(data, provider, parameters);
```

### JAdES API
```bash
# JAdES imza olustur
curl -X POST http://localhost:7701/api/v1/jades/sign \
  -H "Content-Type: application/json" \
  -d '{
    "dataBase64": "<base64 data>",
    "level": "B_B",
    "provider": { "type": "software", "certificateBase64": "...", "certificatePassword": "..." }
  }'

# JAdES dogrulama
curl -X POST http://localhost:7701/api/v1/jades/verify \
  -H "Content-Type: application/json" \
  -d '{ "jwsBase64": "<base64 JWS>" }'

# JAdES yukseltme (B-B -> B-T)
curl -X POST http://localhost:7701/api/v1/jades/upgrade \
  -H "Content-Type: application/json" \
  -d '{ "jwsBase64": "<base64 JWS>", "tsaUrl": "http://tzd.kamusm.gov.tr" }'
```

---

## ASiC Konteynerleri (ASiC Containers)
- ETSI TS 319 162
- ZIP tabanlI imza konteynerI

### ASiC-S (Simple)
- Tek dosya + tek imza
- mimetype dosyasi ile baslar

```csharp
var asicBuilder = new ASiCBuilder();
var container = await asicBuilder.CreateSimpleAsync(
    documentBytes, "document.pdf", provider, parameters);
```

### ASiC-E (Extended)
- Coklu dosya + manifest
- Manifest imzalanir

```csharp
var files = new Dictionary<string, byte[]>
{
    { "contract.pdf", contractBytes },
    { "appendix.pdf", appendixBytes }
};
var container = await asicBuilder.CreateExtendedAsync(files, provider, parameters);
```

### ASiC Dogrulama
```csharp
var asicVerifier = new ASiCVerifier(cAdESVerifier);
var result = asicVerifier.Verify(containerBytes);
Console.WriteLine($"Tur: {result.ContainerType}");  // Simple veya Extended
```

---

## Gelismis Ozellikler (Advanced Features)

### Imzaci Rolu (Signer Role)
```csharp
parameters.SignerRole = new SignerRoleAttribute
{
    ClaimedRoles = new List<string> { "Mudir", "Onayci" }
};
```

### Commitment Type
```csharp
parameters.CommitmentType = new CommitmentTypeAttribute
{
    Type = CommitmentTypeId.ProofOfApproval
};
```

Desteklenen turler: `ProofOfApproval`, `ProofOfCreation`, `ProofOfDelivery`

### Signature Policy (EPES)
```csharp
parameters.SignaturePolicy = new SignaturePolicy
{
    PolicyId = "1.2.3.4.5",
    PolicyUri = "http://policy.example.com",
    PolicyDigestAlgorithm = "SHA256",
    PolicyDigestValue = policyHash
};
```

### Counter-Signature (Karsi Imza)

Mevcut bir imzanin uzerine ikinci bir imzacinin imzasi eklenir.
CMS yapisinda `countersignature` unsigned attribute olarak saklanir.

```csharp
var counterSigner = new CounterSignatureBuilder();
var counterSigned = await counterSigner.AddCounterSignatureAsync(
    originalCmsSignature, counterSignProvider, parameters);
```

---

## Birlesik Dogrulama (Unified Validation)

Tum imza formatlarini otomatik algilayarak tek bir API ile dogrulama yapar.

```csharp
var validationService = new UnifiedValidationService(
    pdfSigner, cAdESVerifier, xAdESVerifier,
    eypPackageService, eypV13PackageService);

var result = await validationService.ValidateAsync(fileBytes);

Console.WriteLine($"Format: {result.DocumentFormat}");  // PDF, CMS, XML, EYP, ASiC
Console.WriteLine($"Gecerli: {result.IsValid}");

foreach (var sig in result.Signatures)
{
    Console.WriteLine($"  Rol: {sig.Role}");      // Document, Timestamp, Seal
    Console.WriteLine($"  Seviye: {sig.Level}");
    Console.WriteLine($"  Gecerli: {sig.IsValid}");
}
```

---

## Format Karsilastirmasi (Format Comparison)

| Ozellik | CAdES | PAdES | XAdES | JAdES |
|---------|-------|-------|-------|-------|
| Dosya formati | Binary (DER) | PDF | XML | JSON (JWS) |
| Uygun icerik | Her turlu dosya | Yalnizca PDF | XML belgeleri | REST API / Web |
| Gorunur imza | Hayir | Evet | Hayir | Hayir |
| Detached mod | Evet | Hayir (gomulu) | Evet | Evet |
| Arsiv destegi | B-LTA | B-LTA | A | - |
| DSS destegi | Hayir | Evet | Hayir | Hayir |
| ASiC uyumlulugu | Evet | Hayir | Evet | Hayir |
| E-fatura uygunlugu | Evet | Hayir | Evet | Hayir |

### Hangi Formati Secmeliyim?
- **PDF belgeleri** --> PAdES (gorunur imza gerekiyorsa ozellikle)
- **XML/e-fatura** --> XAdES
- **Genel dosyalar** --> CAdES
- **REST API / JSON veri** --> JAdES
- **Coklu dosya paketi** --> ASiC-E (CAdES tabanli)
- **Arsivleme** --> B-LTA veya A seviyesi (tum formatlar)
- **Uzun sureli dogrulama** --> En az B-LT seviyesi

---

## Algoritma Destegi

### Hash Algoritmalari

| Algoritma | Boyut | Kullanim |
|-----------|-------|----------|
| SHA-256 | 256-bit | Varsayilan, tum formatlar |
| SHA-384 | 384-bit | EYP 2.0+ icin zorunlu, arsiv onerilen |
| SHA-512 | 512-bit | Yuksek guvenlik |
| SHA3-256 | 256-bit | Yeni nesil (Keccak tabanli) |
| SHA3-384 | 384-bit | Yeni nesil |
| SHA3-512 | 512-bit | Yeni nesil |

> **Not**: SHA-1 imzalama icin yasaklidir (yalnizca eski belge dogrulamada kabul edilir).

### Imza Algoritmalari

| Algoritma | Hash | Imza |
|-----------|------|------|
| RSA + SHA-256 | SHA-256 | RSA PKCS#1 v1.5 |
| RSA + SHA-384 | SHA-384 | RSA PKCS#1 v1.5 |
| RSA + SHA-512 | SHA-512 | RSA PKCS#1 v1.5 |
| ECDSA + SHA-256 | SHA-256 | ECDSA |
| ECDSA + SHA-384 | SHA-384 | ECDSA |
| ECDSA + SHA-512 | SHA-512 | ECDSA |

> **Oneri**: Yeni imzalar icin en az SHA-256 kullanin. Arsiv amacli imzalarda SHA-384 veya SHA-512 tercih edilmelidir. SHA-3 ailesini gelecek-guvenli sistemlerde degerlendirin.

---

## Ilgili Kaynaklar
- [ETSI EN 319 122 - CAdES](https://www.etsi.org/deliver/etsi_en/319100_319199/31912201/)
- [ETSI EN 319 142 - PAdES](https://www.etsi.org/deliver/etsi_en/319100_319199/31914201/)
- [ETSI EN 319 132 - XAdES](https://www.etsi.org/deliver/etsi_en/319100_319199/31913201/)
- [ETSI TS 319 162 - ASiC](https://www.etsi.org/deliver/etsi_ts/319100_319199/31916201/)
