# DigiMR Kullanim Ornekleri

**SDK Versiyon:** 2.0
**Son Guncelleme:** 2026-04-17

Bu dokuman SDK'nin gercek kullanim senaryolarini 3 seviyede sunar: Baslangic (yeni kullanicilar, kutu-icinden ornekler), Orta (gelistirici, tipik kullanim), Ileri (production is akislari).

**On Kosul:**
- SDK kurulu: `dotnet add package DigitalSignature.SDK`
- Lisans yuklu (veya 30 gun deneme modunda)
- Gercek kontrat ve API detaylari icin [SDK_REFERANS.md](SDK_REFERANS.md)

**Guvenlik:** Ornek kodlarda sifreler/PIN'ler ortam degiskenlerinden okunur. Asla kod icine yazmayin.

---

## Icindekiler

### Baslangic
1. PDF imzala (en basit yol — PKCS#12 sertifika)
2. Token ile imzala (akilli kart / USB token)
3. Imzayi dogrula
4. Seviye yukselt (B-B → B-T)

### Orta Seviye
5. Toplu imzalama (batch)
6. ASiC paketi olustur
7. EYP paketi olustur
8. JAdES imzalama
9. Imza envanteri (InspectSignatures) — YENI 2.0
10. TSA provider listesi (ListTsaProviders) — YENI 2.0
11. Token PIN dogrulama (VerifyTokenPin + GetTokenPinStatus) — YENI 2.0

### Ileri Seviye
12. EBYS entegrasyonu (ASP.NET Core HTTP akisi)
13. Toplu imzalama pipeline'i (paralel + Channel)
14. Iki asamali imza (Prepare/Finalize)
15. EYP: Ayri signer + seal sertifikasi
16. HSM + agent remote imzalama
17. Preservation (LTA arsiv yenileme)
18. Long-lived session — Web oturumu boyunca coklu imza

---

## Baslangic

### Ornek 1: PDF imzala (en basit — PKCS#12 sertifika)

Yazilim sertifikasi (PKCS#12 / .pfx dosyasi) ile PDF'i B-B seviyesinde imzala. Sifre environment variable'dan okunur.

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;

var sdk = new DigitalSignatureSDK();

// Sifreyi ASLA kod icinde yazmayin — env var
var pfxPassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")
    ?? throw new InvalidOperationException("PFX_PASSWORD env var tanimlanmamis");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("sertifika.pfx"),
        CertificatePassword = pfxPassword
    });

var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_B
});

if (result.Success)
    File.WriteAllBytes("belge-imzali.pdf", result.SignedData!);
else
    Console.Error.WriteLine($"Hata: {result.Error}");
```

**Beklenen:** `belge-imzali.pdf` olusur, PAdES-B imzasi icerir.

**Onemli noktalar:**
- `SigningProviderType.Software` — PKCS#12 icin dogru enum (Certificate DEGIL)
- `CertificateData` + `CertificatePassword` — dogru alan isimleri
- `using` block — provider dispose edilmeli (token/HSM kaynaklarini serbest birakir)

---

### Ornek 2: Token ile imzala (akilli kart / USB token)

PKCS#11 token ile B-T (zaman damgali) imzalama. PIN ve TSA sifresi env var'dan.

```csharp
var tokenPin = Environment.GetEnvironmentVariable("TOKEN_PIN")!;
var kamusmPassword = Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!;

var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM(
    url: "http://tzd.kamusm.gov.tr",
    userId: 7521,
    password: kamusmPassword));

// PIN durum kontrolu — kilitli mi, son deneme mi
var pinStatus = sdk.GetTokenPinStatus(@"C:\Windows\System32\eTPKCS11.dll");
if (pinStatus.IsLocked)
    throw new Exception($"Token kilitli: {pinStatus.Message}");
if (pinStatus.IsFinalTry)
    Console.WriteLine("DIKKAT: Son deneme hakki!");

// Authenticate
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = tokenPin,
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll",
        SlotId = 0
    });

// Imzala (B-T, TSA zaman damgasi dahil)
var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr",
    HashAlgorithm = "SHA256",
    Reason = "Onay",
    Location = "Ankara"
});

if (result.Success)
    File.WriteAllBytes("belge-imzali.pdf", result.SignedData!);
```

**Beklenen:** PAdES-B-T imza (kriptografik + TSA zaman damgasi).

**Uyari:** PIN kilitleme riski — `GetTokenPinStatus` kontrolu ATLAMA. Memory feedback'te belirtildi.

---

### Ornek 3: Imzayi dogrula

Imzali belgeyi dogrula, bulunan imzalarin detaylarini yazdir.

```csharp
var sdk = new DigitalSignatureSDK();
var data = File.ReadAllBytes("imzali.pdf");

var result = await sdk.ValidateDocumentAsync(data);

if (result.IsValid)
{
    Console.WriteLine($"Gecerli: {result.Signatures.Count} imza, format: {result.Format}");
    foreach (var sig in result.Signatures)
    {
        Console.WriteLine($"- #{sig.Index} {sig.Role}");
        Console.WriteLine($"  Imzalayan: {sig.SignerSubject}");
        Console.WriteLine($"  Zaman: {sig.SignatureTime:yyyy-MM-dd HH:mm}");
        Console.WriteLine($"  Level: {sig.Level}, Hash: {sig.HashAlgorithm}");
    }
}
else
{
    foreach (var err in result.Errors)
        Console.Error.WriteLine($"HATA: {err}");
    foreach (var warn in result.Warnings)
        Console.WriteLine($"UYARI: {warn}");
}
```

**Beklenen:** Her imza icin sertifika sahibi, zaman, level, hash algoritmasi.

**Ek:** Grace period ile dogrulama icin `ValidationOptions` kullanin (SDK_REFERANS Bolum 7.1).

---

### Ornek 4: Seviye yukselt (B-B → B-T)

Mevcut B-B imzasina TSA zaman damgasi ekleyerek B-T yap.

```csharp
var sdk = new DigitalSignatureSDK();
var kamusmPassword = Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!;
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521, kamusmPassword));

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

**Not:** Format otomatik tespit edilir (PAdES, CAdES, XAdES). JAdES icin ayri metot: `UpgradeJadesAsync` (string return).

---

## Orta Seviye

### Ornek 5: Toplu imzalama (batch)

Ayni provider ile coklu PDF'i tek authentication ile imzala.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")!,
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
    });

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

Directory.CreateDirectory("pdf-outbox");
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
Console.WriteLine($"{successCount}/{items.Count} imzalandi");
```

**Beklenen:** pdf-outbox/ altinda imzali dosyalar. Tek token authentication'la coklu imza.

---

### Ornek 6: ASiC paketi olustur

ASiC-E: Coklu belgeyi tek konteynerde gruplandir + imzala.

```csharp
var kamusmPassword = Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!;
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521, kamusmPassword));

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("sertifika.pfx"),
        CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
    });

var docs = new[]
{
    new ASiCDocument
    {
        FileName = "sozlesme.pdf",
        Content = File.ReadAllBytes("sozlesme.pdf"),
        MimeType = "application/pdf"
    },
    new ASiCDocument
    {
        FileName = "ek-1.pdf",
        Content = File.ReadAllBytes("ek-1.pdf"),
        MimeType = "application/pdf"
    }
};

var container = await sdk.CreateAsicEAsync(docs, provider, new SignatureParameters
{
    Format = SignatureFormat.ASiC_E,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});

File.WriteAllBytes("paket.asice", container);
```

**Beklenen:** paket.asice (ZIP-tabanli konteyner) — icinde belgeler + CAdES imza + manifest.

---

### Ornek 7: EYP paketi olustur

TS 13298 V2.0 EYP paketi — elektronik yazisma standardi.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521,
    Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));

using var signerProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")!,
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
    });

var options = new EypCreateOptions
{
    UstYazi = File.ReadAllBytes("ustyazi.pdf"),
    Ustveri = "<?xml version=\"1.0\"?><ustveri>...</ustveri>",  // XML ustveri
    Ekler = new List<EypEk>
    {
        new EypEk { FileName = "ek-1.pdf", Content = File.ReadAllBytes("ek-1.pdf") }
    },
    TsaUrl = "http://tzd.kamusm.gov.tr"
};

var result = await sdk.CreateEypPackageV20Async(options, signerProvider);

if (result.Success)
    File.WriteAllBytes("paket.eyp", result.PackageData!);
```

**Not:** EypCreateOptions alan adlari kaynak `IEypPackageService.cs`'e dayanir (UstYazi, Ustveri, Ekler, TsaUrl, ImzaLevel, MuhurLevel). `EypEk` class — detaylar SDK_REFERANS Appendix A.

---

### Ornek 8: JAdES imzalama

JSON Web Signature tabanli imza — REST API yanitlarini imzalamak icin.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("sertifika.pfx"),
        CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
    });

var apiResponse = Encoding.UTF8.GetBytes("{\"orderId\": 12345, \"status\": \"approved\"}");
var jadesCompact = await sdk.SignJadesAsync(apiResponse, provider, new SignatureParameters
{
    Format = SignatureFormat.JAdES,
    Level = SignatureLevel.B_B,
    HashAlgorithm = "SHA256"
});

Console.WriteLine($"JAdES: {jadesCompact}");
File.WriteAllText("imza.jws", jadesCompact);
```

**Beklenen:** `eyJ...` ile baslayan compact JWS string. Client tarafinda `VerifyJades(jadesCompact)` ile dogrulanir.

---

### Ornek 9: Imza envanteri — InspectSignatures  [YENI 2.0]

Belgedeki tum imzalarin ayrintili envanteri — her imza icin imzalayan, zaman, level, hash algoritmasi.

```csharp
var sdk = new DigitalSignatureSDK();
var data = File.ReadAllBytes("coklu-imzali.pdf");

var inspection = await sdk.InspectSignaturesAsync(data);

Console.WriteLine($"Format: {inspection.DetectedFormat}");
Console.WriteLine($"Toplam imza: {inspection.SignatureCount}");
foreach (var sig in inspection.Signatures)
{
    Console.WriteLine($"\n#{sig.Index} [{sig.Role}]");
    Console.WriteLine($"  Imzalayan: {sig.SignerSubject}");
    Console.WriteLine($"  Zaman: {sig.SignatureTime:yyyy-MM-dd HH:mm}");
    Console.WriteLine($"  Level: {sig.Level}, Hash: {sig.HashAlgorithm}");
    Console.WriteLine($"  Gecerli: {sig.IsValid}");
    if (sig.Errors.Any())
        Console.WriteLine($"  Hatalar: {string.Join("; ", sig.Errors)}");
}
if (inspection.Deficiencies.Any())
    Console.WriteLine($"\nPaket uyarilari: {string.Join("; ", inspection.Deficiencies)}");
```

**Kullanim alani:** Audit, imza zinciri analizi, karsi taraf sertifika dogrulamasi. Tam dogrulama icin `ValidateDocumentAsync` kullanin.

---

### Ornek 10: TSA provider listesi — ListTsaProviders  [YENI 2.0]

Yapilandirilmis TSA saglayicilarini listele ve kullanilabilirligini kontrol et.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.ConfigureTsa(new TsaProvidersOptions
{
    Default = "kamusm",
    Providers = new Dictionary<string, TsaProviderConfig>
    {
        ["kamusm"] = TsaProviderConfig.KamuSM(
            "http://tzd.kamusm.gov.tr", 7521,
            Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!),
        ["freetsa"] = TsaProviderConfig.Anonymous("https://freetsa.org/tsr"),
        ["turktrust"] = TsaProviderConfig.BasicAuth(
            "https://tsp.turktrust.com.tr", "user",
            Environment.GetEnvironmentVariable("TURKTRUST_PASSWORD")!)
    }
});

var providers = sdk.ListTsaProviders();
Console.WriteLine($"Toplam: {providers.Count} saglayici\n");
foreach (var p in providers)
{
    var defaultMark = p.IsDefault ? " [default]" : "";
    var enabledMark = p.IsEnabled ? "" : " [disabled]";
    Console.WriteLine($"{p.Key}: {p.Name}{defaultMark}{enabledMark}");
    Console.WriteLine($"  URL: {p.Url}");
    Console.WriteLine($"  Auth: {p.AuthType}");
}
```

**Kullanim:** Runtime'da hangi TSA'larin mevcut oldugunu listelemek, health check, kullaniciya secenek sunmak.

---

### Ornek 11: Token PIN dogrulama — VerifyTokenPin + GetTokenPinStatus  [YENI 2.0]

Imzalama yapmadan once PIN'i guvenli sekilde dogrula. Token kilitleme riskini azaltmak icin once status kontrolu.

```csharp
var sdk = new DigitalSignatureSDK();
var libPath = @"C:\Windows\System32\eTPKCS11.dll";

// 1. Once PIN durumunu kontrol et — kilit riski varsa deneme yapma
var status = sdk.GetTokenPinStatus(libPath);
if (status.IsLocked)
{
    Console.Error.WriteLine($"HATA: Token kilitli. {status.Message}");
    Console.Error.WriteLine("PUK ile cozunuz veya kart saglayicisina basvurunuz.");
    return;
}
if (status.IsFinalTry)
{
    Console.Error.WriteLine("UYARI: SON DENEME HAKKI — yanlis PIN kartinizi kilitler!");
    // Kullaniciya onay sor, otomatik deneme yapma
    Console.Write("Devam etmek istiyor musunuz? (evet/hayir): ");
    if (Console.ReadLine()?.Trim().ToLower() != "evet") return;
}

// 2. Kullanicidan PIN al
Console.Write("Token PIN: ");
var userPin = Console.ReadLine();

// 3. PIN'i dogrula (imzalama yapmaz, sadece test eder)
if (sdk.VerifyTokenPin(libPath, userPin!))
{
    Console.WriteLine("PIN dogru — imzalamaya devam edilebilir");
    // Simdi `CreateAndAuthenticateProviderAsync` ile provider olustur
}
else
{
    Console.WriteLine("PIN yanlis — tekrar deneyin");
    // GetTokenPinStatus tekrar cagrilmali (deneme hakki azaldi)
}
```

**Kritik:** `VerifyTokenPin` yanlis PIN ile cagrilirsa token'in deneme sayacini azaltir. Tekrar-tekrar yanlis PIN = kilit. `GetTokenPinStatus` kontrolu ATLAMAYIN.

---

## Ileri Seviye

### Senaryo 12: EBYS Entegrasyonu (ASP.NET Core HTTP akisi)

Web uygulamasindan gelen belgeyi sunucuda imzala ve geri don.

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/ebys")]
public class SigningController : ControllerBase
{
    private readonly IDigitalSignatureSDK _sdk;
    public SigningController(IDigitalSignatureSDK sdk) => _sdk = sdk;

    public record SignRequest(string PdfBase64, string Pin, int DocumentId);
    public record SignResponse(bool Success, string? SignedPdfBase64, string? Error);

    [HttpPost("sign")]
    public async Task<ActionResult<SignResponse>> Sign([FromBody] SignRequest req)
    {
        try
        {
            using var provider = await _sdk.CreateAndAuthenticateProviderAsync(
                SigningProviderType.Token,
                new AuthenticationContext
                {
                    Pin = req.Pin,
                    Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
                });

            var pdfBytes = Convert.FromBase64String(req.PdfBase64);
            var result = await _sdk.SignDataWithProviderAsync(pdfBytes, provider, new SignatureParameters
            {
                Format = SignatureFormat.PAdES,
                Level = SignatureLevel.B_T,
                TsaUrl = "http://tzd.kamusm.gov.tr",
                Reason = $"EBYS Onay #{req.DocumentId}",
                HashAlgorithm = "SHA256"
            });

            return result.Success
                ? Ok(new SignResponse(true, Convert.ToBase64String(result.SignedData!), null))
                : BadRequest(new SignResponse(false, null, result.Error));
        }
        catch (Exception ex)
        {
            return StatusCode(500, new SignResponse(false, null, ex.Message));
        }
    }
}

// Startup.cs veya Program.cs:
// builder.Services.AddSingleton<IDigitalSignatureSDK>(sp => {
//     var sdk = new DigitalSignatureSDK();
//     sdk.SetLicenseFile("digimr-sdk.lic");
//     sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM(
//         "http://tzd.kamusm.gov.tr", 7521,
//         Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));
//     return sdk;
// });
```

**Guvenlik:** Request body'de PIN dolasiyor — HTTPS zorunlu, log'a yazmayin. Alternatif: Client-side Prepare/Finalize akisi (Senaryo 14).

---

### Senaryo 13: Paralel Batch Pipeline (Channel + Worker)

Yuksek hacimli toplu imzalama — paralel worker'lar + Channel ile iletisim.

```csharp
using System.Threading.Channels;

var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.HSM,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("HSM_PIN")!,
        Pkcs11LibraryPath = "/usr/lib/softhsm/libsofthsm2.so"
    });

var parameters = new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_B
};

// 1000 belge, 8 paralel worker
var inputChannel = Channel.CreateBounded<(string id, byte[] data)>(100);
var outputChannel = Channel.CreateBounded<(string id, SignatureResult result)>(100);

// Producer
_ = Task.Run(async () =>
{
    foreach (var file in Directory.EnumerateFiles("pdf-inbox/", "*.pdf"))
        await inputChannel.Writer.WriteAsync((Path.GetFileName(file), File.ReadAllBytes(file)));
    inputChannel.Writer.Complete();
});

// Workers
var workers = Enumerable.Range(0, 8).Select(_ => Task.Run(async () =>
{
    await foreach (var (id, data) in inputChannel.Reader.ReadAllAsync())
    {
        var result = await sdk.SignDataWithProviderAsync(data, provider, parameters);
        await outputChannel.Writer.WriteAsync((id, result));
    }
})).ToArray();

_ = Task.WhenAll(workers).ContinueWith(_ => outputChannel.Writer.Complete());

// Consumer — sonuclari diske yaz
int success = 0, failed = 0;
await foreach (var (id, result) in outputChannel.Reader.ReadAllAsync())
{
    if (result.Success)
    {
        File.WriteAllBytes($"pdf-outbox/{id}", result.SignedData!);
        success++;
    }
    else failed++;
}

Console.WriteLine($"Tamamlandi: {success} basarili, {failed} basarisiz");
```

**Not:** HSM tipik olarak session pooling destekler; `SigningProviderType.HSM` pool'dan session alir. Standart `Token` bu olceklemede calismaz (tek oturum).

---

### Senaryo 14: Iki Asamali Imza (Prepare/Finalize — External Signer)

HSM veya uzak imza servisinin sertifikasiyla imzalama. SDK hash uretir, external servis imzalar, SDK belgeye yerlestirir.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

// 1. Sertifika bytes (DER-encoded) — external signer'dan biliniyor
var certBytes = File.ReadAllBytes("signer.cer");
var chainBytes = new[]
{
    File.ReadAllBytes("intermediate-ca.cer"),
    File.ReadAllBytes("root-ca.cer")
};

// 2. Prepare — hash'i uret
var pdfData = File.ReadAllBytes("belge.pdf");
var prepare = await sdk.PrepareSignatureAsync(
    pdfData,
    new SignatureParameters
    {
        Format = SignatureFormat.PAdES,
        Level = SignatureLevel.B_B,
        HashAlgorithm = "SHA256"
    },
    certData: certBytes,
    chainData: chainBytes);

if (!prepare.Success)
    throw new Exception($"Prepare basarisiz: {prepare.Error}");

Console.WriteLine($"Hash ({prepare.HashAlgorithm}): {Convert.ToBase64String(prepare.Hash!)}");

// 3. External signer'a hash'i gonder
// (HSM cagrisi, uzak imza servisi, hardware token vb.)
byte[] externalSignature = await CallRemoteSignerAsync(
    hash: prepare.Hash!,
    hashAlgorithm: prepare.HashAlgorithm!);

// 4. Finalize — imzayi belgeye yerlestir
var result = await sdk.FinalizeSignatureAsync(prepare, externalSignature);

if (result.Success)
    File.WriteAllBytes("imzali.pdf", result.SignedData!);
else
    Console.Error.WriteLine($"Finalize basarisiz: {result.Error}");

// External signer stub (gercek: PKCS#11, REST API, Azure Key Vault vb.)
static async Task<byte[]> CallRemoteSignerAsync(byte[] hash, string hashAlgorithm)
{
    // HTTP ornek:
    using var client = new HttpClient();
    var response = await client.PostAsJsonAsync("https://signer.example.com/sign",
        new { hash = Convert.ToBase64String(hash), algorithm = hashAlgorithm });
    var body = await response.Content.ReadFromJsonAsync<Dictionary<string, string>>();
    return Convert.FromBase64String(body!["signature"]);
}
```

**Kullanim alanlari:**
- Yuksek guvenlikli HSM'de private key asla cikmaz; sadece hash imzalanir
- Mobil imza senaryosu (telefon ile OTP onay + SIM kart imzasi)
- Regulasyon gerektirdigi merkezi imza servisi

**SDK Referansi:** Bolum 20.1 (PrepareSignatureAsync) ve 20.2 (FinalizeSignatureAsync)

---

### Senaryo 15: EYP — Ayri Signer + Seal Sertifikasi

EYP paketi olustururken imzalayici kullanicinin + kurumsal muhur sertifikasi (e-seal) ayri ayri.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521,
    Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));

// 1. Imzalayici provider (Token / PKCS#12)
using var signerProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("TOKEN_PIN")!,
        Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
    });

// 2. Muhur (e-seal) provider — sunucu HSM
using var sealProvider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.HSM,
    new AuthenticationContext
    {
        Pin = Environment.GetEnvironmentVariable("HSM_SEAL_PIN")!,
        Pkcs11LibraryPath = "/usr/lib/softhsm/libsofthsm2.so",
        KeyLabel = "corporate-seal-2026"
    });

// 3. EYP olustur — signer ustyazi imzalar, seal paket butunune muhur basar
var options = new EypCreateOptions
{
    UstYazi = File.ReadAllBytes("resmi-yazi.pdf"),
    Ustveri = File.ReadAllText("ustveri.xml"),
    Ekler = new List<EypEk>
    {
        new EypEk { FileName = "ek-1.pdf", Content = File.ReadAllBytes("ek-1.pdf") }
    },
    TsaUrl = "http://tzd.kamusm.gov.tr",
    ImzaLevel = SignatureLevel.B_T,
    MuhurLevel = SignatureLevel.B_LTA  // muhur icin uzun donem arsiv
};

var result = await sdk.CreateEypPackageV21Async(options, signerProvider, sealProvider);

if (result.Success)
    File.WriteAllBytes("paket.eyp", result.PackageData!);
```

**Hukuki Arka Plan:** TS 13298'de ustyazi imzasi + paket muhuru iki farkli islem. Signer: gercek kisi (memur). Seal: kurum (otomatik).

---

### Senaryo 16: HSM + Remote Agent ile Imzalama

Uzak bir sunucudaki HSM'e Token Agent uzerinden baglanma.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

// Uzak Token Agent'taki token'lari listele
var remoteTokens = await sdk.QueryRemoteTokensAsync(
    agentUrl: "https://agent.corporate.com",
    apiKey: Environment.GetEnvironmentVariable("AGENT_API_KEY")!);

Console.WriteLine($"Uzak token'lar: {remoteTokens.Count}");
foreach (var t in remoteTokens.Where(t => t.IsPresent))
    Console.WriteLine($"  Slot {t.SlotId}: {t.Label} ({t.SerialNumber})");

// Belirli bir token ile remote authentication
using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.RemoteToken,
    new AuthenticationContext
    {
        AgentUrl = "https://agent.corporate.com",
        AgentApiKey = Environment.GetEnvironmentVariable("AGENT_API_KEY")!,
        Pin = Environment.GetEnvironmentVariable("REMOTE_TOKEN_PIN")!,
        SlotId = 0
    });

var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
});

File.WriteAllBytes("imzali.pdf", result.SignedData!);
```

**Kullanim:** Muhasebe/IT uzmaninin kendisi merkez ofise gidip HSM'e baglanmak yerine uzaktan imza. TLS zorunlu, API key rotasyonu yapilir.

---

### Senaryo 17: Preservation — LTA Arsiv Yenileme

B-LTA seviyeli imzalarin arsiv zaman damgasi sureli — yenileme gerekir.

```csharp
var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521,
    Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));

// Arsiv dosyalarini tara
var archiveDir = "/var/archive/lta/";
var renewedDir = "/var/archive/lta-renewed/";
Directory.CreateDirectory(renewedDir);

foreach (var file in Directory.EnumerateFiles(archiveDir, "*.pdf"))
{
    var data = File.ReadAllBytes(file);

    // Yenileme gerekli mi kontrol et
    var status = sdk.CheckPreservationStatus(data);
    if (!status.RenewalNeeded)
    {
        Console.WriteLine($"OK: {Path.GetFileName(file)} — {status.NextRenewal:yyyy-MM-dd}");
        continue;
    }

    Console.WriteLine($"YENILEME: {Path.GetFileName(file)}");
    var renewed = await sdk.RenewArchiveTimestampAsync(data, "http://tzd.kamusm.gov.tr");
    File.WriteAllBytes(Path.Combine(renewedDir, Path.GetFileName(file)), renewed);

    // Kanit kaydi al
    var evidence = sdk.GetEvidenceRecord(renewed);
    File.WriteAllText(Path.Combine(renewedDir, Path.GetFileName(file) + ".evidence.xml"),
        evidence.ToXmlString());
}
```

**Operasyonel Kural:** Arsiv yenileme takvimi — TSA sertifika gecerlilik suresinden onceki son 6 ay icinde yapilmali. Kanit kayitlari ayri dizinde saklanir (denetim icin).

---

### Senaryo 18: Long-lived Session — Web Oturumu Boyunca Coklu Imza

Kullanici web uygulamasina bir kez PIN girer, session acilir, oturum boyunca (10-50 imza) PIN tekrar sorulmadan belge imzalar. Web session sonunda token session kapatilir.

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/user-session")]
public class UserSigningController : ControllerBase
{
    private readonly IDigitalSignatureSDK _sdk;
    private readonly IMemoryCache _cache; // session -> userId mapping
    public UserSigningController(IDigitalSignatureSDK sdk, IMemoryCache cache)
    {
        _sdk = sdk; _cache = cache;
    }

    public record LoginRequest(string Pin);
    public record LoginResponse(string SessionId, int CertCount);
    public record SignRequest(string PdfBase64, string Reason);

    // 1. Kullanici oturumu ac — PIN bir kez
    [HttpPost("login")]
    public async Task<ActionResult<LoginResponse>> Login([FromBody] LoginRequest req)
    {
        var session = await _sdk.CreateTokenSessionAsync(
            SigningProviderType.Token,
            new AuthenticationContext
            {
                Pin = req.Pin,
                Pkcs11LibraryPath = @"C:\Windows\System32\eTPKCS11.dll"
            });

        var userId = User.Identity!.Name!;
        _cache.Set($"token-session:{userId}", session.SessionId,
            TimeSpan.FromMinutes(10)); // session TTL ile uyumlu

        var certs = _sdk.ListSessionCertificates(session.SessionId);
        return Ok(new LoginResponse(session.SessionId, certs.Count));
    }

    // 2. Oturum boyunca N imza (PIN tekrar sormadan)
    [HttpPost("sign")]
    public async Task<IActionResult> Sign([FromBody] SignRequest req)
    {
        var userId = User.Identity!.Name!;
        if (!_cache.TryGetValue<string>($"token-session:{userId}", out var sessionId))
            return Unauthorized("Token session yok veya suresi doldu — tekrar login yapin");

        // Session tabanli imzalama — SDK session'i otomatik bulur
        var session = _sdk.GetTokenSession(sessionId);
        // (Session-tabanli SignHashAsync API'si burada cagrilir;
        //  SDK implementasyonuna bagli olarak session yeniden provider olusturur veya
        //  pool'dan session reuse eder — kullanicidan PIN tekrar alinmaz)

        var pdfBytes = Convert.FromBase64String(req.PdfBase64);
        // ... imza mantigi ...

        return Ok();
    }

    // 3. Oturum kapat
    [HttpPost("logout")]
    public async Task<IActionResult> Logout()
    {
        var userId = User.Identity!.Name!;
        if (_cache.TryGetValue<string>($"token-session:{userId}", out var sessionId))
        {
            await _sdk.CloseTokenSessionAsync(sessionId);
            _cache.Remove($"token-session:{userId}");
        }
        return Ok();
    }
}
```

**Avantajlar:**
- PIN tek sefer (login), sonraki N imza kullanici deneyimini bozmadan
- Token kilitleme riski dusuk (tek login denemesi)
- Idle timeout ile otomatik temizlik (unuttulma durumu)

**Dikkat:**
- `MemoryCache` TTL'i token session TTL'i ile uyumlu olmali (default 5 dk, config ile arttirilabilir)
- Multi-node web'de `MemoryCache` yerine distributed cache (Redis) kullan — session pinning gerekir
- Logout'u zorunlu tut — browser kapatildiğinda beacon ile cagir

