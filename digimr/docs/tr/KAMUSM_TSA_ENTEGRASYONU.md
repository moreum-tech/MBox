# Kamu SM Zaman Damgası Entegrasyonu

**SDK Versiyon:** 2.0 | **Son Guncelleme:** 2026-04-17

## Genel Bakış

DigiMR, Kamu SM (TÜBİTAK BİLGEM) zaman damgası servisini destekler. B-T ve üstü imza seviyelerinde zaman damgası zorunludur.

## Sunucu Adresleri

| Ortam | URL | Açıklama |
|-------|-----|----------|
| **Test** | `http://tzd.kamusm.gov.tr` | Test ortamı — geliştirme ve entegrasyon testleri için |
| **Üretim** | `http://zd.kamusm.gov.tr` | Yasal geçerli zaman damgası — 5070 sayılı Kanun uyumlu |
| **Test (SHA-512)** | `http://tzdsha512.kamusm.gov.tr` | SHA-512 hash algoritması test ortamı |

> **Önemli:** URL'lerin sonuna `/ts` veya başka path **eklemeyin**. Doğru kullanım: `http://tzd.kamusm.gov.tr`

## Başvuru

Kamu SM zaman damgası kullanmak için kurumsal başvuru gereklidir:
1. [zdportal.kamusm.gov.tr](https://zdportal.kamusm.gov.tr) adresinden başvuru yapın
2. Başvuru onaylandığında **UserId** ve **Password** alırsınız
3. Kredi satın alın — her zaman damgası 1 kredi harcar

## DigiMR ile Kullanım

### SDK ile

```csharp
// SDK otomatik olarak Kamu SM kimlik doğrulamasını yönetir
var sdk = new DigitalSignatureSDK();
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext { Pin = "<PIN>", Pkcs11LibraryPath = "eTPKCS11.dll" });

// B-T imza (zaman damgalı)
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"  // test ortamı
});
```

#### SDK Factory Metodu ile KamuSM Yapılandırması

MA3 API'den geçiş yapan kullanıcılar için SDK, doğrudan `TsaProviderConfig.KamuSM()` factory metodu sunar:

```csharp
// MA3 API karşılığı:
// TSSettings settings = new TSSettings("http://tzd.kamusm.gov.tr", userId, password, DigestAlg.SHA256);
// DigiMR SDK karşılığı:
sdk.ConfigureTsa("kamusm", TsaProviderConfig.KamuSM(
    "http://tzd.kamusm.gov.tr", 7521, Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));
```

> **Not:** `TsaProviderConfig.KamuSM()` metodu `AuthType`, `PolicyOid` ve kimlik doğrulama başlıklarını otomatik olarak ayarlar. Ayrıca parametreleri `appsettings.json`'a yazmadan doğrudan kod içinde yapılandırmanıza olanak tanır.

#### SDK ile Çoklu TSA Sağlayıcı Yapılandırması

SDK üzerinden birden fazla TSA sağlayıcıyı programatik olarak tanımlayabilirsiniz:

```csharp
sdk.ConfigureTsa(new TsaProvidersOptions {
    Default = "kamusm",
    Providers = {
        ["kamusm"] = TsaProviderConfig.KamuSM("http://tzd.kamusm.gov.tr", 7521, Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!),
        ["digicert"] = TsaProviderConfig.Anonymous("http://timestamp.digicert.com"),
        ["turktrust"] = TsaProviderConfig.BasicAuth("http://zd.turktrust.com.tr", "user", "pass")
    }
});
```

Desteklenen factory metodları:

| Metod | Kimlik Doğrulama | Kullanım |
|-------|------------------|----------|
| `TsaProviderConfig.KamuSM(url, userId, password)` | Kamu SM Identity Header | TÜBİTAK Kamu SM servisleri |
| `TsaProviderConfig.Anonymous(url)` | Kimlik doğrulama yok | DigiCert gibi açık TSA servisleri |
| `TsaProviderConfig.BasicAuth(url, username, password)` | HTTP Basic Auth | TÜRKTRUST gibi kullanıcı/şifre gerektiren servisler |

#### SDK ile Zaman Damgası Güven Zinciri Doğrulama

Alınan zaman damgasının güvenilir bir kök sertifikaya bağlı olup olmadığını doğrulamak için `TimestampValidator` kullanılabilir:

```csharp
var result = await TimestampValidator.ValidateAsync(
    tokenBytes, hash, sdk.TrustStore);
// result.IsChainTrusted - güven zinciri doğrulandı mı
// result.TrustAnchorSubject - kök sertifika bilgisi
```

> **Neden önemli?** Zaman damgasının sadece geçerli bir yapıda olması yetmez; güven zincirinin (trust chain) tanınmış bir kök sertifikaya kadar izlenebilmesi gerekir. Bu doğrulama, zaman damgasının sahte veya güvenilmeyen bir kaynaktan gelmediğini garanti eder.

### REST API ile

```bash
curl -X POST https://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "<base64-pdf>",
    "format": "PAdES",
    "parameters": {
      "level": "B_T",
      "tsaUrl": "http://tzd.kamusm.gov.tr"
    },
    "provider": {
      "type": "token",
      "pkcs11LibraryPath": "C:/Windows/System32/eTPKCS11.dll",
      "slotId": 0,
      "pin": "1234"
    }
  }'
```

### Yapılandırma (`appsettings.json`)

```json
{
  "TsaProviders": {
    "Default": "kamusm",
    "Providers": {
      "kamusm": {
        "Name": "TÜBİTAK Kamu SM",
        "Url": "http://tzd.kamusm.gov.tr",
        "AuthType": "kamusm-identity",
        "UserId": 7521,
        "Password": "",
        "PolicyOid": "2.16.792.1.2.1.1.5.7.3.1",
        "TimeoutSeconds": 30,
        "Enabled": true
      }
    }
  }
}
```

> **Güvenlik:** Password'ü `appsettings.json`'a yazmak yerine environment variable kullanın:
> `TsaProviders__Providers__kamusm__Password=şifreniz`

### Kredi Sorgulama

```csharp
// SDK üzerinden
var tsClient = new TSClient(new TSSettings("http://tzd.kamusm.gov.tr", 7521, Environment.GetEnvironmentVariable("KAMUSM_PASSWORD")!));
int kalan = tsClient.requestRemainingCredit();
Console.WriteLine($"Kalan kredi: {kalan}");
```

```bash
# REST API üzerinden
curl -X POST https://localhost:7701/api/v1/timestamp/providers \
  -H "Content-Type: application/json"
```

## Çoklu TSA Desteği (API Yapılandırması)

DigiMR birden fazla TSA sağlayıcı destekler. `TsaProviderKey` ile istediğiniz sağlayıcıyı seçebilirsiniz. SDK ile programatik yapılandırma için yukarıdaki [SDK ile Çoklu TSA Sağlayıcı Yapılandırması](#sdk-ile-çoklu-tsa-sağlayıcı-yapılandırması) bölümüne bakın.

```json
{
  "TsaProviders": {
    "Default": "digicert",
    "Providers": {
      "kamusm": {
        "Name": "TÜBİTAK Kamu SM",
        "Url": "http://zd.kamusm.gov.tr",
        "AuthType": "kamusm-identity",
        "UserId": 7521,
        "Password": "",
        "PolicyOid": "2.16.792.1.2.1.1.5.7.3.1",
        "Enabled": true
      },
      "digicert": {
        "Name": "DigiCert",
        "Url": "http://timestamp.digicert.com",
        "AuthType": "none",
        "Enabled": true
      },
      "turktrust": {
        "Name": "TÜRKTRUST",
        "Url": "http://zd.turktrust.com.tr",
        "AuthType": "http-basic",
        "Username": "",
        "Password": "",
        "Enabled": false
      }
    }
  }
}
```

## Test vs Üretim

| | Test | Üretim |
|---|---|---|
| URL | `http://tzd.kamusm.gov.tr` | `http://zd.kamusm.gov.tr` |
| Yasal geçerlilik | ❌ Yok | ✅ 5070 sayılı Kanun |
| Kredi | Test kredisi (sınırlı) | Satın alınan kredi |
| Kullanım | Geliştirme, entegrasyon testleri | Üretim imzalama |

> **Dikkat:** Test ortamında oluşturulan zaman damgaları yasal geçerlilik taşımaz. Üretimde `zd.kamusm.gov.tr` kullanın.

## Hata Giderme

| Belirti | Olası Sebep | Çözüm |
|---------|-------------|-------|
| Bağlantı kesildi (TCP RST) | Kimlik doğrulama eksik | UserId ve Password doğru yapılandırın |
| `Status: 2 (Rejection)` | Kredi yetersiz veya şifre yanlış | [zdportal.kamusm.gov.tr](https://zdportal.kamusm.gov.tr) üzerinden kredi ve şifre kontrol edin |
| Timeout | Sunucu yanıt vermiyor | TimeoutSeconds değerini artırın, ağ bağlantısını kontrol edin |
| `Policy OID hata` | Policy OID eksik | DigiMR otomatik ekler — yapılandırmada `PolicyOid` alanını boş bırakmayın |

## Kaynaklar

- [Kamu SM Zaman Damgası Başvurusu](https://kamusm.bilgem.tubitak.gov.tr/basvurular/zd/)
- [Kamu SM ZD Portal (Kredi/Hesap Yönetimi)](https://zdportal.kamusm.gov.tr)
- [Kamu SM Zaman Damgası Nasıl Çalışır?](https://kamusm.bilgem.tubitak.gov.tr/urunler/zaman_damgasi/kamu_sm_zaman_damgasi_nasil_calisir.jsp)
- [Kamu SM Yazılım Platformu](https://yazilim.kamusm.gov.tr)
