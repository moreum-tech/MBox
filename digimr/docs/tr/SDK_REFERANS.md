# DigiMR SDK Referansi

**SDK Versiyon:** 2.0
**Son Guncelleme:** 2026-04-17
**Durum:** Stabil
**.NET:** 10.0+
**Lisans:** Proprietary
**Test Kapsami:** 1587+ test basarili

---

## Icindekiler

1. Hizli Baslangic
2. Kurulum ve Lisans
3. TSA Yapilandirmasi
4. Trust Store
5. IDigitalSignatureSDK — Tam Metot Tablosu
6. Imzalama
7. Dogrulama
8. Seviye Yukseltme
9. Zaman Damgasi
10. EYP
11. KEP
12. UETS
13. ASiC
14. JAdES
15. OOXML
16. PDF Sifreleme
17. Provider/Token
18. Mobil Imza
19. Preservation (Arsivleme)
20. Iki Asamali Imza (Prepare/Finalize)
21. Health ve Diagnostics
22. Token Session Yonetimi
23. Sertifika Yardimcilari

## Appendix

- A: Model Siniflari
- B: Enum Tipleri
- C: Result Tipleri
- D: Silinen Metotlar (v1.x → v2.0 Gocus)

---

## 1. Hizli Baslangic

SDK'nin minimum kullanimi: lisans yukle, token provider olustur, PDF imzala.

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;

var sdk = new DigitalSignatureSDK();

// 1. Lisans (yoksa 30 gun deneme modu)
sdk.SetLicenseFile("digimr-sdk.lic");

// 2. TSA yapilandirmasi (opsiyonel, B-T ve ustu seviyeler icin)
// Sifreyi ASLA kod icinde yazmayin. Env var veya secrets store kullanin.
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM(
    url: "http://tzd.kamusm.gov.tr",
    userId: 7521,
    password: Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));

// 3. Token ile provider olustur
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext { Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")! });

// 4. PDF'i byte[] olarak oku ve imzala
var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});

if (result.Success)
    File.WriteAllBytes("belge-imzali.pdf", result.SignedData!);
```

> **Not:** Lisans dosyasi olmadan SDK 30 gun boyunca deneme modunda calisir.
> Deneme suresi sona erdiginde `LicenseException` firlatilir.

---

## 2. Kurulum ve Lisans

### Kurulum

```bash
dotnet add package DigitalSignature.SDK
```

### Lisans

SDK ucretsiz 30 gun deneme moduyla baslar. Sureklilik icin `.lic` dosyasi yuklenmelidir:

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
```

**Lisans metotlari** (concrete class `DigitalSignatureSDK` uzerinde — interface `IDigitalSignatureSDK`'da yer almaz):

- `sdk.SetLicenseFile(path)` — .lic dosya yolundan yukler
- `sdk.SetLicenseJson(jsonString)` — JSON string'i dogrudan yukler
- `sdk.SetLicense(licenseData)` — `LicenseData` objesi ile yukler

**Lisans yerlestirme oncelik sirasi** (programla belirleme disinda otomatik):

1. `SetLicenseFile(path)` cagrisi
2. `DIGIMR_LICENSE_PATH` ortam degiskeni
3. Uygulama dizininde `digimr-sdk.lic`

Deneme suresi sona erdiginde imzalama/dogrulama/yukseltme metotlari `LicenseException` firlatir. Salt dogrulama (`ValidateDocumentAsync`) kisitlamadan calismaya devam eder.

---

## 3. TSA Yapilandirmasi

Zaman damgasi servisi (TSA) yapilandirmasi `ConfigureTsa` ile yapilir. Iki asiri yuk varyanti vardir: tek provider veya coklu provider (fallback).

### Tek Provider

```csharp
// Sifreyi ASLA kod icinde yazmayin. Env var okuyun:
var kamusmPassword = Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")
    ?? throw new InvalidOperationException("KAMUSM_PASSWORD env var tanimlanmamis");

sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM(
    url: "http://tzd.kamusm.gov.tr",
    userId: 7521,
    password: kamusmPassword));
```

**UYARI — Production'da Sifre Yonetimi:** `appsettings.json` (Production) ve `appsettings.Standalone.json` dosyalarinda KamuSM `Password` alani bilerek **bos** birakilir. Gercek sifre asagidaki yollardan saglanmalidir:

- Environment variable: `TsaProviders__Providers__kamusm__Password=<gercek-sifre>`
- .NET User Secrets (gelistirici makinesi)
- Azure Key Vault / AWS Secrets Manager (cloud deployment)

Sifre bos kalirsa identity header atlanir ve KamuSM baglantisi sessizce basarisiz olur. Kontrol: `CheckReadinessAsync()` ile TSA erisilebilirligini dogrulayin.

### Coklu Provider

```csharp
var kamusmPassword = Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!;
var turktrustPassword = Environment.GetEnvironmentVariable("TURKTRUST_PASSWORD")!;

sdk.ConfigureTsa(new TsaProvidersOptions
{
    Default = "kamusm",
    Providers = new Dictionary<string, TsaProviderConfig>
    {
        ["kamusm"] = TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521, kamusmPassword),
        ["freetsa"] = TsaProviderConfig.Anonymous("https://freetsa.org/tsr"),
        ["turktrust"] = TsaProviderConfig.BasicAuth("https://tsp.turktrust.com.tr", "user", turktrustPassword)
    }
});
```

### TsaProviderConfig Factory Metotlari

- `KamuSM(url, userId, password, hashAlgorithm = "SHA256")` — KamuSM (TUBITAK) TSA. `userId` int'tir (MA3 TSSettings uyumlu).
- `BasicAuth(url, username, password, hashAlgorithm = "SHA256")` — HTTP Basic Auth (TURKTRUST, E-Guven)
- `Anonymous(url, hashAlgorithm = "SHA256")` — Kimlik dogrulamasiz (FreeTSA, DigiCert)

### Provider Listesi

```csharp
foreach (var p in sdk.ListTsaProviders().Where(p => p.IsEnabled))
    Console.WriteLine($"{p.Key} ({p.AuthType}): {p.Url}");
```

---

## 4. Trust Store

Dogrulama islemlerinde kullanilan guven deposu. `sdk.TrustStore` property'si (`ITrustedCertificateStore`) uzerinden yonetilir.

### Guvenilir Kok Sertifika Ekleme

```csharp
using Org.BouncyCastle.X509;

// Tek sertifika (X509Certificate objesi)
var parser = new X509CertificateParser();
var cert = parser.ReadCertificate(File.ReadAllBytes("custom-root.crt"));
sdk.TrustStore.AddTrustedRoot(cert);

// PEM dosyasindan tum sertifikalari yukle
sdk.TrustStore.AddTrustedRootsFromPem("trusted-roots.pem");

// Dizindeki tum sertifika dosyalarini tara
sdk.TrustStore.AddTrustedRootsFromDirectory(@"C:\certs\trusted");
```

**Not:** `ValidationOptions.InitialCertificates` ile tek dogrulama icin gecici sertifika/CRL/OCSP de saglanabilir (bkz. Bolum 7).

---

## 5. IDigitalSignatureSDK — Tam Metot Tablosu

Asagidaki tablo `IDigitalSignatureSDK` interface'inin tum metotlarini gruplara ayrilmis halde listeler. Her metot detay icin ilgili bolume bagli.

**Not:** Lisans metotlari (`SetLicenseFile`, `SetLicense`, `SetLicenseJson`) concrete class `DigitalSignatureSDK` uzerinde — interface'de yer almaz. Bkz. Bolum 2 (Kurulum ve Lisans).

| Grup | Metot | Donus | Bolum |
|---|---|---|---|
| Config | `ConfigureTsa(TsaProvidersOptions)` | `void` | 3 |
| Config | `ConfigureTsa(string, TsaProviderConfig)` | `void` | 3 |
| Imzalama | `SignDataWithProviderAsync(byte[], ISigningProvider, SignatureParameters)` | `Task<SignatureResult>` | 6.1 |
| Imzalama | `SignBatchAsync(List<SignBatchItem>, ISigningProvider, SignatureParameters)` | `Task<List<SignatureResult>>` | 6.2 |
| Dogrulama | `ValidateDocumentAsync(byte[])` | `Task<ValidationResult>` | 7.1 |
| Dogrulama | `ValidateDocumentAsync(byte[], ValidationOptions)` | `Task<ValidationResult>` | 7.1 |
| Dogrulama | `ValidateBatchAsync(List<byte[]>)` | `Task<List<ValidationResult>>` | 7.2 |
| Dogrulama | `CheckSignatureTypeAsync(byte[], string, string?)` [YENI] | `Task<SignatureTypeCheckResult>` | 7.3 |
| Dogrulama | `InspectSignaturesAsync(byte[])` [YENI] | `Task<SignatureInspectionResult>` | 7.4 |
| Dogrulama | `ExportSignature(byte[])` | `byte[]` | 7.5 |
| Seviye | `UpgradeSignatureAsync(byte[], SignatureLevel, string?)` | `Task<UpgradeResult>` | 8.1 |
| Seviye | `UpgradeJadesAsync(string, SignatureLevel, string?)` | `Task<string>` | 8.2 |
| TSA | `GetTimestampTokenAsync(byte[], string?)` | `Task<byte[]>` | 9.1 |
| TSA | `GetTimestampTokenFromHashAsync(byte[], string, string?)` | `Task<byte[]>` | 9.2 |
| TSA | `VerifyTimestamp(byte[], byte[]?)` | `TimestampVerificationResult` | 9.3 |
| TSA | `ListTsaProviders()` [YENI] | `List<TsaProviderInfo>` | 9.4 |
| EYP | `CreateEypPackageV21Async(EypCreateOptions, ISigningProvider, ISigningProvider?)` | `Task<EypCreateResult>` | 10.1 |
| EYP | `CreateEypPackageV13Async(...)` | `Task<EypCreateResult>` | 10.2 |
| EYP | `VerifyEyp(byte[])` | `EypVerificationResult` | 10.3 |
| EYP | `ExtractEyp(byte[])` | `EypPackageInfo` | 10.4 |
| EYP | `SignAndCreateEypAsync(...)` | `Task<EypCreateResult>` | 10.5 |
| EYP | `CreateEypPackageV20Async(EypCreateOptions, ISigningProvider, ISigningProvider?)` | `Task<EypCreateResult>` | 10.1 |
| EYP Sifreleme | `EncryptEypPackage(byte[], IReadOnlyList<X509Certificate2>, IReadOnlyList<HedefBilgisi>?)` | `byte[]` | 10.1 |
| EYP Sifreleme | `DecryptEypPackage(byte[], X509Certificate2, RSA)` | `byte[]` | 10.1 |
| KEP | `CreateKepPackageAsync(KepPackage, ISigningProvider, string)` | `Task<byte[]>` | 11.1 |
| KEP | `VerifyKepPackage(byte[])` | `KepVerificationResult` | 11.2 |
| KEP | `ExtractKepPackage(byte[])` | `KepPackage` | 11.3 |
| UETS | `CreateUetsPackageAsync(UetsMessage, ISigningProvider, string)` | `Task<byte[]>` | 12.1 |
| UETS | `VerifyUetsPackage(byte[])` | `UetsVerificationResult` | 12.2 |
| UETS | `ExtractUetsPackage(byte[])` | `UetsMessage` | 12.3 |
| ASiC | `CreateAsicSAsync(byte[], string, ISigningProvider, SignatureParameters?)` | `Task<byte[]>` | 13.1 |
| ASiC | `CreateAsicEAsync(ASiCDocument[], ISigningProvider, SignatureParameters?)` | `Task<byte[]>` | 13.2 |
| ASiC | `VerifyAsic(byte[])` | `ASiCVerificationResult` | 13.3 |
| ASiC | `ExtractAsic(byte[])` | `ASiCContainer` | 13.4 |
| JAdES | `SignJadesAsync(byte[], ISigningProvider, SignatureParameters)` | `Task<string>` | 14.1 |
| JAdES | `VerifyJades(string)` | `JadesVerificationResult` | 14.2 |
| PDF | `EncryptPdf(byte[], string, string?)` | `byte[]` | 16.1 |
| PDF | `DecryptPdf(byte[], string)` | `byte[]` | 16.2 |
| PDF | `IsPdfEncrypted(byte[])` | `bool` | 16.3 |
| Provider | `CreateAndAuthenticateProviderAsync(SigningProviderType, AuthenticationContext)` | `Task<ISigningProvider>` | 17.1 |
| Provider | `ProviderFactory` (property) — özel provider tipi kaydı / alma | `ISigningProviderFactory` | 17 |
| Provider | `GetTokenPinStatus(string, int)` | `TokenPinStatus` | 17.2 |
| Provider | `VerifyTokenPin(string, string, int)` [YENI] | `bool` | 17.3 |
| Provider | `ListSlotCertificates(string, int)` [YENI] | `List<CertificateInfo>` | 17.4 |
| Provider Discovery | `GetProviders()` | `List<ProviderInfo>` | 17.5 |
| Provider Discovery | `ListTokens(string?)` | `List<TokenInfo>` | 17.6 |
| Provider Discovery | `QueryRemoteTokensAsync(string, string)` | `Task<List<TokenInfo>>` | 17.7 |
| Provider Discovery | `VerifyTokenLocation(string, int)` | `bool` | 17.8 |
| Provider Discovery | `ClearTokenCache()` | `void` | 17.9 |
| Preservation | `CheckPreservationStatus(byte[])` | `PreservationStatus` | 19.1 |
| Preservation | `RenewArchiveTimestampAsync(byte[], string?)` | `Task<byte[]>` | 19.2 |
| Preservation | `GetEvidenceRecord(byte[])` | `EvidenceRecord` | 19.3 |
| Iki Asamali | `PrepareSignatureAsync(byte[], SignatureParameters, byte[]?, byte[][]?)` | `Task<SignPrepareResult>` | 20.1 |
| Iki Asamali | `FinalizeSignatureAsync(SignPrepareResult, byte[])` | `Task<SignatureResult>` | 20.2 |
| Health | `CheckHealth()` | `HealthStatus` | 21.1 |
| Health | `CheckReadinessAsync()` [YENI] | `Task<ReadinessStatus>` | 21.2 |
| Session | `CreateTokenSessionAsync(SigningProviderType, AuthenticationContext)` | `Task<TokenSessionInfo>` | 22.1 |
| Session | `ListTokenSessions()` | `List<TokenSessionInfo>` | 22.2 |
| Session | `GetTokenSession(string)` | `TokenSessionInfo` | 22.3 |
| Session | `CloseTokenSessionAsync(string)` | `Task` | 22.4 |
| Session | `ListSessionCertificates(string)` | `List<CertificateInfo>` | 22.5 |
| Sertifika | `LoadCertificate(string, string?)` | `CertificateInfo` | 23.1 |

**Note:** `[YENI]` etiketi 2.0 surumunde eklenen 6 metodu isaretler: `CheckSignatureTypeAsync`, `InspectSignaturesAsync`, `ListTsaProviders`, `VerifyTokenPin`, `ListSlotCertificates`, `CheckReadinessAsync`.

---

## 6. Imzalama

### 6.1 SignDataWithProviderAsync

Format otomatik tespit edilerek veri imzalanir. PAdES (PDF), CAdES (CMS), XAdES (XML), JAdES (JWT), OOXML (Office) destekler. Format `SignatureParameters.Format` ile belirtilir.

```csharp
Task<SignatureResult> SignDataWithProviderAsync(
    byte[] data,
    ISigningProvider provider,
    SignatureParameters parameters);
```

**Parametreler:**
- `data` — Imzalanacak veri (PDF icin PDF bytes, XML icin XML bytes, DOCX icin DOCX bytes vb.)
- `provider` — Onceden authenticate edilmis `ISigningProvider` (bkz. Bolum 17)
- `parameters` — `SignatureParameters`: Format, Level, TsaUrl, HashAlgorithm, Reason, Location, VisualSignature vb.

**Donus:** `SignatureResult`
- `Success: bool`
- `SignedData: byte[]?` — Imzalanmis belge (basariliysa)
- `Message: string?` — Bilgi mesaji
- `Error: string?` — Hata mesaji (basarisizsa)
- `DocumentId: string?` — Toplu imzalamada belge tanimlayicisi

**Ornek: PAdES B-T imzalama:**

```csharp
var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    HashAlgorithm = "SHA256",
    TsaUrl = "http://tzd.kamusm.gov.tr",
    Reason = "Onay",
    Location = "Ankara"
});

if (!result.Success)
    throw new Exception($"Imzalama basarisiz: {result.Error}");

File.WriteAllBytes("belge-imzali.pdf", result.SignedData!);
```

**Olasi hatalar:**
- `LicenseException` — Deneme suresi doldu
- `InvalidDataException` — Veri formati tanimlanamadi veya parametreler gecersiz
- TSA hatalari — `SignatureParameters.Validate()` TSA URL kontrolu yapar; Level B-T ve ustu TSA URL'ini zorunlu kilar
- Provider hatalari — Token iletisimi basarisiz (result.Error'a yansir)

### 6.2 SignBatchAsync

Ayni provider ile coklu belge imzalama. Token'da tek authentication ile N belge imzalanir.

```csharp
Task<List<SignatureResult>> SignBatchAsync(
    List<SignBatchItem> items,
    ISigningProvider provider,
    SignatureParameters parameters);
```

**Parametreler:**
- `items` — `SignBatchItem { DocumentId, Data, FileName }` listesi
- `provider` — Ortak signing provider
- `parameters` — Ortak imza parametreleri (her belge ayni Format/Level ile imzalanir)

**Donus:** `List<SignatureResult>` — Her item icin sonuc, girdi ile ayni sirada. `SignatureResult.DocumentId` girdideki `SignBatchItem.DocumentId` ile eslesir.

**Ornek:**

```csharp
var items = Directory.GetFiles("pdf-inbox/", "*.pdf")
    .Select(p => new SignBatchItem
    {
        DocumentId = Path.GetFileName(p),
        Data = File.ReadAllBytes(p),
        FileName = Path.GetFileName(p)
    })
    .ToList();

var results = await sdk.SignBatchAsync(items, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_B
});

int successCount = 0;
for (int i = 0; i < results.Count; i++)
{
    if (results[i].Success)
    {
        File.WriteAllBytes($"pdf-outbox/{items[i].DocumentId}", results[i].SignedData!);
        successCount++;
    }
    else
    {
        Console.Error.WriteLine($"{items[i].DocumentId}: {results[i].Error}");
    }
}
Console.WriteLine($"{successCount}/{items.Count} belge basariyla imzalandi");
```

## 7. Dogrulama

### 7.1 ValidateDocumentAsync

Imzali belgeyi dogrular. Format otomatik tespit edilir (PAdES, CAdES, XAdES, EYP, OOXML).

```csharp
Task<ValidationResult> ValidateDocumentAsync(byte[] documentData);
Task<ValidationResult> ValidateDocumentAsync(byte[] documentData, ValidationOptions options);
```

**Donus:** `ValidationResult`
- `IsValid: bool` — Genel dogrulama sonucu
- `Format: DocumentFormat` — Tespit edilen format
- `IsDocumentIntact: bool` — Belge icerigi bozulmamis mi
- `Signatures: List<SignatureInfo>` — Bulunan tum imzalar (imzalayan, zaman, level)
- `Errors: List<string>` — Dogrulama hatalari
- `Warnings: List<string>` — Uyarilar
- `Details: string?` — Ek bilgi
- `ValidationState: string?` — "PREMATURE" / "MATURE" / null (grace period kullanildiysa)
- `IsPremature: bool` — Kesinlesme suresi dolmadan dogrulandiysa

**Ornek (basit):**

```csharp
var data = File.ReadAllBytes("imzali.pdf");
var result = await sdk.ValidateDocumentAsync(data);

if (result.IsValid)
    Console.WriteLine($"Gecerli: {result.Signatures.Count} imza, format: {result.Format}");
else
    foreach (var err in result.Errors) Console.WriteLine($"HATA: {err}");
```

**ValidationOptions ile ozel dogrulama** (MA3 EParameters esdegerli alanlar):

```csharp
var result = await sdk.ValidateDocumentAsync(data, new ValidationOptions
{
    // Signing-time attribute'u revokasyon zamani olarak guvenli sayilsin mi? (default: true)
    TrustSigningTimeAttribute = true,

    // Cevrimdisi dogrulama icin onceden saglanan sertifika/CRL/OCSP
    InitialCertificates = new List<byte[]> { File.ReadAllBytes("intermediate-ca.crt") },
    InitialCrls = new List<byte[]> { File.ReadAllBytes("ca.crl") },
    InitialOcspResponses = new List<byte[]>(),

    // Kesinlesme suresi (saniye) — bu surede iptal bildirimi olsa bile imza gecerli sayilir
    GracePeriodSeconds = 86400 // 24 saat
});

if (result.IsPremature)
    Console.WriteLine("Kesinlesme suresi henuz dolmamis, tekrar dogrulama gerekebilir");
```

### 7.2 ValidateBatchAsync

Coklu belge dogrulama.

```csharp
Task<List<ValidationResult>> ValidateBatchAsync(List<byte[]> documents);
```

**Ornek:**

```csharp
var docs = files.Select(File.ReadAllBytes).ToList();
var results = await sdk.ValidateBatchAsync(docs);

var validCount = results.Count(r => r.IsValid);
Console.WriteLine($"{validCount}/{results.Count} gecerli");
```

### 7.3 CheckSignatureTypeAsync  [YENI 2.0]

Belgedeki imzanin beklenen format ve level'a uyup uymadigini kontrol eder — tam dogrulama yapmaz, hizli kontrol icindir (integration test, routing, hazir-kontrol).

```csharp
Task<SignatureTypeCheckResult> CheckSignatureTypeAsync(
    byte[] documentData,
    string expectedFormat,
    string? expectedLevel = null);
```

**Parametreler:**
- `documentData` — Belge bytes
- `expectedFormat` — "PAdES", "CAdES", "XAdES", "JAdES", "OOXML", "ASiC-S", "ASiC-E"
- `expectedLevel` — (opsiyonel) "B-B", "B-T", "B-LT", "B-LTA"

**Donus:** `SignatureTypeCheckResult`
- `Matches: bool` — Format ve (opsiyonel) level uyumlu mu
- `IsValid: bool` — Imzanin kendisi gecerli mi (ayri bir kontrol)
- `DetectedFormat: string?` — Tespit edilen format
- `DetectedLevel: string?` — Tespit edilen level
- `ExpectedFormat: string` — Cagrida verilen format
- `ExpectedLevel: string?` — Cagrida verilen level
- `Reason: string?` — Uymama nedeni (Matches = false ise)

**Ornek:**

```csharp
var check = await sdk.CheckSignatureTypeAsync(data, "PAdES", "B-T");
if (!check.Matches)
    throw new Exception($"Beklenen {check.ExpectedFormat}/{check.ExpectedLevel}, " +
                        $"bulunan {check.DetectedFormat}/{check.DetectedLevel}. " +
                        $"Sebep: {check.Reason}");
```

### 7.4 InspectSignaturesAsync  [YENI 2.0]

Belge icindeki tum imzalarin ayrintili envanteri (imzalayan, zaman, level, hash algoritmasi, rol, hatalar).

```csharp
Task<SignatureInspectionResult> InspectSignaturesAsync(byte[] documentData);
```

**Donus:** `SignatureInspectionResult`
- `Success: bool`
- `DetectedFormat: string?` — Belge formati
- `SignatureCount: int`
- `Signatures: List<SignatureInspectionEntry>` — Her imza icin detay
- `Deficiencies: List<string>` — Paket duzeyinde eksiklikler/uyarilar

**SignatureInspectionEntry:**
- `Index: int` — 0-bazli imza indeksi
- `SignerSubject: string?` — Sertifika DN'si (string olarak, CertificateInfo objesi degil)
- `CertificateSerial: string?`
- `SignatureTime: DateTime?`
- `Level: string?` — "B-B", "B-T", "B-LT", "B-LTA"
- `HashAlgorithm: string?`
- `IsValid: bool`
- `Role: string` — "Document", "Timestamp", "CounterSignature", "Seal"
- `Errors: List<string>`
- `Metadata: Dictionary<string, string>`

**Ornek:**

```csharp
var inspection = await sdk.InspectSignaturesAsync(data);
Console.WriteLine($"Format: {inspection.DetectedFormat}, {inspection.SignatureCount} imza");
foreach (var sig in inspection.Signatures)
{
    Console.WriteLine($"- #{sig.Index} Imzalayan: {sig.SignerSubject}");
    Console.WriteLine($"  Zaman: {sig.SignatureTime:yyyy-MM-dd HH:mm}");
    Console.WriteLine($"  Seviye: {sig.Level}, Hash: {sig.HashAlgorithm}");
    Console.WriteLine($"  Rol: {sig.Role}, Gecerli: {sig.IsValid}");
    if (sig.Errors.Any())
        Console.WriteLine($"  Hatalar: {string.Join("; ", sig.Errors)}");
}
if (inspection.Deficiencies.Any())
    Console.WriteLine($"Paket uyarilari: {string.Join("; ", inspection.Deficiencies)}");
```

### 7.5 ExportSignature

Imzali belgeden ham imza verisini (raw signature bytes) ayiklar. Farkli bir sistemle paylasmak, arsivlemek veya ayri dogrulama yapmak icin kullanilir.

```csharp
byte[] ExportSignature(byte[] signedDocument);
```

**Parametreler:**
- `signedDocument` — Imzali belge (PAdES PDF, CAdES, XAdES vb.)

**Donus:** `byte[]` — Ham imza verisi (format korunur, CMS/PKCS7 bytes)

**Ornek:**

```csharp
var signedPdf = File.ReadAllBytes("imzali.pdf");
var signatureBytes = sdk.ExportSignature(signedPdf);
File.WriteAllBytes("imza.p7s", signatureBytes);
```

---

## 8. Seviye Yukseltme

Mevcut imzalari daha yuksek seviyeye (B-B → B-T → B-LT → B-LTA) cikarir. Yukseltme sirasinda TSA baglantisi gerekir.

### 8.1 UpgradeSignatureAsync

CAdES, PAdES, XAdES icin format-agnostik yukseltme. Format otomatik tespit.

```csharp
Task<UpgradeResult> UpgradeSignatureAsync(
    byte[] documentData,
    SignatureLevel targetLevel,
    string? tsaUrl = null);
```

**Donus:** `UpgradeResult`
- `Success: bool`
- `UpgradedData: byte[]?` — Yukseltilmis belge
- `DetectedFormat: DocumentFormat` — Tespit edilen format
- `PreviousLevel: SignatureLevel?` — Onceki seviye
- `NewLevel: SignatureLevel?` — Yeni seviye
- `Error: string?`

**Ornek: B-B → B-T yukseltme:**

```csharp
var signedData = File.ReadAllBytes("b-seviye.pdf");
var upgrade = await sdk.UpgradeSignatureAsync(
    signedData,
    SignatureLevel.B_T,
    tsaUrl: "http://tzd.kamusm.gov.tr");

if (upgrade.Success)
{
    File.WriteAllBytes("bt-seviye.pdf", upgrade.UpgradedData!);
    Console.WriteLine($"{upgrade.DetectedFormat}: {upgrade.PreviousLevel} → {upgrade.NewLevel}");
}
else
{
    Console.Error.WriteLine($"Yukseltme basarisiz: {upgrade.Error}");
}
```

### 8.2 UpgradeJadesAsync

JAdES icin ayri metot (compact/JSON format; string return).

```csharp
Task<string> UpgradeJadesAsync(
    string jadesData,
    SignatureLevel targetLevel,
    string? tsaUrl = null);
```

**Ornek:**

```csharp
var jadesCompact = File.ReadAllText("imza.jws");
var upgraded = await sdk.UpgradeJadesAsync(jadesCompact, SignatureLevel.B_T, "http://tzd.kamusm.gov.tr");
File.WriteAllText("imza-bt.jws", upgraded);
```

---

## 9. Zaman Damgasi

### 9.1 GetTimestampTokenAsync

Veriye RFC 3161 zaman damgasi al. Yapilandirilmis bir TSA provider key'i ile kullanilir.

```csharp
Task<byte[]> GetTimestampTokenAsync(byte[] data, string? tsaProviderKey = null);
```

**Parametreler:**
- `data` — Zaman damgasi alinacak veri
- `tsaProviderKey` — (opsiyonel) `ConfigureTsa` ile yapilandirilmis provider anahtari. null ise default provider kullanilir.

**Ornek:**

```csharp
var data = Encoding.UTF8.GetBytes("onemli-veri");
var token = await sdk.GetTimestampTokenAsync(data, tsaProviderKey: "kamusm");
File.WriteAllBytes("zaman-damgasi.tsr", token);
```

### 9.2 GetTimestampTokenFromHashAsync

Once hash hesaplanmissa dogrudan hash'e damga al (buyuk dosyalar icin verimli).

```csharp
Task<byte[]> GetTimestampTokenFromHashAsync(
    byte[] hash,
    string hashAlgorithm = "SHA256",
    string? tsaProviderKey = null);
```

**Ornek:**

```csharp
using var sha = System.Security.Cryptography.SHA256.Create();
var hash = sha.ComputeHash(File.OpenRead("buyuk.pdf"));
var token = await sdk.GetTimestampTokenFromHashAsync(hash, "SHA256", "kamusm");
```

### 9.3 VerifyTimestamp

Zaman damgasini dogrula. Orijinal veri verilirse hash eslesmesi de kontrol edilir.

```csharp
TimestampVerificationResult VerifyTimestamp(byte[] timestampToken, byte[]? originalData = null);
```

**Ornek:**

```csharp
var token = File.ReadAllBytes("zaman-damgasi.tsr");
var verification = sdk.VerifyTimestamp(token, originalData: File.ReadAllBytes("veri.bin"));
if (verification.IsValid)
    Console.WriteLine($"Zaman: {verification.GenTime}, TSA: {verification.TsaName}");
```

### 9.4 ListTsaProviders  [YENI 2.0]

Yapilandirilmis TSA provider'larini listeler.

```csharp
List<TsaProviderInfo> ListTsaProviders();
```

**TsaProviderInfo:**
- `Key: string` — Provider anahtari ("kamusm", "freetsa" vb.)
- `Name: string` — Insan-okunabilir ad
- `Url: string`
- `AuthType: string` — "none", "http-basic", "kamusm-identity"
- `IsDefault: bool` — Varsayilan mi
- `IsEnabled: bool` — Aktif mi

**Ornek:**

```csharp
foreach (var p in sdk.ListTsaProviders().Where(p => p.IsEnabled))
    Console.WriteLine($"{p.Key} ({p.AuthType}): {p.Url}" + (p.IsDefault ? " [default]" : ""));
```

---

## 10. EYP (Elektronik Yazismalar Paketi)

TS 13298 V2.0 ve V1.3 standart paketleri. Turkce mevzuat gerekliligi — kurumlar arasi resmi yazisman paketi.

### 10.1 CreateEypPackageV20Async / CreateEypPackageV21Async

```csharp
// EYP 2.0 — mühür OPSIYONEL
Task<EypCreateResult> CreateEypPackageV20Async(EypCreateOptions options, ISigningProvider signerProvider, ISigningProvider? sealProvider = null);
// EYP 2.1 — mühür ZORUNLU (V2.1 default)
Task<EypCreateResult> CreateEypPackageV21Async(EypCreateOptions options, ISigningProvider signerProvider, ISigningProvider? sealProvider = null);
```

> REST `/api/v1/eyp/create` uç noktası `CreateEypPackageV21Async`'i çağırır.

**EypCreateOptions:**
- `Version: EypVersion` — `V2_0` (default)
- `SealMode: EypSealMode` — `WithSeal` / `WithoutSeal` (V2.X icin WithSeal zorunlu)
- `SignatureFormat: EypSignatureFormat` — `CAdES` (V2.X icin tek secek)
- `UstYazi: EypDocumentItem?` — Ust yazi belgesi (opsiyonel)
- `Ekler: List<EypDocumentItem>` — Ek belgeler listesi
- `Ustveri: EypUstveri` — Paket ustveri (BelgeNo, Konu, Tarih, vb.)
- `NihaiUstveri: EypNihaiUstveri` — Nihai ustveri
- `TsaUrl: string?` — Zaman damgasi URL'i
- `ImzaLevel: SignatureLevel` — Imza seviyesi (default: `B_LT`)
- `MuhurLevel: SignatureLevel` — Muhur seviyesi (default: `B_LTA`)

**EypDocumentItem:**
- `FileName: string`
- `Content: byte[]`
- `MimeType: string` — "application/pdf" vb.

**EypCreateResult:**
- `Success: bool`
- `PackageData: byte[]?` — Olusturulan .eyp dosyasi
- `Message: string?`
- `Error: string?`

**Ornek:**

```csharp
var options = new EypCreateOptions
{
    Version = EypVersion.V2_0,
    SealMode = EypSealMode.WithSeal,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    ImzaLevel = SignatureLevel.B_LT,
    MuhurLevel = SignatureLevel.B_LTA,
    Ustveri = new EypUstveri
    {
        BelgeNo = "2024/001",
        Konu = "Proje Teknik Dokumani",
        Tarih = DateTime.UtcNow,
        MimeTuru = "application/pdf"
    },
    UstYazi = new EypDocumentItem
    {
        FileName = "ustyazi.pdf",
        Content = File.ReadAllBytes("ustyazi.pdf"),
        MimeType = "application/pdf"
    },
    Ekler = new List<EypDocumentItem>
    {
        new() { FileName = "ek1.pdf", Content = File.ReadAllBytes("ek1.pdf"), MimeType = "application/pdf" }
    }
};

var result = await sdk.CreateEypPackageV21Async(options, signerProvider, sealProvider);
if (result.Success)
    File.WriteAllBytes("paket.eyp", result.PackageData!);
```

### 10.2 CreateEypPackageV13Async

```csharp
Task<EypCreateResult> CreateEypPackageV13Async(
    EypV13Ustveri ustveri,
    EypDocumentItem? ustYazi,
    List<EypDocumentItem> ekler,
    ISigningProvider signerProvider,
    ISigningProvider? sealProvider = null,
    string? tsaUrl = null,
    EypImzaProfili imzaProfil = EypImzaProfili.P4_CAdES_X_Long,   // imza profili
    EypImzaProfili muhurProfil = EypImzaProfili.P4_CAdES_A,       // mühür profili
    EypV13BelgeHedef? belgeHedef = null,                          // V1.3 belge-hedef (opsiyonel)
    EypV13BelgeImza? belgeImza = null);                           // V1.3 belge-imza (opsiyonel)
```

> Son 4 parametre opsiyoneldir; varsayılan profillerle (`P4_CAdES_X_Long` imza, `P4_CAdES_A` mühür) çağrılabilir.

**EypV13Ustveri (V1.3'e ozgu — Tarih ve BelgeNo burada):**
- `BelgeId: string`
- `BelgeNo: string`
- `Tarih: DateTime`
- `Konu: string`
- `GuvenlikKodu: string` — V1.3: "TSD"|"HZO"|"OZL"|"GZL"|"CGZ"|"KSO" (V2.x: "YOK"|"HZO"|"OZL"|"GZL"|"CGZ"). Geçersiz değer İmzager'da paketi "Tanımlanmamış" gösterir.
- `MimeTuru: string` — "application/pdf" vb.
- `Dil: string` — "tur" (default)
- `Olusturan: EypKurum?`
- `DagitimListesi: List<EypDagitim>`

**Ornek:**

```csharp
var ustveri = new EypV13Ustveri
{
    BelgeNo = "2024/100",
    Konu = "Bilgi Notu",
    Tarih = DateTime.UtcNow,
    GuvenlikKodu = "TSD",   // V1.3 geçerli kod (Tasnif Dışı); V2.x'te "YOK" kullanılır
    MimeTuru = "application/pdf"
};

var ustYazi = new EypDocumentItem
{
    FileName = "bilgi-notu.pdf",
    Content = File.ReadAllBytes("bilgi-notu.pdf"),
    MimeType = "application/pdf"
};

var result = await sdk.CreateEypPackageV13Async(
    ustveri, ustYazi, ekler: new List<EypDocumentItem>(),
    signerProvider, sealProvider, tsaUrl: "http://tzd.kamusm.gov.tr");

if (result.Success)
    File.WriteAllBytes("paket-v13.eyp", result.PackageData!);
```

### 10.3 VerifyEyp

```csharp
EypVerificationResult VerifyEyp(byte[] eypData);
```

Versiyon otomatik tespit (V1.3, V2.0, V2.1).

**EypVerificationResult:**
- `IsValid: bool`
- `IsPackageStructureValid: bool`
- `IsPaketOzetiValid: bool`
- `IsNihaiOzetValid: bool`
- `IsImzaValid: bool`
- `IsMuhurValid: bool`
- `AreHashesValid: bool`
- `ImzaLevel: SignatureLevel`
- `MuhurLevel: SignatureLevel`
- `ImzaciSubject: string?`
- `MuhurSubject: string?`
- `PackageInfo: EypPackageInfo?`
- `Errors: List<string>`

**Ornek:**

```csharp
var eypData = File.ReadAllBytes("paket.eyp");
var result = sdk.VerifyEyp(eypData);
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Imzaci: {result.ImzaciSubject}");
Console.WriteLine($"Muhur: {result.MuhurSubject}");
if (!result.IsValid)
    result.Errors.ForEach(Console.WriteLine);
```

### 10.4 ExtractEyp

```csharp
EypPackageInfo ExtractEyp(byte[] eypData);
```

**EypPackageInfo:**
- `PackageId: string`
- `CreatedAt: DateTime`
- `UstYazi: EypDocumentItem?`
- `Ekler: List<EypDocumentItem>`
- `Ustveri: EypUstveri`
- `NihaiUstveri: EypNihaiUstveri?`
- `PackageData: byte[]?`

**Ornek:**

```csharp
var info = sdk.ExtractEyp(File.ReadAllBytes("paket.eyp"));
Console.WriteLine($"BelgeNo: {info.Ustveri.BelgeNo}, Konu: {info.Ustveri.Konu}");
if (info.UstYazi != null)
    File.WriteAllBytes(info.UstYazi.FileName, info.UstYazi.Content);
foreach (var ek in info.Ekler)
    File.WriteAllBytes(ek.FileName, ek.Content);
```

### 10.5 SignAndCreateEypAsync

Tek cagrida belgeleri imzalar ve EYP paketi olusturur.

```csharp
Task<EypCreateResult> SignAndCreateEypAsync(
    EypCreateOptions options,
    ISigningProvider signerProvider,
    ISigningProvider documentSignerProvider,
    ISigningProvider? sealProvider = null);
```

**Parametreler:**
- `options` — `CreateEypPackageV21Async` ile ayni `EypCreateOptions`
- `signerProvider` — Paket imzaci (NihaiUstveri imzasi)
- `documentSignerProvider` — Belge imzaci (UstYazi/Ekler icin)
- `sealProvider` — Muhur (opsiyonel)

**Ornek:**

```csharp
var options = new EypCreateOptions
{
    Version = EypVersion.V2_0,
    SealMode = EypSealMode.WithSeal,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    Ustveri = new EypUstveri { BelgeNo = "2024/055", Konu = "Sozlesme" },
    UstYazi = new EypDocumentItem
    {
        FileName = "sozlesme.pdf",
        Content = File.ReadAllBytes("sozlesme.pdf"),
        MimeType = "application/pdf"
    }
};

var result = await sdk.SignAndCreateEypAsync(options, signerProvider, signerProvider, sealProvider);
if (result.Success)
    File.WriteAllBytes("sozlesme-imzali.eyp", result.PackageData!);
```

---

## 11. KEP (Kayitli Elektronik Posta)

7201 sayili Kanun ve KEP Yonetmeligi uyarinca yapılandırılmıs e-posta paketi. ZIP: kep-bilgi.xml + CAdES-T imza + ekler.

### 11.1 CreateKepPackageAsync

```csharp
Task<byte[]> CreateKepPackageAsync(KepPackage package, ISigningProvider provider, string tsaUrl);
```

**KepPackage:**
- `Id: string` — KEP benzersiz ID (UUID, otomatik)
- `SenderAddress: string` — Gonderen KEP adresi
- `RecipientAddresses: List<string>` — Alici KEP adresleri
- `Subject: string` — Konu
- `SendTime: DateTime` — Gonderim zamani (UTC)
- `DeliveryTime: DateTime?` — Teslim zamani (opsiyonel)
- `Type: KepType` — `Gonderi`, `Kabul`, `Alindi`, `Icerik`
- `ServiceProvider: string?` — KEPHS adi
- `Attachments: Dictionary<string, byte[]>` — Ek dosyalar (ad → icerik)
- `Body: string?` — Metin govde

**Ornek:**

```csharp
var kep = new KepPackage
{
    SenderAddress = "gonderen@firma.kep.tr",
    RecipientAddresses = new List<string> { "alici@kurum.kep.tr" },
    Subject = "Sozlesme Gonderimi",
    SendTime = DateTime.UtcNow,
    Type = KepType.Gonderi,
    Body = "Ekteki sozlesmeyi bilgilerinize sunarim.",
    Attachments = new Dictionary<string, byte[]>
    {
        ["sozlesme.pdf"] = File.ReadAllBytes("sozlesme.pdf")
    }
};

var kepZip = await sdk.CreateKepPackageAsync(kep, provider, "http://tzd.kamusm.gov.tr");
File.WriteAllBytes("gonderim.kep", kepZip);
```

### 11.2 VerifyKepPackage

```csharp
KepVerificationResult VerifyKepPackage(byte[] kepData);
```

**KepVerificationResult:**
- `IsValid: bool`
- `IsSignatureValid: bool`
- `IsTimestampValid: bool`
- `HasRequiredFields: bool`
- `SignerSubject: string?`
- `SigningTime: DateTime?`
- `TimestampTime: DateTime?`
- `Errors: List<string>`
- `Warnings: List<string>`

**Ornek:**

```csharp
var result = sdk.VerifyKepPackage(File.ReadAllBytes("gonderim.kep"));
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Imzaci: {result.SignerSubject}");
Console.WriteLine($"Zaman Damgasi: {result.TimestampTime}");
```

### 11.3 ExtractKepPackage

```csharp
KepPackage ExtractKepPackage(byte[] kepData);
```

**Ornek:**

```csharp
var kep = sdk.ExtractKepPackage(File.ReadAllBytes("gonderim.kep"));
Console.WriteLine($"Gonderen: {kep.SenderAddress}");
Console.WriteLine($"Konu: {kep.Subject}");
foreach (var (ad, icerik) in kep.Attachments)
    File.WriteAllBytes(ad, icerik);
```

---

## 12. UETS (Ulusal Elektronik Tebligat Sistemi)

PTT UETS altyapisi uzerinden elektronik tebligat. ZIP: META-INF/uets-bilgi.xml + META-INF/imza.p7s + ekler.

### 12.1 CreateUetsPackageAsync

```csharp
Task<byte[]> CreateUetsPackageAsync(UetsMessage message, ISigningProvider provider, string tsaUrl);
```

**UetsMessage:**
- `TebligatId: string` — Tekil tebligat ID (UUID, otomatik)
- `GonderimZamani: DateTime` — Gonderim zamani (UTC)
- `TeslimZamani: DateTime?` — Teslim zamani (PTT tarafindan doldurulur)
- `TebligatTuru: UetsTebligatTuru` — `Gonderim`, `Iade`, `Bildirim`, `Dogrulama`
- `BirimKodu: string?` — Gonderen kurum kodu (opsiyonel)
- `Gonderen: UetsTaraf` — Gonderen taraf bilgileri
- `Muhatap: UetsTaraf` — Alici taraf bilgileri
- `Konu: string` — Tebligat konusu
- `Aciklama: string?` — Ek not (opsiyonel)
- `Attachments: Dictionary<string, byte[]>` — Ek dosyalar

**UetsTaraf:**
- `UetsAdresi: string` — orn: `ahmetyilmaz@vatandas.uets.ptt.gov.tr`
- `AdSoyad: string`
- `TCKimlikNo: string?` — Gercek kisi icin
- `VKN: string?` — Tuzel kisi icin
- `MuhatapTuru: UetsMuhatapTuru` — `GercekKisi` / `TuzelKisi`

**Ornek:**

```csharp
var mesaj = new UetsMessage
{
    TebligatTuru = UetsTebligatTuru.Gonderim,
    Konu = "Vergi Cezasi Tebligati",
    Gonderen = new UetsTaraf
    {
        UetsAdresi = "vd.istanbul@idare.uets.ptt.gov.tr",
        AdSoyad = "Istanbul Vergi Dairesi",
        VKN = "1234567890",
        MuhatapTuru = UetsMuhatapTuru.TuzelKisi
    },
    Muhatap = new UetsTaraf
    {
        UetsAdresi = "ahmetyilmaz@vatandas.uets.ptt.gov.tr",
        AdSoyad = "Ahmet Yilmaz",
        TCKimlikNo = "12345678901",
        MuhatapTuru = UetsMuhatapTuru.GercekKisi
    },
    Attachments = new Dictionary<string, byte[]>
    {
        ["ceza-ihbarnamesi.pdf"] = File.ReadAllBytes("ceza-ihbarnamesi.pdf")
    }
};

var paket = await sdk.CreateUetsPackageAsync(mesaj, provider, "http://tzd.kamusm.gov.tr");
File.WriteAllBytes("tebligat.uets", paket);
```

### 12.2 VerifyUetsPackage

```csharp
UetsVerificationResult VerifyUetsPackage(byte[] packageData);
```

**UetsVerificationResult:**
- `IsValid: bool`
- `IsSignatureValid: bool`
- `IsTimestampValid: bool`
- `HasRequiredFields: bool`
- `SignerSubject: string?`
- `SigningTime: DateTime?`
- `TimestampTime: DateTime?`
- `Errors: List<string>`
- `Warnings: List<string>`

**Ornek:**

```csharp
var result = sdk.VerifyUetsPackage(File.ReadAllBytes("tebligat.uets"));
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Imzaci: {result.SignerSubject}");
if (!result.IsValid)
    result.Errors.ForEach(Console.WriteLine);
```

### 12.3 ExtractUetsPackage

```csharp
UetsMessage ExtractUetsPackage(byte[] packageData);
```

**Ornek:**

```csharp
var mesaj = sdk.ExtractUetsPackage(File.ReadAllBytes("tebligat.uets"));
Console.WriteLine($"TebligatId: {mesaj.TebligatId}");
Console.WriteLine($"Muhatap: {mesaj.Muhatap.AdSoyad} ({mesaj.Muhatap.UetsAdresi})");
foreach (var (ad, icerik) in mesaj.Attachments)
    File.WriteAllBytes(ad, icerik);
```

---

## 13. ASiC (Associated Signature Container)

ETSI EN 319 162 standardı. ASiC-S (tek belge) ve ASiC-E (coklu belge) container formatlari.

### 13.1 CreateAsicSAsync

```csharp
Task<byte[]> CreateAsicSAsync(byte[] document, string fileName, ISigningProvider provider, SignatureParameters? parameters = null);
```

Tek belge + CAdES detached imza. Cikti: `.asics` ZIP dosyasi.

**Ornek:**

```csharp
var container = await sdk.CreateAsicSAsync(
    document: File.ReadAllBytes("belge.pdf"),
    fileName: "belge.pdf",
    provider: provider,
    parameters: new SignatureParameters
    {
        Format = SignatureFormat.ASiC_S,
        Level = SignatureLevel.B_T,
        TsaUrl = "http://tzd.kamusm.gov.tr"
    });
File.WriteAllBytes("belge.asics", container);
```

### 13.2 CreateAsicEAsync

```csharp
Task<byte[]> CreateAsicEAsync(ASiCDocument[] documents, ISigningProvider provider, SignatureParameters? parameters = null);
```

Coklu belge + CAdES detached + manifest. Cikti: `.asice` ZIP dosyasi.

**ASiCDocument:**
- `FileName: string`
- `Content: byte[]` — (NOT `Data`)
- `MimeType: string` — "application/pdf" vb. (default: "application/octet-stream")

**Ornek:**

```csharp
var docs = new[]
{
    new ASiCDocument { FileName = "belge1.pdf", Content = File.ReadAllBytes("belge1.pdf"), MimeType = "application/pdf" },
    new ASiCDocument { FileName = "belge2.pdf", Content = File.ReadAllBytes("belge2.pdf"), MimeType = "application/pdf" }
};

var container = await sdk.CreateAsicEAsync(docs, provider, new SignatureParameters
{
    Format = SignatureFormat.ASiC_E,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
File.WriteAllBytes("paket.asice", container);
```

### 13.3 VerifyAsic

```csharp
ASiCVerificationResult VerifyAsic(byte[] containerData);
```

**Ornek:**

```csharp
var result = sdk.VerifyAsic(File.ReadAllBytes("paket.asice"));
Console.WriteLine($"Gecerli: {result.IsValid}");
```

### 13.4 ExtractAsic

```csharp
ASiCContainer ExtractAsic(byte[] containerData);
```

**ASiCContainer:**
- `Type: ASiCType` — `ASiC_S` veya `ASiC_E`
- `MimeType: string`
- `Documents: List<ASiCDocument>` — Imzalanmis belgeler (`Content` property)
- `Signatures: List<ASiCSignature>` — Imza dosyalari (.p7s)
- `Manifests: List<ASiCManifest>` — Manifest dosyalari (sadece ASiC-E)

**Ornek:**

```csharp
var container = sdk.ExtractAsic(File.ReadAllBytes("paket.asice"));
Console.WriteLine($"Tip: {container.Type}, Belge sayisi: {container.Documents.Count}");
foreach (var doc in container.Documents)
    File.WriteAllBytes(doc.FileName, doc.Content);
```

---

## 14. JAdES (JSON Advanced Electronic Signature)

RFC 7515 JWT-tabanli imza, JWS (JSON Web Signature) compact/serialized format.

### 14.1 SignJadesAsync

```csharp
Task<string> SignJadesAsync(byte[] data, ISigningProvider provider, SignatureParameters parameters);
```

Cikti: JWS string (header.payload.signature compact format).

**Ornek:**

```csharp
var data = File.ReadAllBytes("api-response.json");
var jadesCompact = await sdk.SignJadesAsync(data, provider, new SignatureParameters
{
    Format = SignatureFormat.JAdES,
    Level = SignatureLevel.B_B,
    HashAlgorithm = "SHA256"
});
File.WriteAllText("imza.jws", jadesCompact);
```

### 14.2 VerifyJades

```csharp
JadesVerificationResult VerifyJades(string jadesData);
```

**JadesVerificationResult:**
- `IsValid: bool`
- `SignerSubject: string?` — Imzaci sertifika subject
- `SigningTime: DateTime?` — `sigT` header'dan
- `Level: SignatureLevel?` — Tespit edilen imza seviyesi
- `Errors: List<string>`
- `Warnings: List<string>`

**Ornek:**

```csharp
var jwsString = File.ReadAllText("imza.jws");
var result = sdk.VerifyJades(jwsString);
Console.WriteLine($"Gecerli: {result.IsValid}");
Console.WriteLine($"Imzaci: {result.SignerSubject}");
Console.WriteLine($"Imzalama Zamani: {result.SigningTime}");
if (!result.IsValid)
    result.Errors.ForEach(Console.WriteLine);
```

---

## 15. OOXML (Office Open XML)

Word (docx), Excel (xlsx), PowerPoint (pptx) imzalama ve dogrulama.

**Not:** OOXML icin ayri SDK metodu YOKTUR. `SignDataWithProviderAsync` + `Format = SignatureFormat.OOXML` ile imzalanir, `ValidateDocumentAsync` ile dogrulanir (format otomatik tespit). Sonraki surumlerle onceki `SignOoxmlAsync`/`VerifyOoxml` metotlari kaldirilmistir.

### Imzalama

```csharp
var docxData = File.ReadAllBytes("sozlesme.docx");
var result = await sdk.SignDataWithProviderAsync(docxData, provider, new SignatureParameters
{
    Format = SignatureFormat.OOXML,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});

if (result.Success)
    File.WriteAllBytes("sozlesme-imzali.docx", result.SignedData!);
```

### Dogrulama

```csharp
var data = File.ReadAllBytes("sozlesme-imzali.docx");
var verification = await sdk.ValidateDocumentAsync(data);
Console.WriteLine($"Format: {verification.Format}, Gecerli: {verification.IsValid}");
```

---

## 16. PDF Sifreleme

Imzadan bagimsiz PDF sifreleme/cozme. Imzalanan PDF sifreli olmamalidir — once coz, sonra imzala.

### 16.1 EncryptPdf

```csharp
byte[] EncryptPdf(byte[] pdfData, string ownerPassword, string? userPassword = null);
```

**Parametreler:**
- `pdfData` — Sifrelenecek PDF
- `ownerPassword` — Sahip sifresi (tum islemler icin)
- `userPassword` — Kullanici sifresi (opsiyonel; null ise sadece ownerPassword istenir)

**Ornek:**

```csharp
var pdf = File.ReadAllBytes("gizli.pdf");
var ownerPwd = Environment.GetEnvironmentVariable("PDF_OWNER_PWD")!;
var userPwd = Environment.GetEnvironmentVariable("PDF_USER_PWD")!;
var encrypted = sdk.EncryptPdf(pdf, ownerPwd, userPwd);
File.WriteAllBytes("gizli-encrypted.pdf", encrypted);
```

### 16.2 DecryptPdf

```csharp
byte[] DecryptPdf(byte[] encryptedPdf, string password);
```

**Ornek:**

```csharp
var pwd = Environment.GetEnvironmentVariable("PDF_OWNER_PWD")!;
var decrypted = sdk.DecryptPdf(File.ReadAllBytes("gizli-encrypted.pdf"), pwd);
```

### 16.3 IsPdfEncrypted

```csharp
bool IsPdfEncrypted(byte[] pdfData);
```

**Ornek:**

```csharp
if (sdk.IsPdfEncrypted(pdf))
    Console.WriteLine("Bu PDF sifreli — imzalamadan once cozulmelidir");
```

---

## 17. Provider/Token

Imzalama saglayicilarinin olusturulmasi, PIN yonetimi ve kesif.

### 17.1 CreateAndAuthenticateProviderAsync

```csharp
Task<ISigningProvider> CreateAndAuthenticateProviderAsync(
    SigningProviderType type,
    AuthenticationContext context);
```

**SigningProviderType** (tam enum):
- `Software` — PKCS#12 yazilim sertifika (.pfx/.p12)
- `Token` — PKCS#11 donanim token / akilli kart
- `Mobile` — Mobil imza (Turkcell, Vodafone, Turk Telekom)
- `Biometric` — Biyometrik imza (imza pad, parmak izi)
- `ESeal` — Elektronik muhur (sunucu tarafli)
- `RemoteToken` — Uzak Token Agent uzerinden PKCS#11
- `HSM` — HSM Enterprise (session pooling, multi-slot)
- `CloudHSM` — Azure Key Vault, AWS KMS, Google Cloud KMS

**Ornek (Token):**

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")!,
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll",
        SlotId = 0
    });
```

**Ornek (Software / PKCS#12):**

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("sertifika.pfx"),
        CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
    });
```

**Ornek (Mobile imza):**

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905XX1234567",
        Otp = userEnteredOtp  // kullanici girisi
    });
```

### 17.2 GetTokenPinStatus

Token PIN durumunu kontrol et (kilitli mi, son deneme mi). Login denemeden once kullanilmali.

```csharp
TokenPinStatus GetTokenPinStatus(string pkcs11LibraryPath, int slotId = 0);
```

**TokenPinStatus:**
- `IsOk: bool` — PIN durumu normal
- `IsLocked: bool` — PIN kilitli, login denenmemeli
- `IsFinalTry: bool` — Son deneme hakki, dikkat
- `IsCountLow: bool` — Kalan deneme hakki az
- `Message: string?` — Kullaniciya gosterilecek mesaj
- `HasProtectedAuthPath: bool` — PIN pad veya biyometrik PIN girisi destegi
- `SupportsAlwaysAuthenticate: bool` — Her imzada ek PIN gerektiren private key

**Ornek:**

```csharp
var status = sdk.GetTokenPinStatus(@"C:\Windows\System32\eTPKCS11.dll");
if (status.IsLocked)
    throw new Exception("Token kilitli — PUK ile cozun");
if (status.IsFinalTry)
    Console.WriteLine("DIKKAT: Son deneme hakki!");
```

### 17.3 VerifyTokenPin  [YENI 2.0]

PIN dogrulugunu test et — imzalama yapmadan. Session acmaz, pin-pad destekler.

```csharp
bool VerifyTokenPin(string libraryPath, string pin, int slotId = 0);
```

**Dikkat:** Yanlis PIN denemesi token'i kilitleyebilir. Once `GetTokenPinStatus` ile deneme hakkini kontrol edin.

**Ornek:**

```csharp
var status = sdk.GetTokenPinStatus(libPath);
if (status.IsLocked || status.IsFinalTry)
{
    Console.Error.WriteLine($"Kilit riski: {status.Message}");
    return;
}
if (sdk.VerifyTokenPin(libPath, userEnteredPin))
    Console.WriteLine("PIN dogru");
else
    Console.WriteLine("PIN yanlis — tekrar deneyin");
```

### 17.4 ListSlotCertificates  [YENI 2.0]

Token'daki sertifikalari listele (imzalamadan, PIN gerektirmeden, public cert objects).

```csharp
List<CertificateInfo> ListSlotCertificates(string libraryPath, int slotId = 0);
```

**Ornek:**

```csharp
var certs = sdk.ListSlotCertificates(@"C:\Windows\System32\eTPKCS11.dll");
foreach (var c in certs)
    Console.WriteLine($"{c.Subject} — gecerli: {c.NotBefore:d} - {c.NotAfter:d}");
```

### 17.5 GetProviders

Sistemde mevcut imza saglayicilarinin listesi.

```csharp
List<ProviderInfo> GetProviders();
```

**ProviderInfo:**
- `Type: SigningProviderType`
- `Name: string`
- `Description: string`
- `Version: string?`
- `SupportsHashSigning: bool` — Hash'ten (2-asamali) imza destegi
- `SupportedAlgorithms: List<string>` — Ornegin ["SHA256", "SHA384", "SHA512"]

**Ornek:**

```csharp
foreach (var p in sdk.GetProviders())
    Console.WriteLine($"{p.Type}: {p.Name} ({p.Description}) v{p.Version}");
```

### 17.6 ListTokens

PKCS#11 token'larini listele. `libraryPath` null ise sistemin varsayilan PKCS#11 kutuphaneleri taranir.

```csharp
List<TokenInfo> ListTokens(string? libraryPath = null);
```

**TokenInfo:**
- `SlotId: int`
- `Label: string`
- `Manufacturer: string`
- `Model: string`
- `SerialNumber: string`
- `IsPresent: bool`
- `LibraryPath: string?`

**Ornek:**

```csharp
var tokens = sdk.ListTokens(@"C:\Windows\System32\eTPKCS11.dll");
foreach (var t in tokens.Where(t => t.IsPresent))
    Console.WriteLine($"Slot {t.SlotId}: {t.Label} ({t.Manufacturer} {t.Model}, SN: {t.SerialNumber})");
```

### 17.7 QueryRemoteTokensAsync

Uzak Token Agent sunucusundaki token'lari sorgula.

```csharp
Task<List<TokenInfo>> QueryRemoteTokensAsync(string agentUrl, string apiKey);
```

**Ornek:**

```csharp
var remoteTokens = await sdk.QueryRemoteTokensAsync(
    "https://agent.example.com",
    Environment.GetEnvironmentVariable("AGENT_API_KEY")!);
```

### 17.8 VerifyTokenLocation

Daha onceden discovery cache'ine kaydedilmis bir (libraryPath, slotId) hala mevcut mu?
UI tarafinda hizli ping icin kullanilir; cache miss durumunda `false` doner.

```csharp
bool VerifyTokenLocation(string libraryPath, int slotId);
```

**Ornek:**

```csharp
if (!sdk.VerifyTokenLocation("C:/Windows/System32/eTPKCS11.dll", 0))
    sdk.ClearTokenCache();   // cache stale → bosalt
```

### 17.9 ClearTokenCache

Token discovery cache'ini bosaltir. PKCS#11 middleware veya cihaz degisiminden sonra
cagrilmali; aksi halde stale (libraryPath, slotId) eslesmeleri donebilir.

```csharp
void ClearTokenCache();
```

---

## 18. Mobil Imza

SIM-tabanli mobil imza (Turkcell, Vodafone, Turk Telekom). Kullanici telefonunda OTP/PIN onaylar, sertifika operator tarafinda.

### Kullanim

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905XX1234567",
        Otp = userEnteredOtp  // kullanici girisi
    });

var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
```

Detayli is akisi icin `ORNEKLER.md` ileri senaryolar bolumune bakin.

---

## 19. Preservation (Arsivleme)

Uzun donem arsiv (LTA) imzalari icin arsiv-zaman-damgasi yenileme ve kanit (evidence) kayitlari.

### 19.1 CheckPreservationStatus

```csharp
PreservationStatus CheckPreservationStatus(byte[] signedData);
```

Arsiv imzasinin yenileme ihtiyaci varsa isaretler.

### 19.2 RenewArchiveTimestampAsync

```csharp
Task<byte[]> RenewArchiveTimestampAsync(byte[] signedData, string? tsaUrl = null);
```

B-LTA imzasinin arsiv zaman damgasini yeniler (TSA sertifikasi sureli; yenileme gerekir).

**Ornek (LTA yenileme):**

```csharp
var status = sdk.CheckPreservationStatus(ltaData);
if (status.RenewalNeeded)
{
    var renewed = await sdk.RenewArchiveTimestampAsync(ltaData, "http://tzd.kamusm.gov.tr");
    File.WriteAllBytes("belge-renewed.pdf", renewed);
}
```

### 19.3 GetEvidenceRecord

```csharp
EvidenceRecord GetEvidenceRecord(byte[] signedData);
```

Arsiv kaniti kaydini (hash agaci, zaman damgasi zinciri) dondurur.

---

## 20. Iki Asamali Imza (Prepare/Finalize)

External signer kullanilirken (donanim HSM, uzak imza servisi) iki asamali akis. SDK'nin kendisi imzalamaz; hash'i external signer'a gonderir, imzayi geri alip belgeye entegre eder.

### 20.1 PrepareSignatureAsync

```csharp
Task<SignPrepareResult> PrepareSignatureAsync(
    byte[] documentData,
    SignatureParameters parameters,
    byte[]? certData = null,
    byte[][]? chainData = null);
```

**Parametreler:**
- `documentData` — Imzalanacak belge
- `parameters` — Imza parametreleri
- `certData` — (opsiyonel) Imzalayan sertifika DER-encoded bytes
- `chainData` — (opsiyonel) Sertifika zinciri (her eleman DER-encoded)

**SignPrepareResult** (ozet — tum alanlar icin bkz. Appendix C):
- `Success: bool`
- `Hash: byte[]?` — External signer'in imzalayacagi hash
- `HashAlgorithm: string?` — "SHA256"/"SHA384"/"SHA512"
- `PreparedDocument: byte[]?` — Finalize icin opak (PAdES/CAdES/XAdES farkli alanlar kullanir)
- `Error: string?`

### 20.2 FinalizeSignatureAsync

```csharp
Task<SignatureResult> FinalizeSignatureAsync(
    SignPrepareResult prepareResult,
    byte[] externalSignature);
```

External signer'dan alinan imzayi belgeye yerlestirir.

**Tam akis ornegi:**

```csharp
// 1. Sertifika bytes (DER-encoded)
var certBytes = File.ReadAllBytes("signer.cer");

// 2. Prepare
var prepare = await sdk.PrepareSignatureAsync(pdfData,
    new SignatureParameters { Format = SignatureFormat.PAdES, Level = SignatureLevel.B_B },
    certData: certBytes);

// 3. External signer'a hash gonder (HSM, remote service vb.)
var signature = await externalSigner.SignHashAsync(prepare.Hash!, prepare.HashAlgorithm!);

// 4. Finalize
var result = await sdk.FinalizeSignatureAsync(prepare, signature);
File.WriteAllBytes("imzali.pdf", result.SignedData!);
```

---

## 21. Health ve Diagnostics

### 21.1 CheckHealth

SDK'nin kendi durum kontrolu (senkron, hizli).

```csharp
HealthStatus CheckHealth();
```

**HealthStatus:**
- `Status: string` — "healthy"/"degraded"/"unhealthy"
- `Version: string` — SDK surumu
- `Timestamp: DateTime`

### 21.2 CheckReadinessAsync  [YENI 2.0]

SDK + bagimliliklarinin hazir oldugunu kontrol eder — lisans, TSA erisilebilirligi, trust store, PKCS#11 kutuphaneleri.

```csharp
Task<ReadinessStatus> CheckReadinessAsync();
```

**ReadinessStatus:**
- `IsReady: bool` — Tum bagimliliklar hazir mi
- `Status: string` — "ready"/"degraded"/"not-ready"
- `Dependencies: List<ReadinessDependency>` — Her bagimlilik icin detay
- `Timestamp: DateTime`

**ReadinessDependency:**
- `Name: string` — Ornegin "TSA:kamusm", "TrustStore", "License"
- `IsReady: bool`
- `Error: string?`
- `LatencyMs: int?`

**Ornek (health probe — Kubernetes readinessProbe):**

```csharp
var ready = await sdk.CheckReadinessAsync();
if (!ready.IsReady)
{
    foreach (var dep in ready.Dependencies.Where(d => !d.IsReady))
        Console.Error.WriteLine($"SORUN: {dep.Name} — {dep.Error}");
    Environment.Exit(1);
}
```

---

## 22. Token Session Yonetimi

DigiMR SDK iki session yonetim modeli destekler. Ikisi de PIN lockout riskini azaltmak ve performans icin tasarlanmistir.

### 22.0 Iki Model: Kisa Omurlu vs. Uzun Omurlu

#### Model A: Kisa Omurlu Session (fire-and-forget)

Her imzalama talebi icin yeni bir provider olustur, imzala, dispose et. SDK icinde `HsmSessionPool` tek provider omru boyunca session'lari reuse eder (PIN tek sefer).

```csharp
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token, authContext);
var result = await sdk.SignDataWithProviderAsync(data, provider, parameters);
// using scope sonunda provider.Dispose() -> token session kapatilir
```

**Kullanim alani:**
- HTTP request basina tek imza (web API)
- Basit CLI araclari
- Kullanicinin bir kez onay verdigi senaryolar

**Dezavantaj:** PIN her yeni provider icin yeniden verilmeli. Hizli arka arkaya imzalamada (>3-5 imza) PIN tekrar tekrar sorulur.

#### Model B: Uzun Omurlu Session (`CreateTokenSessionAsync`)

Bir kez authenticate et, session ID al, N imzalama boyunca session'i kullan, sonra `CloseTokenSessionAsync` ile kapat.

```csharp
// 1. Session ac (PIN bir kez)
var session = await sdk.CreateTokenSessionAsync(SigningProviderType.Token, authContext);

// 2. N adet imza (PIN tekrar sormadan)
var certs = sdk.ListSessionCertificates(session.SessionId);
// (session-tabanli imzalama API'leri: HTTP API katmaninda SignWithSession endpoint'i — bkz. API_REFERANS)

// 3. Session'i kapat — token kaynaklarini serbest birak
await sdk.CloseTokenSessionAsync(session.SessionId);
```

**Kullanim alani:**
- Web uygulamasi oturumu (kullanici login → N imza → logout)
- Toplu imzalama pipeline (100+ belge, tek authentication)
- Tablet imza akisi (imzalayan oturum acar, tablet boyunca imzalar)

**Dikkat:** Kapatilmamis session'lar idle timeout (varsayilan 5 dk) sonrasi otomatik kapanir. Ama yine de `CloseTokenSessionAsync` cagrisi onerilir — token kaynaklari daha hizli serbest kalir.

#### Model Secimi

| Durum | Model |
|---|---|
| Tek imza isteği | **A (kisa omurlu)** |
| HTTP/request basina tek imza | **A** |
| 3+ imza arka arkaya, ayni kullanici | **B (uzun omurlu)** |
| Web oturumu — kullanici oturum acik tutuyor | **B** |
| Batch signing (>10 belge) | **B** |
| Web tabanli tablet/kiosk | **B** |

**Not:** Her iki model de `HsmSigningProvider` (HSM icin) veya `TokenSigningProvider` (PKCS#11 token) altinda ayni `HsmSessionPool` / `SemaphoreSlim`-based session reuse mekanizmasini kullanir. Fark **API katmaninda** — provider ne kadar sure yasar?

---

### 22.1 CreateTokenSessionAsync

```csharp
Task<TokenSessionInfo> CreateTokenSessionAsync(SigningProviderType type, AuthenticationContext context);
```

**Ornek:**

```csharp
var session = await sdk.CreateTokenSessionAsync(SigningProviderType.Token, new AuthenticationContext
{
    Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")!,
    Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
});
Console.WriteLine($"Session ID: {session.SessionId} ({session.ProviderType}), Aktif: {session.IsActive}");
```

### 22.2 ListTokenSessions

```csharp
List<TokenSessionInfo> ListTokenSessions();
```

**Ornek:**

```csharp
foreach (var s in sdk.ListTokenSessions().Where(s => s.IsActive))
    Console.WriteLine($"{s.SessionId}: {s.ProviderType} — olusturuldu {s.CreatedAt:yyyy-MM-dd HH:mm}");
```

### 22.3 GetTokenSession

```csharp
TokenSessionInfo GetTokenSession(string sessionId);
```

### 22.4 CloseTokenSessionAsync

```csharp
Task CloseTokenSessionAsync(string sessionId);
```

Oturumu kapatir ve token kaynaklarini serbest birakir.

### 22.5 ListSessionCertificates

```csharp
List<CertificateInfo> ListSessionCertificates(string sessionId);
```

Oturumdaki sertifikalari listeler (PIN tekrar sormadan).

**Ornek:**

```csharp
var certs = sdk.ListSessionCertificates(session.SessionId);
foreach (var c in certs)
    Console.WriteLine($"- {c.Subject} ({c.NotAfter:yyyy-MM-dd})");
```

---

## 23. Sertifika Yardimcilari

### 23.1 LoadCertificate

```csharp
CertificateInfo LoadCertificate(string certPath, string? password = null);
```

Dosyadan sertifika yukler. `.cer/.crt/.pem` veya `.p12/.pfx` (password ile) destekler.

**Ornek:**

```csharp
var cert = sdk.LoadCertificate("signer.pfx", Environment.GetEnvironmentVariable("PFX_PASSWORD"));
Console.WriteLine($"{cert.Subject}, gecerli: {cert.NotBefore:d} - {cert.NotAfter:d}");
```

---

## Appendix A: Model Siniflari

SDK tarafindan kullanilan temel model siniflari. Tam alan listeleri icin ilgili kaynak dosyalari referans alinmistir.

### SignatureParameters
Imza uretimi parametreleri. Alanlar: `Format` (enum), `Level` (enum), `TsaUrl`, `Reason`, `Location`, `ContactInfo`, `DetachedSignature` (bool), `HashAlgorithm`, `VisualSignature` (PAdES icin), `SignerRole`, `CommitmentType`, `ProductionPlace`, `DataObjectFormat`, `SignaturePolicy`, `ValidateCertificateBeforeSigning` (bool, default false), `SigningTime` (DateTime?), `ContentIdentifier` (byte[]?).

### AuthenticationContext
Provider kimlik dogrulama. Alanlar: `Pin`, `Msisdn`, `Otp`, `BiometricData` (byte[]?), `CertificatePath`, `CertificateData` (byte[]?), `CertificatePassword`, `SlotId`, `KeyLabel`, `KeyId`, `Pkcs11LibraryPath`, `TokenFilter`, `AgentUrl`, `AgentApiKey`, `AgentSessionId`, `CertificateBase64`, `CertificateIndex`, `Metadata` (Dictionary). Hassas alanlar `[JsonIgnore]`.

### ValidationOptions
MA3 EParameters esdegerli. Alanlar: `TrustSigningTimeAttribute` (bool, default true), `InitialCertificates` (List<byte[]>), `InitialCrls` (List<byte[]>), `InitialOcspResponses` (List<byte[]>), `GracePeriodSeconds` (int).

### SignBatchItem
Toplu imzalama girdisi. Alanlar: `DocumentId`, `Data` (byte[]), `FileName`.

### EypCreateOptions (Xml.EYP)
EYP paket olusturma. Alanlar: `UstYazi`, `Ekler` (List), `Ustveri`, `NihaiUstveri`, `TsaUrl`, `ImzaLevel`, `MuhurLevel`, EYP versiyonu vb. (bkz. `src/DigitalSignature.Xml/EYP/Interfaces/IEypPackageService.cs`).

### KepPackage (Xml)
KEP mesaji. Alanlar (bkz. Xml/Models/KepPackage.cs): gonderi, alici, gonderi zamani, ek'ler vb.

### UetsMessage (Xml)
UETS teblig mesaji. Alanlar (bkz. Xml/Models/UetsMessage.cs).

### ASiCDocument (Core.Models.ASiCContainer)
ASiC konteyner belgesi. Alanlar: `FileName`, `Content` (byte[]), `MimeType`.

### TsaProviderConfig
TSA yapilandirmasi. Factory: `KamuSM(url, userId, password, hashAlg)`, `BasicAuth(url, username, password, hashAlg)`, `Anonymous(url, hashAlg)`.

### TsaProvidersOptions
Coklu TSA yapilandirmasi. Alanlar: `Default` (string), `Providers` (Dictionary).

### VisualSignatureOptions
PAdES gorunur imza. Alanlar: `Page` (int), `X`, `Y`, `Width`, `Height`, `Text`, `StampText`, `SignatureImage`, `BackgroundImage`.

---

## Appendix B: Enum Tipleri

### SignatureFormat
`CAdES`, `PAdES`, `XAdES`, `ASiC_S`, `ASiC_E`, `JAdES`, `OOXML`

### SignatureLevel
`B_B`, `B_T`, `B_C`, `B_X`, `B_LT`, `B_LTA`

### SigningProviderType
`Software`, `Token`, `Mobile`, `Biometric`, `ESeal`, `RemoteToken`, `HSM`, `CloudHSM`

### DocumentFormat (ValidationResult.Format)
`Unknown`, `PAdES`, `CAdES`, `XAdES`, `EYP_V1_3`, `EYP_V2_0`, `EYP_V2_1`, `OOXML`

### SignatureRole (SignatureInfo.Role)
`Document`, `Timestamp`, `CounterSignature`, `Seal`

### ASiCType
`ASiC_S`, `ASiC_E`

---

## Appendix C: Result Tipleri

Kullanilan result/info model'lerinin tam alan listesi (ozel bolumler referans alir):

- **SignatureResult** (Bolum 6): `Success`, `SignedData`, `Message`, `Error`, `DocumentId`.
- **ValidationResult** (Bolum 7.1): `IsValid`, `Format`, `IsDocumentIntact`, `Signatures`, `Errors`, `Warnings`, `Details`, `ValidationState`, `IsPremature`.
- **SignatureInfo** (ValidationResult.Signatures): `Index`, `Role`, `IsValid`, `SignerSubject`, `CertificateSerial`, `SignatureTime`, `Level`, `HashAlgorithm`, `Errors`, `Metadata`.
- **SignatureTypeCheckResult** (Bolum 7.3): `Matches`, `IsValid`, `DetectedFormat`, `DetectedLevel`, `ExpectedFormat`, `ExpectedLevel`, `Reason`.
- **SignatureInspectionResult** (Bolum 7.4): `Success`, `DetectedFormat`, `SignatureCount`, `Signatures`, `Deficiencies`.
- **SignatureInspectionEntry** (Bolum 7.4): `Index`, `SignerSubject`, `CertificateSerial`, `SignatureTime`, `Level`, `HashAlgorithm`, `IsValid`, `Role`, `Errors`, `Metadata`.
- **UpgradeResult** (Bolum 8.1): `Success`, `UpgradedData`, `DetectedFormat`, `PreviousLevel`, `NewLevel`, `Error`.
- **TimestampVerificationResult** (Bolum 9.3): `IsValid`, `GenTime` (DateTime), `TsaName`, `SerialNumber`, `HashAlgorithm`.
- **TsaProviderInfo** (Bolum 9.4): `Key`, `Name`, `Url`, `AuthType`, `IsDefault`, `IsEnabled`.
- **EypCreateResult** / **EypVerificationResult** / **EypPackageInfo** (Bolum 10).
- **KepVerificationResult** / **UetsVerificationResult** / **ASiCVerificationResult** / **ASiCContainer** (Bolum 11-13).
- **JadesVerificationResult** (Bolum 14.2).
- **TokenPinStatus** (Bolum 17.2): `IsOk`, `IsLocked`, `IsFinalTry`, `IsCountLow`, `Message`, `HasProtectedAuthPath`, `SupportsAlwaysAuthenticate`.
- **ProviderInfo** (Bolum 17.5): `Type`, `Name`, `Description`, `Version`, `SupportsHashSigning`, `SupportedAlgorithms`.
- **TokenInfo** (Bolum 17.6): `SlotId`, `Label`, `Manufacturer`, `Model`, `SerialNumber`, `IsPresent`, `LibraryPath`.
- **TokenSessionInfo** (Bolum 22.1): `SessionId`, `ProviderType`, `CreatedAt`, `IsActive`.
- **HealthStatus** (Bolum 21.1): `Status`, `Version`, `Timestamp`.
- **ReadinessStatus** (Bolum 21.2): `IsReady`, `Status`, `Dependencies`, `Timestamp`.
- **ReadinessDependency**: `Name`, `IsReady`, `Error`, `LatencyMs`.
- **PreservationStatus** (Bolum 19.1): arsiv durumu alanlari.
- **EvidenceRecord** (Bolum 19.3): kanit kaydi alanlari.
- **SignPrepareResult** (Bolum 20.1): `Success`, `PreparedDocument`, `Hash`, `HashAlgorithm`, `Error`, `Format`, `PreparedPdfBytes`, `EstimatedSignatureSize`, `SignedAttrsBytes`, `SignerCertBytes`, `ChainBytes`, `OriginalData`, `XadesSignedInfoXml`, `XadesSignatureId`, `XadesQualifyingPropsXml`, `XadesOriginalXmlContent`, `XadesIsDetached`.
- **CertificateInfo** (Bolum 17/22/23): `Subject`, `Issuer`, `NotBefore`, `NotAfter`, `SerialNumber`, `Thumbprint`, `IsValid` (tam liste: `src/DigitalSignature.Core/Models/CertificateInfo.cs`).

**Not:** `BatchResult` class'i YOKTUR — `SignBatchAsync` dogrudan `List<SignatureResult>` doner.

---

## Appendix D: Silinen Metotlar (v1.x → v2.0 Gocus)

v2.0'da API yuzeyi sadelestirildi. Asagidaki metotlar kaldirildi — bunlari kullanan kodlar `byte[]` versiyonuna gecmelidir.

| Kaldirilan | Yerine |
|---|---|
| `SignPdfAsync(string path, ...)` | `var data = File.ReadAllBytes(path); SignDataWithProviderAsync(data, ...)` |
| `SignPdfWithProviderAsync(string path, ...)` | Ayni yontem |
| `SignXmlAsync(string path, ...)` | `var data = File.ReadAllBytes(path); SignDataWithProviderAsync(data, ...)` |
| `SignXmlWithProviderAsync(string path, ...)` | Ayni yontem |
| `ValidateDocumentAsync(string path)` | `var data = File.ReadAllBytes(path); ValidateDocumentAsync(data)` |
| `CreateEypAsync(params string[] paths)` | `CreateEypPackageV21Async(EypCreateOptions)` |
| `sdk.GetCertificateInfo(cert)` | Ayri `ICertificateManager` kullan |
| `sdk.ListSigners(...)` | `InspectSignaturesAsync` |
| `sdk.GetPrivateKey(...)` | Kaldirildi (guvenlik) |
| `sdk.GetMimeType(...)` | Kaldirildi (SDK disi sorumluluk) |
| `SignOoxmlAsync(...)` | `SignDataWithProviderAsync` + `Format = SignatureFormat.OOXML` |
| `VerifyOoxml(...)` | `ValidateDocumentAsync` (format otomatik tespit) |

**Not:** `ExportSignature(byte[])` SILINMEDI — interface'de hala var. Bolum 7.5'te dokumante edilmistir.
