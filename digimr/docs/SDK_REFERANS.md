# DigiMR SDK Referansi (SDK Reference)

> Turkiye'nin nitelikli elektronik imza altyapisi icin kapsamli SDK.
> Comprehensive SDK for Turkish qualified electronic signatures.

**SDK Versiyon:** 2.0 | **.NET:** 8.0+ | **Lisans:** Proprietary | **Test:** 700+ test basarili

---

## Icindekiler (Table of Contents)

1. [IDigitalSignatureSDK Interface](#idigitalsignaturesdk-interface)
2. [ISigningProvider Interface](#isigningprovider-interface)
3. [ISigningProviderFactory Interface](#isigningproviderfactory-interface)
4. [Model Siniflari (Models)](#model-siniflari-models)
5. [Enum Tipleri (Enumerations)](#enum-tipleri-enumerations)
6. [Sonuc Tipleri (Result Types)](#sonuc-tipleri-result-types)
7. [EYP Modelleri (EYP Models)](#eyp-modelleri-eyp-models)
8. [Guvenlik Notlari (Security Notes)](#guvenlik-notlari-security-notes)

---

## IDigitalSignatureSDK Interface

Ana SDK arayuzu. Tum imzalama, dogrulama ve belge islemlerini tek bir noktadan sunar.
Main SDK interface. Provides all signing, validation, and document operations from a single entry point.

**Namespace:** `DigitalSignature.SDK`

---

### Provider Yonetimi (Provider Management)

#### ProviderFactory

```csharp
ISigningProviderFactory ProviderFactory { get; }
```

Provider fabrikasi. Yeni provider tipleri kaydetmek veya mevcut provider almak icin kullanilir.
Provider factory for registering new provider types or retrieving existing ones.

**Ornek / Example:**
```csharp
var supportedTypes = sdk.ProviderFactory.GetSupportedTypes();
var provider = sdk.ProviderFactory.CreateProvider(SigningProviderType.Software);
```

---

#### CreateAndAuthenticateProviderAsync

```csharp
Task<ISigningProvider> CreateAndAuthenticateProviderAsync(
    SigningProviderType type,
    AuthenticationContext context);
```

Belirtilen tipte bir provider olusturur ve kimlik dogrulamasi yapar.
Creates and authenticates a provider of the specified type.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `type` | `SigningProviderType` | Provider turu (Software, Token, Mobile, vb.) |
| `context` | `AuthenticationContext` | Kimlik dogrulama bilgileri |

**Donus:** Dogrulanmis `ISigningProvider` instance. Hata durumunda exception firlatir.

**Ornek / Example:**
```csharp
var context = new AuthenticationContext
{
    CertificatePath = "sertifika.pfx",
    CertificatePassword = "sifre123"
};
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software, context);
```

---

### Imzalama (Signing)

#### SignDataWithProviderAsync

```csharp
Task<SignatureResult> SignDataWithProviderAsync(
    byte[] data,
    ISigningProvider provider,
    SignatureParameters parameters);
```

Herhangi bir veriyi provider ile CMS/CAdES formatinda imzalar.
Signs arbitrary data using a provider in CMS/CAdES format.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `data` | `byte[]` | Imzalanacak veri |
| `provider` | `ISigningProvider` | Dogrulanmis imzalama provider'i |
| `parameters` | `SignatureParameters` | Imza parametreleri (format, seviye, vb.) |

**Donus:** `SignatureResult` - Basarili ise `SignedData` dolu doner.

---

#### SignPdfWithProviderAsync

```csharp
Task<SignatureResult> SignPdfWithProviderAsync(
    string documentPath,
    ISigningProvider provider,
    SignatureParameters parameters);
```

PDF dosyasini PAdES formatinda imzalar. Gorunur imza destegi vardir.
Signs a PDF file in PAdES format. Supports visible signatures.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `documentPath` | `string` | PDF dosya yolu |
| `provider` | `ISigningProvider` | Dogrulanmis imzalama provider'i |
| `parameters` | `SignatureParameters` | Imza parametreleri + VisualSignature opsiyonu |

**Donus:** `SignatureResult` - `SignedData` imzalanmis PDF byte dizisi.

**Ornek / Example:**
```csharp
var parameters = new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr/timestamp",
    Reason = "Onaylandi",
    Location = "Istanbul",
    VisualSignature = new VisualSignatureOptions
    {
        Page = 1, X = 50, Y = 50, Width = 200, Height = 80,
        Mode = VisualSignatureMode.TextOnly,
        Text = "Dijital olarak imzalanmistir"
    }
};
var result = await sdk.SignPdfWithProviderAsync("belge.pdf", provider, parameters);
```

---

#### SignXmlWithProviderAsync

```csharp
Task<SignatureResult> SignXmlWithProviderAsync(
    string documentPath,
    ISigningProvider provider,
    SignatureParameters parameters);
```

XML dosyasini XAdES formatinda imzalar.
Signs an XML file in XAdES format.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `documentPath` | `string` | XML dosya yolu |
| `provider` | `ISigningProvider` | Dogrulanmis imzalama provider'i |
| `parameters` | `SignatureParameters` | Imza parametreleri (XAdES seviyesi, vb.) |

**Donus:** `SignatureResult` - `SignedData` imzalanmis XML byte dizisi.

---

### EYP 2.0/2.1 Paket Islemleri

#### CreateEypPackageV2Async

```csharp
Task<EypCreateResult> CreateEypPackageV2Async(
    EypCreateOptions options,
    ISigningProvider signerProvider,
    ISigningProvider? sealProvider = null);
```

EYP 2.0/2.1 OPC paketi olusturur. CAdES detached imza + dual hash (SHA-384 + SHA-512).
Creates an EYP 2.0/2.1 OPC package with CAdES detached signature and dual hashing.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `options` | `EypCreateOptions` | Paket icerik ve yapilandirma secenekleri |
| `signerProvider` | `ISigningProvider` | Imzaci provider (CAdES P4/B-LT) |
| `sealProvider` | `ISigningProvider?` | Muhur provider (CAdES-A/B-LTA), opsiyonel |

**Donus:** `EypCreateResult` - `PackageData` EYP dosya byte dizisi.

---

#### VerifyEypPackageV2

```csharp
EypVerificationResult VerifyEypPackageV2(byte[] eypData);
```

EYP 2.0/2.1 paketini dogrular: yapi + hash + imza + muhur kontrolleri.
Verifies an EYP 2.0/2.1 package: structure, hashes, signature, and seal.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `eypData` | `byte[]` | EYP dosya byte dizisi |

**Donus:** `EypVerificationResult` - Detayli dogrulama sonucu.

---

#### ExtractEypPackageV2

```csharp
EypPackageInfo ExtractEypPackageV2(byte[] eypData);
```

EYP 2.0/2.1 paketinden belgeleri ve metadatayi cikarir.
Extracts documents and metadata from an EYP 2.0/2.1 package.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `eypData` | `byte[]` | EYP dosya byte dizisi |

**Donus:** `EypPackageInfo` - UstYazi, Ekler, Ustveri, NihaiUstveri.

---

### Dogrulama (Unified Validation)

#### ValidateDocumentAsync (byte[])

```csharp
Task<ValidationResult> ValidateDocumentAsync(byte[] documentData);
```

Herhangi bir imzali belgeyi otomatik format tespiti ile dogrular.
Validates any signed document with automatic format detection.

Desteklenen formatlar / Supported formats:
- **PDF** (PAdES) - magic bytes: `%PDF` (0x25504446)
- **CAdES** (CMS/PKCS#7) - ASN.1 tag: 0x30
- **XAdES** (XML) - tag: `<` (0x3C)
- **EYP 1.3 / 2.0 / 2.1** - ZIP/OPC: 0x504B

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `documentData` | `byte[]` | Belge byte dizisi |

**Donus:** `ValidationResult` - Format, imza bilgileri, hatalar, uyarilar.

---

#### ValidateDocumentAsync (string)

```csharp
Task<ValidationResult> ValidateDocumentAsync(string filePath);
```

Dosya yolundan okuyup otomatik format tespiti ile dogrular.
Reads from file path and validates with automatic format detection.

| Parametre | Tip | Aciklama |
|-----------|-----|----------|
| `filePath` | `string` | Belge dosya yolu |

---

### Legacy Metodlar (Backward Compatibility)

> Bu metodlar geriye uyumluluk icin korunmaktadir. Yeni gelistirmelerde provider tabanli
> metodlarin kullanilmasi onerilir.
> These methods are maintained for backward compatibility. Provider-based methods are
> recommended for new development.

#### LoadCertificate / GetCertificateInfo

```csharp
CertificateInfo LoadCertificate(string certPath, string? password = null);
CertificateInfo GetCertificateInfo(string certPath, string? password = null);
```

PKCS#12 sertifika dosyasini yukler ve bilgilerini doner.

---

#### SignPdfAsync (Legacy)

```csharp
Task<SignatureResult> SignPdfAsync(
    string documentPath,
    string certPath,
    string? certPassword = null,
    string? tsaUrl = null);
```

PDF dosyasini dogrudan sertifika dosyasi ile imzalar.

---

#### SignXmlAsync (Legacy)

```csharp
Task<SignatureResult> SignXmlAsync(
    string documentPath,
    string certPath,
    string? certPassword = null,
    string? tsaUrl = null);
```

XML dosyasini dogrudan sertifika dosyasi ile imzalar.

---

#### CreateEypAsync (Legacy)

```csharp
Task<SignatureResult> CreateEypAsync(
    List<string> documentPaths,
    string certPath,
    string? certPassword = null,
    List<string>? signers = null,
    string? tsaUrl = null);
```

EYP 1.3 paketi olusturur (eski format, DotNetZip/ZipArchive tabanli).

---

#### VerifySignature (Legacy)

```csharp
VerificationResult VerifySignature(string documentPath);
```

Imzali belgeyi dogrular. Yeni kod icin `ValidateDocumentAsync` kullanin.

---

#### AddTimestampAsync

```csharp
Task<byte[]> AddTimestampAsync(string documentPath, string tsaUrl);
```

Belgeye zaman damgasi ekler.

---

#### ListSigners

```csharp
List<string> ListSigners(string eypFilePath);
```

EYP paketindeki imzacilari listeler.

---

#### ExportSignature

```csharp
byte[] ExportSignature(string documentPath);
```

Belgeden imza verisini cikarir (DER/CMS formatinda).

---

## ISigningProvider Interface

Tum imzalama provider'larinin uyguladigi temel arayuz.
Base interface implemented by all signing providers.

**Namespace:** `DigitalSignature.Core.Interfaces`
**Kalitim:** `IDisposable`

```csharp
public interface ISigningProvider : IDisposable
{
    ProviderInfo GetProviderInfo();
    Task<AuthResult> AuthenticateAsync(AuthenticationContext context);
    bool IsAuthenticated { get; }
    Task<SigningResult> SignHashAsync(byte[] hash, string hashAlgorithm = "SHA256");
    Task<X509Certificate> GetCertificateAsync();
    Task<X509Certificate[]> GetCertificateChainAsync();
}
```

### Kullanim Akisi (Usage Flow)

```
1. AuthenticateAsync()  -->  Kimlik dogrula (PIN, sertifika, OTP)
2. GetCertificateAsync() -->  Imzaci sertifikasini al
3. SignHashAsync()       -->  Hash'i imzala
4. Dispose()            -->  Kaynaklari serbest birak
```

### Metodlar

#### GetProviderInfo

```csharp
ProviderInfo GetProviderInfo();
```

Provider hakkinda bilgi doner: tip, isim, desteklenen algoritmalar.

---

#### AuthenticateAsync

```csharp
Task<AuthResult> AuthenticateAsync(AuthenticationContext context);
```

Provider'a kimlik dogrulama yapar. Her provider tipi icin farkli alanlar gereklidir:

| Provider Tipi | Gerekli Alanlar |
|---------------|----------------|
| `Software` | `CertificatePath` + `CertificatePassword` veya `CertificateData` + `CertificatePassword` |
| `Token` | `Pin` + opsiyonel `SlotId`, `KeyLabel`, `Pkcs11LibraryPath`, `TokenFilter` |
| `Mobile` | `Msisdn` + `Otp` |
| `RemoteToken` | `AgentUrl` + opsiyonel `AgentApiKey`, `CertificateIndex` |
| `HSM` | `Pin` + `SlotId` + `Pkcs11LibraryPath` |

---

#### IsAuthenticated

```csharp
bool IsAuthenticated { get; }
```

Provider'in kimlik dogrulamasi yapilip yapilmadigini doner. `true` ise `SignHashAsync` cagrilabilir.

---

#### SignHashAsync

```csharp
Task<SigningResult> SignHashAsync(byte[] hash, string hashAlgorithm = "SHA256");
```

Hash'i imzalar. Provider kendi mekanizmasiyla (token PKCS#11, mobil gateway, yazilim RSA) imzalar.

| Parametre | Tip | Varsayilan | Aciklama |
|-----------|-----|-----------|----------|
| `hash` | `byte[]` | - | Imzalanacak hash degeri |
| `hashAlgorithm` | `string` | `"SHA256"` | Hash algoritmasi: SHA256, SHA384, SHA512 |

**Donus:** `SigningResult` - Imza byte'lari, sertifika, algoritma bilgisi.

---

#### GetCertificateAsync / GetCertificateChainAsync

```csharp
Task<X509Certificate> GetCertificateAsync();
Task<X509Certificate[]> GetCertificateChainAsync();
```

Imzacinin BouncyCastle `X509Certificate` sertifikasini ve zincirini doner (leaf -> root).

---

## ISigningProviderFactory Interface

Provider fabrikasi. Provider tipine gore dogru implementasyonu olusturur.

**Namespace:** `DigitalSignature.Core.Interfaces`

```csharp
public interface ISigningProviderFactory
{
    ISigningProvider CreateProvider(SigningProviderType type);
    IReadOnlyList<SigningProviderType> GetSupportedTypes();
    bool IsTypeSupported(SigningProviderType type);
}
```

| Metod | Aciklama |
|-------|----------|
| `CreateProvider` | Belirtilen tipte yeni bir provider instance olusturur |
| `GetSupportedTypes` | Desteklenen provider tiplerini listeler |
| `IsTypeSupported` | Tipin desteklenip desteklenmedigi kontrolu |

---

## Model Siniflari (Models)

### SignatureParameters

Imza parametreleri. Tum imzalama islemlerinde format, seviye ve gorsel imza ayarlarini tasir.
Signature parameters carrying format, level, and visual signature settings for all signing operations.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `Format` | `SignatureFormat` | `CAdES` | Imza formati |
| `Level` | `SignatureLevel` | `B_B` | Imza seviyesi (ETSI Baseline) |
| `TsaUrl` | `string?` | `null` | Zaman damgasi sunucu URL'si. B-T ve uzeri icin **zorunlu**. |
| `Reason` | `string?` | `null` | Imza nedeni (PDF imzalarinda gorunur) |
| `Location` | `string?` | `null` | Imza konumu (PDF imzalarinda gorunur) |
| `ContactInfo` | `string?` | `null` | Imzaci iletisim bilgisi |
| `DetachedSignature` | `bool` | `false` | Detached (ayrik) imza mi? |
| `HashAlgorithm` | `string` | `"SHA256"` | Hash algoritmasi (SHA256, SHA384, SHA512) |
| `VisualSignature` | `VisualSignatureOptions?` | `null` | Gorunur imza secenekleri (sadece PAdES) |
| `SignerRole` | `SignerRoleAttribute?` | `null` | Imzaci rolu (CAdES/XAdES) |
| `CommitmentType` | `CommitmentTypeAttribute?` | `null` | Taahhut turu (CAdES/XAdES) |
| `ProductionPlace` | `SignatureProductionPlace?` | `null` | Imza uretim yeri (CAdES/XAdES) |
| `DataObjectFormat` | `DataObjectFormat?` | `null` | Veri nesnesi format bilgisi (XAdES) |
| `SignaturePolicy` | `SignaturePolicy?` | `null` | Imza politikasi (ES-EPES) |

#### Validate Metodu

```csharp
public bool Validate(out string? error)
```

Parametre dogrulamasi yapar. TSA URL gerekliligi, string uzunluk limitleri ve koordinat araliklari kontrol edilir.
Basarisiz ise `error` parametresinde hata mesaji doner.

---

### AuthenticationContext

Provider kimlik dogrulama bilgileri. Her provider turu icin gerekli alanlar farklidir.

**Namespace:** `DigitalSignature.Core.Models`

**GUVENLIK:** Hassas alanlar `[JsonIgnore]` ile isaretlenmistir; JSON serializasyonunda yer almaz.

| Alan | Tip | JsonIgnore | Aciklama |
|------|-----|-----------|----------|
| `Pin` | `string?` | **EVET** | Token/akilli kart PIN kodu |
| `Msisdn` | `string?` | Hayir | Mobil imza telefon numarasi (MSISDN) |
| `Otp` | `string?` | **EVET** | Mobil imza OTP kodu |
| `BiometricData` | `byte[]?` | **EVET** | Biyometrik veri (imza pad, parmak izi) |
| `CertificatePath` | `string?` | Hayir | PKCS#12 sertifika dosyasi yolu |
| `CertificateData` | `byte[]?` | Hayir | PKCS#12 sertifika byte dizisi |
| `CertificatePassword` | `string?` | **EVET** | PKCS#12 sertifika sifresi |
| `SlotId` | `int?` | Hayir | Token slot ID (PKCS#11) |
| `KeyLabel` | `string?` | Hayir | Anahtar etiketi (PKCS#11) |
| `Pkcs11LibraryPath` | `string?` | Hayir | PKCS#11 kutuphane yolu |
| `TokenFilter` | `string?` | Hayir | Token filtre/arama kriteri (label, seri no, uretici) |
| `AgentUrl` | `string?` | Hayir | Uzak Token Agent URL'si |
| `AgentApiKey` | `string?` | **EVET** | Uzak Token Agent API key'i |
| `CertificateIndex` | `int?` | Hayir | Hangi sertifika ile imzalanacak (0 = ilk) |
| `Metadata` | `Dictionary<string, string>` | Hayir | Ek metadata (varsayilan: bos) |

#### Validate Metodu

```csharp
public bool Validate(out string? error)
```

URL formati, sertifika dosya yolu, PKCS#11 kutuphane yolu ve veri boyutu dogrulamasi yapar.

#### ToString (Guvenli)

`ToString()` hassas alanlari maskeler (`****`), loglarda guvenli kullanim icin.

---

### VisualSignatureOptions

PDF gorunur imza secenekleri. Text, image, stamp ve kombinasyonlarini destekler.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `Page` | `int` | `1` | Imzanin konulacagi sayfa (1-tabanli) |
| `X` | `float` | `0` | Sol kenardan uzaklik (pt) |
| `Y` | `float` | `0` | Alt kenardan uzaklik (pt) |
| `Width` | `float` | `200` | Imza alani genisligi (pt) |
| `Height` | `float` | `80` | Imza alani yuksekligi (pt) |
| `Mode` | `VisualSignatureMode` | `TextOnly` | Goruntuleme modu |
| `Text` | `string?` | `null` | Gorunecek metin |
| `SignerNameTemplate` | `string?` | `null` | Imzaci adi sablonu |
| `FontName` | `string?` | `null` | Yazi tipi adi |
| `FontSize` | `float` | `10` | Yazi tipi boyutu |
| `SignatureImage` | `byte[]?` | `null` | Imza gorseli (PNG/JPG) |
| `BackgroundImage` | `byte[]?` | `null` | Arka plan gorseli |
| `StampText` | `string?` | `null` | Damga metni |
| `StampStyle` | `StampStyle` | `RoundedRectangle` | Damga stili |
| `StampColor` | `string?` | `null` | Damga rengi |
| `AnchorBookmark` | `string?` | `null` | Bookmark bazli yerlestirme |
| `AnchorFieldName` | `string?` | `null` | Form alani bazli yerlestirme |

---

### Imza Nitelik Siniflari (Signature Attribute Classes)

#### SignerRoleAttribute

Imzaci rolu (CAdES OID: `1.2.840.113549.1.9.16.2.28`, XAdES: SignerRole).

| Alan | Tip | Aciklama |
|------|-----|----------|
| `ClaimedRoles` | `List<string>` | Beyan edilen roller ("Manager", "Director" vb.) |
| `CertifiedRoles` | `List<byte[]>` | Sertifikalanmis roller (attribute certificate DER) |

#### CommitmentTypeAttribute

Taahhut turu (CAdES OID: `1.2.840.113549.1.9.16.2.16`).

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `Type` | `CommitmentTypeId` | `ProofOfOrigin` | Taahhut turu tanımlayıcısı |

`GetOidString()` metodu ilgili OID string'ini doner.

#### SignatureProductionPlace

Imza uretim yeri (CAdES OID: `1.2.840.113549.1.9.16.2.17`).

| Alan | Tip | Aciklama |
|------|-----|----------|
| `City` | `string?` | Sehir |
| `StateOrProvince` | `string?` | Eyalet/il |
| `PostalCode` | `string?` | Posta kodu |
| `CountryName` | `string?` | Ulke adi |

#### DataObjectFormat

Veri nesnesi format bilgisi (XAdES).

| Alan | Tip | Aciklama |
|------|-----|----------|
| `MimeType` | `string?` | MIME tipi (orn: "application/pdf") |
| `Encoding` | `string?` | Kodlama |
| `Description` | `string?` | Aciklama |

#### SignaturePolicy

Imza politikasi (ES-EPES, CAdES OID: `1.2.840.113549.1.9.16.2.15`).

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `PolicyId` | `string` | `""` | Politika OID'si (orn: "1.2.3.4.5") |
| `PolicyUri` | `string?` | `null` | Politika dokumani URL'si |
| `PolicyDigestAlgorithm` | `string` | `"SHA256"` | Hash algoritmasi |
| `PolicyDigestValue` | `byte[]?` | `null` | Politika dokumaninin hash degeri |
| `PolicyDescription` | `string?` | `null` | Politika aciklamasi |

---

## Enum Tipleri (Enumerations)

### SignatureFormat

Desteklenen imza formatlari.

**Namespace:** `DigitalSignature.Core.Models`

| Deger | Aciklama |
|-------|----------|
| `CAdES` | CMS Advanced Electronic Signature |
| `PAdES` | PDF Advanced Electronic Signature |
| `XAdES` | XML Advanced Electronic Signature |
| `ASiC_S` | Associated Signature Container - Simple |
| `ASiC_E` | Associated Signature Container - Extended |

---

### SignatureLevel

ETSI imza seviyeleri (Baseline Profiles).

**Namespace:** `DigitalSignature.Core.Models`

| Deger | Aciklama | TSA Gerekli? |
|-------|----------|-------------|
| `B_B` | Basic - Temel imza (sertifika + imza) | Hayir |
| `B_T` | With Timestamp - Zaman damgali imza | **Evet** |
| `B_C` | ES-C - CompleteCertificateRefs + CompleteRevocationRefs | **Evet** |
| `B_X` | ES-X - B-C + SigAndRefsTimeStamp | **Evet** |
| `B_LT` | Long Term - CRL/OCSP gomulu, uzun donemli dogrulama | **Evet** |
| `B_LTA` | Long Term with Archive - Arsiv zaman damgali | **Evet** |

---

### SigningProviderType

Imzalama saglayici turleri.

**Namespace:** `DigitalSignature.Core.Models`

| Deger | Aciklama |
|-------|----------|
| `Software` | PKCS#12 yazilim sertifikasi |
| `Token` | PKCS#11 donanim token / akilli kart |
| `Mobile` | Mobil imza (Turkcell, Vodafone, Turk Telekom) |
| `Biometric` | Biyometrik imza (imza pad, parmak izi) |
| `ESeal` | Elektronik muhur (sunucu tarafli otomatik imza) |
| `RemoteToken` | Uzak token (Token Agent uzerinden PKCS#11) |
| `HSM` | HSM Enterprise (session pooling, PIN modlari, multi-slot) |

---

### DocumentFormat

Otomatik tespit edilen belge formati (UnifiedValidation).

**Namespace:** `DigitalSignature.Core.Models`

| Deger | Magic Bytes | Aciklama |
|-------|------------|----------|
| `Unknown` | - | Taninmayan format |
| `PAdES` | `%PDF` (0x25504446) | PDF imzali belge |
| `CAdES` | 0x30 (ASN.1) | CMS/PKCS#7 detached veya attached |
| `XAdES` | `<` (0x3C) | XML imzali belge |
| `EYP_V1_3` | 0x504B (ZIP) | EYP 1.3 paketi |
| `EYP_V2_0` | 0x504B (ZIP/OPC) | EYP 2.0 paketi |
| `EYP_V2_1` | 0x504B (ZIP/OPC) | EYP 2.1 paketi |

---

### SignatureRole

Imza rolu siniflandirmasi.

**Namespace:** `DigitalSignature.Core.Models`

| Deger | Aciklama |
|-------|----------|
| `Document` | Ana belge imzasi |
| `Timestamp` | RFC 3161 zaman damgasi |
| `CounterSignature` | Karsi-imza |
| `Seal` | Elektronik muhur |

---

### VisualSignatureMode

Gorunur imza goruntuleme modu.

| Deger | Aciklama |
|-------|----------|
| `TextOnly` | Sadece metin |
| `ImageOnly` | Sadece gorsel |
| `TextAndImage` | Metin + gorsel |
| `StampOnly` | Sadece damga |

---

### StampStyle

Damga stili.

| Deger | Aciklama |
|-------|----------|
| `Rectangle` | Dikdortgen |
| `RoundedRectangle` | Yuvarlak koseli dikdortgen |
| `Oval` | Oval |
| `Badge` | Rozet |
| `Ribbon` | Serit |

---

### CommitmentTypeId

Taahhut turu tanimlayicilari (ETSI TS 101 733).

| Deger | OID | Aciklama |
|-------|-----|----------|
| `ProofOfOrigin` | `1.2.840.113549.1.9.16.6.1` | Kaynak kaniti |
| `ProofOfReceipt` | `1.2.840.113549.1.9.16.6.2` | Teslim alma kaniti |
| `ProofOfDelivery` | `1.2.840.113549.1.9.16.6.3` | Teslimat kaniti |
| `ProofOfSender` | `1.2.840.113549.1.9.16.6.4` | Gonderici kaniti |
| `ProofOfApproval` | `1.2.840.113549.1.9.16.6.5` | Onay kaniti |
| `ProofOfCreation` | `1.2.840.113549.1.9.16.6.6` | Olusturma kaniti |

---

## Sonuc Tipleri (Result Types)

### SignatureResult

Imzalama islemi sonucu.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `Success` | `bool` | Islem basarili mi? |
| `SignedData` | `byte[]?` | Imzalanmis veri (basarili ise dolu) |
| `Message` | `string?` | Bilgi mesaji |
| `Error` | `string?` | Hata mesaji (basarisiz ise dolu) |

---

### ValidationResult

Birlesik dogrulama sonucu (tum belge tipleri icin).

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `IsValid` | `bool` | - | Genel dogrulama durumu |
| `Format` | `DocumentFormat` | - | Otomatik tespit edilen format |
| `IsDocumentIntact` | `bool` | `true` | Belge butunlugu (icerik degistirilmis mi?) |
| `Signatures` | `List<SignatureInfo>` | `[]` | Belgede bulunan tum imzalar |
| `Errors` | `List<string>` | `[]` | Dogrulama hatalari |
| `Warnings` | `List<string>` | `[]` | Dogrulama uyarilari |
| `Details` | `string?` | `null` | Ek detaylar |

**IsValid kosulu:** Hata yok + tum imzalar gecerli + belge butunlugu saglanmis.

---

### SignatureInfo

Belgede bulunan tek bir imzanin bilgileri.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `Index` | `int` | Imza sirasi (0-tabanli) |
| `Role` | `SignatureRole` | Imza rolu (Document, Timestamp, CounterSignature, Seal) |
| `IsValid` | `bool` | Bu imza gecerli mi? |
| `SignerSubject` | `string?` | Imzaci sertifika konusu (CN, vb.) |
| `CertificateSerial` | `string?` | Imzaci sertifika seri numarasi |
| `SignatureTime` | `DateTime?` | Imza zamani |
| `Level` | `SignatureLevel?` | Imza seviyesi (B-B, B-T, B-LT, B-LTA, vb.) |
| `HashAlgorithm` | `string?` | Kullanilan hash algoritmasi |
| `Errors` | `List<string>` | Imzaya ozel hatalar |
| `Metadata` | `Dictionary<string, string>` | Ek imza detaylari |

---

### AuthResult

Provider kimlik dogrulama sonucu.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `Success` | `bool` | Dogrulama basarili mi? |
| `Message` | `string?` | Bilgi mesaji |
| `Error` | `string?` | Hata mesaji |
| `RemainingAttempts` | `int?` | Kalan deneme hakki (token PIN icin) |

**Statik Fabrika Metodlari:**

```csharp
AuthResult.Succeeded(string? message = null);
AuthResult.Failed(string error, int? remainingAttempts = null);
```

---

### SigningResult

Provider imzalama sonucu. Hash imzalama isleminden donen imza byte'lari ve sertifika bilgisi.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `Success` | `bool` | Imzalama basarili mi? |
| `Signature` | `byte[]?` | Imza byte dizisi |
| `Certificate` | `X509Certificate?` | Imzaci sertifikasi (BouncyCastle) |
| `CertificateChain` | `X509Certificate[]?` | Sertifika zinciri (leaf -> root) |
| `Algorithm` | `string?` | Kullanilan algoritma (orn: "SHA256withRSA") |
| `SigningTime` | `DateTime?` | Imzalama zamani (UTC) |
| `Error` | `string?` | Hata mesaji |

**Statik Fabrika Metodlari:**

```csharp
SigningResult.Succeeded(byte[] signature, X509Certificate certificate,
    X509Certificate[]? chain = null, string algorithm = "SHA256withRSA");
SigningResult.Failed(string error);
```

---

### CertificateInfo

Sertifika bilgileri.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `SerialNumber` | `string` | `""` | Sertifika seri numarasi |
| `Subject` | `string` | `""` | Konu (CN, O, C, vb.) |
| `Issuer` | `string` | `""` | Veren kurum |
| `NotBefore` | `DateTime` | - | Gecerlilik baslangici |
| `NotAfter` | `DateTime` | - | Gecerlilik bitisi |
| `Thumbprint` | `string` | `""` | Parmak izi (SHA-1 hash) |
| `IsValid` | `bool` | - | Sertifika gecerli mi? (tarih + zincir kontrolu) |

---

### VerificationResult (Legacy)

Eski dogrulama sonucu. Yeni kodda `ValidationResult` kullanin.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `IsValid` | `bool` | Imza gecerli mi? |
| `Message` | `string?` | Bilgi mesaji |
| `SigningTime` | `DateTime?` | Imzalama zamani |
| `TimestampTime` | `DateTime?` | Zaman damgasi zamani |
| `SignerCertificate` | `X509Certificate?` | Imzaci sertifikasi (BouncyCastle) |
| `Errors` | `List<string>` | Dogrulama hatalari |

---

### ProviderInfo

Imzalama saglayicisi hakkinda bilgi.

**Namespace:** `DigitalSignature.Core.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `Type` | `SigningProviderType` | - | Provider turu |
| `Name` | `string` | `""` | Provider adi |
| `Description` | `string` | `""` | Provider aciklamasi |
| `Version` | `string?` | `null` | Provider versiyonu |
| `SupportsHashSigning` | `bool` | `true` | Hash imzalama destegi var mi? |
| `SupportedAlgorithms` | `List<string>` | `[SHA256, SHA384, SHA512]` | Desteklenen algoritmalar |

---

## EYP Modelleri (EYP Models)

### EypCreateOptions

EYP paketi olusturma secenekleri.

**Namespace:** `DigitalSignature.Xml.EYP.Interfaces`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `Version` | `EypVersion` | `V2_0` | EYP versiyon (V2_0, V2_1) |
| `SealMode` | `EypSealMode` | `WithSeal` | E-muhur modu |
| `SignatureFormat` | `EypSignatureFormat` | `CAdES` | Imza formati (V2.X: sadece CAdES) |
| `UstYazi` | `EypDocumentItem?` | `null` | Ust yazi belgesi |
| `Ekler` | `List<EypDocumentItem>` | `[]` | Ek belgeler |
| `Ustveri` | `EypUstveri` | `new()` | Metadata |
| `NihaiUstveri` | `EypNihaiUstveri` | `new()` | Nihai metadata |
| `TsaUrl` | `string?` | `null` | Zaman damgasi sunucu URL'si |
| `ImzaLevel` | `SignatureLevel` | `B_LT` | Imza seviyesi (CAdES P4) |
| `MuhurLevel` | `SignatureLevel` | `B_LTA` | Muhur seviyesi (CAdES-A) |

---

### EypCreateResult

EYP paketi olusturma sonucu.

| Alan | Tip | Aciklama |
|------|-----|----------|
| `Success` | `bool` | Islem basarili mi? |
| `PackageData` | `byte[]?` | EYP dosya byte dizisi |
| `Message` | `string?` | Bilgi mesaji |
| `Error` | `string?` | Hata mesaji |

---

### EypVerificationResult

EYP paketi dogrulama sonucu.

**Namespace:** `DigitalSignature.Xml.EYP.Models`

| Alan | Tip | Aciklama |
|------|-----|----------|
| `IsValid` | `bool` | Genel dogrulama durumu |
| `IsPackageStructureValid` | `bool` | Paket yapisi gecerli mi? |
| `IsPaketOzetiValid` | `bool` | PaketOzeti hashleri gecerli mi? |
| `IsNihaiOzetValid` | `bool` | NihaiOzet hashleri gecerli mi? |
| `IsImzaValid` | `bool` | CAdES imza gecerli mi? |
| `IsMuhurValid` | `bool` | CAdES-A muhur gecerli mi? |
| `AreHashesValid` | `bool` | Tum hashler (SHA-384 + SHA-512) gecerli mi? |
| `ImzaLevel` | `SignatureLevel` | Tespit edilen imza seviyesi |
| `MuhurLevel` | `SignatureLevel` | Tespit edilen muhur seviyesi |
| `ImzaciSubject` | `string?` | Imzaci sertifika konusu |
| `MuhurSubject` | `string?` | Muhur sertifika konusu |
| `PackageInfo` | `EypPackageInfo?` | Cikartilan paket bilgileri |
| `Errors` | `List<string>` | Dogrulama hatalari |
| `Warnings` | `List<string>` | Dogrulama uyarilari |

---

### EypPackageInfo

EYP paketinden cikartilan bilgiler.

**Namespace:** `DigitalSignature.Xml.EYP.Models`

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `PackageId` | `string` | `Guid.NewGuid()` | Paket benzersiz kimlik |
| `CreatedAt` | `DateTime` | `DateTime.UtcNow` | Olusturma zamani |
| `UstYazi` | `EypDocumentItem?` | `null` | Ust yazi belgesi |
| `Ekler` | `List<EypDocumentItem>` | `[]` | Ek belgeler |
| `Ustveri` | `EypUstveri` | `new()` | Metadata |
| `NihaiUstveri` | `EypNihaiUstveri?` | `null` | Nihai metadata |
| `PackageData` | `byte[]?` | `null` | Ham paket verisi |

---

### EypDocumentItem

EYP paketi icindeki belge ogesi.

| Alan | Tip | Varsayilan | Aciklama |
|------|-----|-----------|----------|
| `FileName` | `string` | `""` | Dosya adi |
| `Content` | `byte[]` | `[]` | Dosya icerigi |
| `MimeType` | `string` | `"application/octet-stream"` | MIME tipi |
| `EkId` | `string?` | `null` | Ek benzersiz kimligi |

---

## Guvenlik Notlari (Security Notes)

### Hassas Veri Korumasi

`AuthenticationContext` sinifindaki hassas alanlar `[JsonIgnore]` ile korunmaktadir:
- `Pin`, `Otp`, `CertificatePassword`, `AgentApiKey`, `BiometricData`
- Bu alanlar JSON serializasyonunda **yer almaz**
- `ToString()` metodu hassas alanlari `****` ile maskeler
- Uretim ortaminda sifreleri cevre degiskenleri veya guvenli yapilandirma deposu uzerinden saglayin

### Girdi Dogrulamasi

- `SignatureParameters.Validate()` ve `AuthenticationContext.Validate()` metodlari input dogrulama saglar
- TSA URL formati, dosya yolu gecisligi (path traversal), string uzunluk limitleri ve belge boyutu kontrol edilir
- `VisualSignatureOptions` koordinat ve boyut araliklari dogrulanir

### Provider Yasam Dongusu

```csharp
// Provider'lar IDisposable uygular - using blogu ile kullanin
using var provider = await sdk.CreateAndAuthenticateProviderAsync(type, context);
// ... imzalama islemleri ...
// Dispose otomatik cagrilir, PKCS#11 session/token kaynaklari serbest birakir
```

---

> **Not:** Bu referans, DigiMR SDK v1.0 public API'sini belgelemektedir.
> BouncyCastle.Cryptography 2.4.0 ve iTextSharp.LGPLv2.Core 3.4.22 kullanilmaktadir.
> Dahili (internal) sinif ve metodlar bu dokumana dahil degildir.
