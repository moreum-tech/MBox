# DigiMR Surum Gecmisi

Bu dokuman DigiMR surumleri arasindaki degisiklikleri, yeni ozellikleri ve migration gerekliliklerini listeler.

**Son Guncelleme:** 2026-08-27

---

## v2.3.1 — 2026-08-27

> Bu bolum yalniz release varliklarindaki degisikligi kapsar; v2.2.0-v2.3.1 arasindaki
> kod degisiklikleri bu dokumana henuz islenmedi.

### Java gRPC istemci SDK'si geri geldi

- **`digimr-sdk-1.0.0-all.jar` yeniden yayinda.** v2.0.1'de kaldirilan gRPC Java istemci
  SDK'si, v2.3.1 proto'lariyla yeniden derlenerek release'e geri kondu. Jar yalniz gRPC
  stub'larini ve gomulu grpc/netty calisma zamanini icerir; imza motoru jar'in **icinde
  degildir** — istekler API sunucusuna (gRPC port 7702) gider.
- **`digimr-java-examples.tar.gz`** — 10 calisir ornek program (imzalama, dogrulama, KEP,
  zaman damgasi, toplu imza, saglayici listesi, seviye yukseltme, benchmark) ve kullanim
  kilavuzu.
- **Neden geri geldi:** v2.0.1'de "yerine saf-Java in-process SDK gelecek" denmisti;
  `sdk/java-native` henuz yayinlanmadi. Java'dan API'yi cagirmak isteyen musterilerin
  arada hazir bir istemcisi kalmamisti.
- **Kok neden giderildi:** SDK'nin uretilmis gRPC stub'lari depoda tutuluyor, proto
  degistiginde guncellenmiyordu; EYP uclari eklendiginde modul sessizce derlenemez hale
  gelmisti. Stub'lar artik depoda tutulmuyor, her build'de API'nin kanonik
  proto'larindan uretiliyor.

`sdk/java-native` (saf-Java, in-process imza) calismasi suruyor ve ayri bir surumde
yayinlanacak.

---

## v2.1.0 — 2026-06-18

### İmzager Konformite (tüm formatlar × her iki SDK)
- **XAdES-A** hem .NET API hem Java native'de İmzager Kurumsal 2.10.5'te **TAM GEÇERLİ / UYARISIZ**:
  - **.NET c14n satır-sonu düzeltmesi** — CRLF kaynaklı `\r` → `&#xD;` kaçışı Santuario (İmzager) ile uyuşmuyordu (çekirdek imza geçersiz görünüyordu). c14n öncesi LF normalizasyonu eklendi; içerik referansı artık inclusive-c14n (ham `OuterXml` değil).
  - **"İptal bilgisi referanslarında doğrulamada kullanılmayan kaynak" uyarısı giderildi** (her iki SDK) — imza-TS'nin TSA revocation'ı yalnız `RevocationValues`'ta tutulur; imzacının `CompleteRevocationRefs`'ine konmaz.
  - `SigningProcessor` seviye eşlemesi düzeltildi (B_LTA→A): API artık XAdES-A üretiyor (T değil).
- **CAdES B-LTA .NET API** konformite üreteci eklendi; örnekler **attached** CMS (detached → İmzager "İçerik Özet" + "Arşiv ZD Özet" hatası verir).
- **Tüm format × platform İmzager geçerli**: CAdES, PAdES, XAdES-A, EYP 1.3/2.1, MA3-compat CAdES (.NET + Java native). Tek istisna: EYP 2.0 mührü software örnek e-mühür olduğunda güvensiz işaretlenir (beklenen; gerçek e-mühür HSM gerektirir).
- MA3-compat katmanı native SDK'ya delege ettiğinden bu düzeltmelerden otomatik yararlanır.

### Dokümantasyon
- Geliştirici dokümanları kod ile hizalandı: EYP metod adları `CreateEypPackageV20Async`/`V21Async`, gRPC `ValidateDocument`/`ValidateBatch`, timestamp verify alanları (`timestampTokenBase64`/`dataBase64`), PAdES/XAdES upgrade API'leri, XAdES 7 seviye (BES/T/C/X_Type1/X_Type2/XL/A).

---

## v2.0.1 — 2026-06-18

### Degisiklikler

- **Obfuscation**: .NET imza motorlari + SDK artik obfuscated (ters muhendislik direncli) olarak yayinlaniyor; korumasiz v2.0.0 kaldirildi. Docker / Linux / Windows ikililerinin tamami obfuscated.
- **Eski gRPC Java istemci SDK'si kaldirildi** (`digimr-sdk-*-all.jar`). Yerine, .NET SDK ile ayni modelde calisan **saf-Java in-process imza SDK'si** (`sdk/java-native`, BouncyCastle/PDFBox) hazirlaniyor; ayri bir surumde yayinlanacak. gRPC/REST API (port 7701/7702) degismedi — Java dahil tum diller API'yi cagirmaya devam edebilir.
  **Guncelleme: bu SDK v2.3.1'de geri getirildi — yukaridaki v2.3.1 bolumune bakin.**

---

## v2.0.0 — 2026-04-17

> **Breaking changes var.** Asagidaki migration rehberini uygulayin.

### Ozet

- **11 duplike metot silindi** — tek API yuzeyi (`byte[]` tabanli)
- **6 yeni SDK metodu** — readiness, inspection, TSA listesi, PIN dogrulama, slot sertifikalari
- **16 yeni Java gRPC metodu** — tam REST parity
- **5 yeni REST endpoint** — verify/check, verify/inspect, timestamp/providers, token/pin/verify, token/slots/{id}/certificates, health/ready
- **OOXML cross-platform** — `net10.0-windows` bagimliligi kaldirildi
- **1587+ test, 0 fail**

---

### 🔴 Breaking Changes — Migration Gerekli

#### 1. String-Path Overload'lar Kaldirildi

Tum imzalama, dogrulama, yukseltme metotlarinin `string path` overload'lari kaldirildi. Sadece `byte[]` versiyonlar kaldi. Caller dosyayi kendi okumalidir.

**Genel donusum kalibi:**

```csharp
// v1.x (artik calismiyor)
var result = await sdk.SignPdfWithProviderAsync("belge.pdf", provider, params);

// v2.0
var data = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(data, provider, params);
```

**Kaldirilan metot tablosu:**

| Kaldirilan | Yerine |
|---|---|
| `SignPdfAsync(string path, ...)` | `SignDataWithProviderAsync(byte[] data, ...)` |
| `SignPdfWithProviderAsync(string path, ...)` | Ayni yontem |
| `SignXmlAsync(string path, ...)` | `SignDataWithProviderAsync(byte[] data, ...)` |
| `SignXmlWithProviderAsync(string path, ...)` | Ayni yontem |
| `ValidateDocumentAsync(string path)` | `ValidateDocumentAsync(byte[] data)` |
| `CreateEypAsync(params string[] paths)` | `CreateEypPackageV20Async / V21Async(EypCreateOptions)` |
| `sdk.GetCertificateInfo(cert)` | `ICertificateManager` dogrudan kullan |
| `sdk.ListSigners(...)` | `InspectSignaturesAsync` (yeni, daha zengin) |
| `sdk.GetPrivateKey(...)` | **Kaldirildi** (guvenlik — API hic donmeliydi) |
| `sdk.GetMimeType(...)` | **Kaldirildi** (SDK kapsami disi) |
| `SignOoxmlAsync(...)` | `SignDataWithProviderAsync` + `Format = SignatureFormat.OOXML` |
| `VerifyOoxml(...)` | `ValidateDocumentAsync` (format otomatik tespit) |

#### 2. Enum Degisiklikleri

`SigningProviderType`:

| v1.x | v2.0 |
|---|---|
| `Certificate` | `Software` |
| `Hsm` | `HSM` (tum harfler buyuk) |

```csharp
// v1.x
SigningProviderType.Certificate

// v2.0
SigningProviderType.Software
```

#### 3. Model Property Isim Degisiklikleri

`AuthenticationContext`:

| v1.x | v2.0 |
|---|---|
| `Pkcs12Data` | `CertificateData` |
| `Password` | `CertificatePassword` |

```csharp
// v1.x
new AuthenticationContext
{
    Pkcs12Data = File.ReadAllBytes("sertifika.pfx"),
    Password = "..."
}

// v2.0
new AuthenticationContext
{
    CertificateData = File.ReadAllBytes("sertifika.pfx"),
    CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
}
```

`ASiCDocument`:

| v1.x | v2.0 |
|---|---|
| `Data` | `Content` |

```csharp
// v2.0
new ASiCDocument { FileName = "...", Content = File.ReadAllBytes("..."), MimeType = "application/pdf" }
```

#### 4. TsaProviderConfig Factory Degisikligi

`KamuSM` factory'sinin ikinci parametresi artik `userId` (int) — onceki dokumantasyonda yanlislikla `port` olarak yazilmisti.

```csharp
// v1.x belgelenen ama yanlis kullanilan:
TsaProviderConfig.KamuSM(url, port: 12345, password)

// v2.0 dogru kullanim:
TsaProviderConfig.KamuSM(url, userId: 12345, password)
```

`Http()` ve `RfcTsp()` factory'leri yerine:
- `BasicAuth(url, username, password, hashAlg)` — HTTP Basic Auth
- `Anonymous(url, hashAlg)` — Kimlik dogrulamasiz

#### 5. SignBatchAsync Imzasi

```csharp
// v1.x
Task<BatchResult> SignBatchAsync(List<BatchItem> items, ...)

// v2.0
Task<List<SignatureResult>> SignBatchAsync(List<SignBatchItem> items, ...)
```

**Not:** `BatchResult` sinifi kaldirildi. `SignBatchItem` alanlari: `DocumentId`, `Data`, `FileName`.

#### 6. PrepareSignatureAsync Imzasi

```csharp
// v1.x
Task<SignPrepareResult> PrepareSignatureAsync(
    byte[] documentData,
    SignatureParameters parameters,
    X509Certificate signerCertificate);

// v2.0
Task<SignPrepareResult> PrepareSignatureAsync(
    byte[] documentData,
    SignatureParameters parameters,
    byte[]? certData = null,
    byte[][]? chainData = null);
```

`X509Certificate` objesi yerine DER-encoded bytes kullanilir.

#### 7. Lisans Metotlari Interface'den Cikti

`SetLicenseFile`, `SetLicense`, `SetLicenseJson` artik sadece concrete class `DigitalSignatureSDK` uzerinde — `IDigitalSignatureSDK` interface'de yer almaz. Dependency injection kullaniyorsaniz:

```csharp
// Startup
builder.Services.AddSingleton<IDigitalSignatureSDK>(sp =>
{
    var sdk = new DigitalSignatureSDK();
    sdk.SetLicenseFile("digimr-sdk.lic"); // concrete class method
    return sdk; // interface olarak kullanima sunulur
});
```

---

### 🟢 Yeni Ozellikler

#### 6 Yeni SDK Metodu

1. **`CheckReadinessAsync()`** — SDK hazirlik kontrolu (lisans, TSA erisim, trust store, PKCS#11)
   ```csharp
   var ready = await sdk.CheckReadinessAsync();
   if (!ready.IsReady) foreach (var d in ready.Dependencies.Where(d => !d.IsReady))
       Console.Error.WriteLine($"{d.Name}: {d.Error}");
   ```

2. **`CheckSignatureTypeAsync(data, format, level?)`** — Hizli format/level kontrol
   ```csharp
   var check = await sdk.CheckSignatureTypeAsync(data, "PAdES", "B-T");
   if (!check.Matches) throw new Exception(check.Reason);
   ```

3. **`InspectSignaturesAsync(data)`** — Detayli imza envanteri
   ```csharp
   var i = await sdk.InspectSignaturesAsync(data);
   foreach (var sig in i.Signatures) Console.WriteLine($"{sig.SignerSubject} — {sig.Level}");
   ```

4. **`ListTsaProviders()`** — Yapilandirilmis TSA listesi
   ```csharp
   foreach (var p in sdk.ListTsaProviders().Where(p => p.IsEnabled))
       Console.WriteLine($"{p.Key} ({p.AuthType}): {p.Url}");
   ```

5. **`VerifyTokenPin(libPath, pin, slotId)`** — PIN dogrulugunu imzalamadan test et
   ```csharp
   var status = sdk.GetTokenPinStatus(libPath);
   if (!status.IsLocked && !status.IsFinalTry && sdk.VerifyTokenPin(libPath, pin))
       Console.WriteLine("PIN dogru");
   ```

6. **`ListSlotCertificates(libPath, slotId)`** — Token sertifikalari (PIN gerektirmez)
   ```csharp
   var certs = sdk.ListSlotCertificates(libPath);
   foreach (var c in certs) Console.WriteLine($"{c.Subject} — {c.NotAfter:d}");
   ```

Detay: [SDK_REFERANS.md](SDK_REFERANS.md)

#### 16 Yeni Java gRPC Metodu

Java SDK tam REST parity saglandi:
`prepareSignature`, `finalizeSignature`, `listFormats`, `listValidationProfiles`, `enqueueSign`, `getSignJob`, `checkSignatureType`, `inspectSignatures`, `verifyKep`, `verifyEypDetail`, `checkReady`, `signAndCreateKep`, `signEypAndCreateKep`, `verifyKepDelil`, `batchCreateKep`, `getTokenSession`

#### 5 Yeni REST Endpoint

- `POST /verify/check` — Format/level hizli kontrol
- `POST /verify/inspect` — Imza envanteri
- `GET /timestamp/providers` — TSA listesi
- `POST /token/pin/verify` — PIN dogrulama
- `GET /token/slots/{slotId}/certificates` — Slot sertifikalari
- `GET /health/ready` — Readiness probe (Kubernetes)

Detay: [API_REFERANS.md](API_REFERANS.md)

#### OOXML Cross-Platform Destegi

OOXML imzalama artik tum platformlarda calisir:
- v1.x: Sadece Windows (`net10.0-windows` TFM)
- v2.0: BouncyCastle + manuel XML-DSig ile Linux, macOS, Windows

```csharp
await sdk.SignDataWithProviderAsync(docxData, provider, new SignatureParameters
{
    Format = SignatureFormat.OOXML,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});
```

---

### 🔒 Guvenlik

- **Dokumantasyon temizligi:** Hardcoded sifreler kaldirildi, tum ornek kodlar `Environment.GetEnvironmentVariable(...)` pattern'i kullaniyor
- **Production `appsettings.json`:** KamuSM `Password` alani bilerek bos; env var veya secrets store'dan doldurulur
- **Uyari:** Sifre bos kalirsa KamuSM identity header atlanir ve baglanti sessizce basarisiz olur. `CheckReadinessAsync()` ile TSA erisilebilirligini kontrol edin.

---

### 🚀 Performans

- **Paralel batch pipeline:** HSM + 8 worker ile 1000 PDF ~= 15s
- **Session pooling:** Token/HSM oturumlari yeniden kullaniliyor (`CreateTokenSessionAsync`)
- **gRPC vs REST:** Buyuk belgelerde (>5MB) gRPC %30-50 daha hizli

Detay olcum: `tests/DigitalSignature.Tests/PerformanceBenchmarkTests.cs` (urun deposu)

---

### 📦 Dagitim (MBox Release)

v2.0 release'inde yeni arsiv formatlari:

| Asset | Icerik |
|---|---|
| `digimr-linux-amd64.tar.gz` | Linux binary (self-contained) |
| `digimr-docker.tar.gz` | Docker image (gzipped) |
| `digimr-sdk-*-all.jar` | Java SDK (uber JAR) |
| `digimr-docs.tar.gz` | **YENI** — 15 public .md dokumani |
| `digimr-dotnet-examples.tar.gz` | **YENI** — .NET ornek projeleri (Eyp, KepSaglayici) |
| `digimr-java-examples.tar.gz` | **YENI** — 10 Java ornek program |
| `docker-compose.demo.yml` | Demo compose dosyasi |
| `.env.example` | Ortam degiskenleri sablonu |

---

### 🔄 Migration Checklist

v1.x'ten v2.0'a gecis icin:

- [ ] **Kod:** `SignPdfWithProviderAsync("path.pdf", ...)` → `SignDataWithProviderAsync(File.ReadAllBytes("path.pdf"), ...)`
- [ ] **Kod:** `SignXmlWithProviderAsync("path.xml", ...)` → `SignDataWithProviderAsync(...)`
- [ ] **Kod:** `ValidateDocumentAsync("path")` → `ValidateDocumentAsync(File.ReadAllBytes("path"))`
- [ ] **Kod:** `SigningProviderType.Certificate` → `.Software`
- [ ] **Kod:** `SigningProviderType.Hsm` → `.HSM`
- [ ] **Kod:** `AuthenticationContext.Pkcs12Data` → `.CertificateData`
- [ ] **Kod:** `AuthenticationContext.Password` → `.CertificatePassword`
- [ ] **Kod:** `ASiCDocument.Data = ...` → `ASiCDocument.Content = ..., MimeType = "..."`
- [ ] **Kod:** `SignBatchAsync` — `BatchItem` → `SignBatchItem`, donus tipi `List<SignatureResult>`
- [ ] **Kod:** `PrepareSignatureAsync(data, params, X509Certificate)` → `PrepareSignatureAsync(data, params, certData: certBytes, chainData: chainBytesArray)`
- [ ] **Yapilandirma:** `appsettings.json` — KamuSM `Password` alanini env var'a tasi
- [ ] **Test:** Derleme + test suite calistir
- [ ] **Dokumantasyon:** Icinizdeki notlari `docs/public/` altindaki yeni referanslara yonlendirin

---

### 🐛 Bug Fixleri

- KamuSM TSA identity header bos sifre durumunda sessizce atliyordu → artik `CheckReadinessAsync` ile tespit edilebilir
- OOXML imzalama Windows-only idi → cross-platform BouncyCastle
- gRPC servisleri bazi alanlari dondurmuyordu → REST ile tam parity
- PKCS11 parallel test conflict → fixed (commit `f96f6f1`)

---

## v1.x — Oncesi

v2.0 oncesi surumler icin detayli changelog yok. Onemli kilometre taslari:

- **2026-04-11 — Phase 1:** .NET SDK refactor, unified API surface (f8499c1)
- **2026-04-12 — Phase 2:** REST API SDK-ify (b0f6452)
- **2026-04-12 — Phase 3:** gRPC full REST parity (5479aa7, d16a632)
- **2026-04-13 — Phase 4:** Java SDK full gRPC parity (13ba039, cf6f810)
- **2026-04-14 — Phase 5:** Java eToken hardware tests (f96f6f1)

---

## Referanslar

- [SDK_REFERANS.md](SDK_REFERANS.md) — Tam API referansi
- [API_REFERANS.md](API_REFERANS.md) — REST API referansi
- [ORNEKLER.md](ORNEKLER.md) — 17 kullanim senaryosu
- [ARCHITECTURE.md](ARCHITECTURE.md) — Sistem mimarisi diyagramlari
- [GENEL_BAKIS.md](GENEL_BAKIS.md) — Hizli navigasyon
- [English CHANGELOG](../en/CHANGELOG.md)
