# DigiMR Dokümantasyonu

**SDK Versiyon:** 2.0 | **Son Guncelleme:** 2026-04-17 | **Durum:** Stabil

DigiMR, Türkiye'nin nitelikli elektronik imza altyapısı için kapsamlı bir SDK + REST/gRPC API sistemidir. Bu dokümantasyon müşteri entegrasyon ekiplerine yöneliktir.

---

## 30 Saniyede Başla

```csharp
using DigitalSignature.SDK;
using DigitalSignature.Core.Models;

var sdk = new DigitalSignatureSDK();
sdk.SetLicenseFile("digimr-sdk.lic");

using var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificateData = File.ReadAllBytes("sertifika.pfx"),
        CertificatePassword = Environment.GetEnvironmentVariable("PFX_PASSWORD")!
    });

var pdfData = File.ReadAllBytes("belge.pdf");
var result = await sdk.SignDataWithProviderAsync(pdfData, provider, new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_B
});

File.WriteAllBytes("belge-imzali.pdf", result.SignedData!);
```

Detay: [ORNEKLER.md — Başlangıç](ORNEKLER.md#baslangic)

---

## Hızlı Navigasyon

| Amacınız | Doküman |
|-------|---------|
| **Hemen kod yaz** | [ORNEKLER.md](ORNEKLER.md) — 17 senaryo (C# + curl) |
| **Kavramları öğren** | [KULLANIM_KILAVUZU.md](KULLANIM_KILAVUZU.md) — Rehber anlatım |
| **Ne zaman ne kullanayım?** | [SENARYO_REHBERI.md](SENARYO_REHBERI.md) — Karar diyagramları |
| **SDK referansı** (.NET) | [SDK_REFERANS.md](SDK_REFERANS.md) — 23 bölüm + 4 Appendix |
| **REST API referansı** | [API_REFERANS.md](API_REFERANS.md) — 20 bölüm + 3 Appendix |

---

## Sürüm 2.0'da Yenilik

**6 yeni SDK metodu:**
- [`CheckReadinessAsync()`](SDK_REFERANS.md#21-health-ve-diagnostics) — Sunucu hazır mı (lisans, TSA, trust store)?
- [`CheckSignatureTypeAsync()`](SDK_REFERANS.md#7-dogrulama) — Format/level hızlı kontrol
- [`InspectSignaturesAsync()`](SDK_REFERANS.md#7-dogrulama) — İmza envanteri (sertifika, zaman, rol)
- [`ListTsaProviders()`](SDK_REFERANS.md#9-zaman-damgasi) — Yapılandırılmış TSA listesi
- [`VerifyTokenPin()`](SDK_REFERANS.md#17-providertoken) — İmzalamadan PIN doğrulama
- [`ListSlotCertificates()`](SDK_REFERANS.md#17-providertoken) — Token sertifika listesi

**5 yeni REST endpoint:**
- `POST /verify/check`, `POST /verify/inspect`
- `GET /timestamp/providers`
- `POST /token/pin/verify`, `GET /token/slots/{id}/certificates`
- `GET /health/ready`

**Temel değişiklikler (migration gerekli):**
- String-path overload'lar kaldırıldı — yalnız `byte[]` versiyonlar kaldı
- Tam liste: [SDK_REFERANS.md Appendix D](SDK_REFERANS.md#appendix-d-silinen-metotlar-v1x--v20-gocus)
- Migration rehberi: [CHANGELOG.md](CHANGELOG.md)

---

## Hedef Kitleye Göre Yol Haritası

### Geliştirici — SDK Entegrasyonu

1. [ORNEKLER.md — Başlangıç](ORNEKLER.md) — ilk 4 örneği kopyala-çalıştır
2. [SDK_REFERANS.md Bölüm 1-5](SDK_REFERANS.md) — hızlı başlangıç + tam metot tablosu
3. [KULLANIM_KILAVUZU.md](KULLANIM_KILAVUZU.md) — konsept anlatımı
4. [SAGLAYICILAR.md](SAGLAYICILAR.md) — 8 sağlayıcı tipi (Token, HSM, CloudHSM, Mobile, ESeal, RemoteToken, Biometric, Software)

### Geliştirici — REST/gRPC Entegrasyonu

1. [API_REFERANS.md Bölüm 2](API_REFERANS.md) — endpoint özet tablosu
2. [ORNEKLER.md İleri Senaryo 12](ORNEKLER.md#senaryo-12-ebys-entegrasyonu-aspnet-core-http-akisi) — ASP.NET Core controller örneği
3. [API_REFERANS.md Appendix C](API_REFERANS.md) — gRPC yansıması (port 7702)

### Operasyon / DevOps

1. [KURULUM_REHBERI.md](KURULUM_REHBERI.md) — Docker, Kubernetes, sunucu kurulumu
2. [AGENT_KURULUM.md](AGENT_KURULUM.md) — Agent kurulumu (masaüstü, tray, port, SSL)
3. [API_REFERANS.md Bölüm 19](API_REFERANS.md) — Health endpoints (`/health`, `/health/ready`)

### Entegrasyon — Özel Konular

1. [KAMUSM_TSA_ENTEGRASYONU.md](KAMUSM_TSA_ENTEGRASYONU.md) — Kamu SM zaman damgası (RFC 3161 + identity)
2. [KEP_ENTEGRASYONU.md](KEP_ENTEGRASYONU.md) — Kayıtlı Elektronik Posta
3. [MOBIL_IMZA.md](MOBIL_IMZA.md) — Mobil imza (Turkcell, Vodafone, Türk Telekom)
4. [IMZA_FORMATLARI.md](IMZA_FORMATLARI.md) — CAdES/PAdES/XAdES/JAdES/ASiC detayı

### Uyum / Hukuk

1. [TURK_MEVZUATI.md](TURK_MEVZUATI.md) — Türk mevzuatı (5070, BTK, P1-P4 profilleri)
2. [CHANGELOG.md](CHANGELOG.md) — Sürüm geçmişi

### Test / Kalite

1. [DOGRULAMA_ARACLARI.md](DOGRULAMA_ARACLARI.md) — DSS Demo ve Türk doğrulayıcıları

---

## Desteklenen Formatlar ve Seviyeler

| Format | Standart | Desteklenen Seviyeler |
|--------|----------|-----------|
| **PAdES** | ETSI EN 319 142 | B-B, B-T, B-LT, B-LTA |
| **CAdES** | ETSI EN 319 122 | B-B, B-T, B-C, B-X, B-LT, B-LTA |
| **XAdES** | ETSI EN 319 132 | B-B, B-T, B-LT, B-LTA |
| **JAdES** | ETSI TS 119 182 | B-B, B-T |
| **ASiC-S / ASiC-E** | ETSI EN 319 162 | B-T, B-LT, B-LTA |
| **OOXML** | ISO/IEC 29500 | B-T |
| **EYP** | CBDDO V2.0 / V1.3 | OPC + CAdES (mühür: B-LTA) |
| **KEP** | 7201 sayılı Kanun | CAdES-T + ZIP |
| **UETS** | PTT UETS | CAdES-T + ZIP |

---

## Mimari

İki kullanım modeli:

```
In-process SDK (yerel imza)            API sunucusu (uzak / diğer diller)
  ├─ .NET SDK  (DigitalSignature.SDK)    ├─ REST API  (:7701)
  └─ Java SDK  (saf Java, java-native)   └─ gRPC API  (:7702)
```

- **.NET SDK**: in-process ana kütüphane (`IDigitalSignatureSDK`) — imzayı yerelde üretir.
- **Java SDK**: in-process **saf Java** kütüphanesi (`sdk/java-native`, BouncyCastle/PDFBox) — .NET SDK ile aynı modelde imzayı yerelde üretir; modül servisleri (CadesSigner, XAdesSigner, PadesSigner, EYP, …). _Saf-Java SDK release'i hazırlanıyor._
- **Java gRPC istemcisi**: API'yi uzaktan çağıran ince istemci (`digimr-sdk-1.0.0-all.jar`) — imza sunucuda üretilir. v2.3.2 ile yeniden yayında; `sdk/java-native` çıkana kadar Java'dan API'ye bağlanmanın hazır yolu.
- **REST API**: HTTP/JSON API (port 7701) — tüm SDK metotlarının HTTP yansıması (sunucu gerektirir).
- **gRPC API**: Binary protokol (port 7702) — yüksek performans, streaming (sunucu gerektirir).

Detay: [docs/ARCHITECTURE.md](ARCHITECTURE.md) (diagramlar)

---

## SSS

**S: Lisanssız kullanabilir miyim?**
C: 30 gün deneme modu otomatik aktif. Sonrasında `.lic` dosyası gerekli. Lisans almak: `info@moreum.com`

**S: Hangi token'larla uyumlu?**
C: Tüm PKCS#11 uyumlu token'lar — SafeNet eToken, Akis, Gemalto, Feitian, Cardwerk. Detay: [SAGLAYICILAR.md](SAGLAYICILAR.md)

**S: Linux sunucuda çalışır mı?**
C: Evet. REST/gRPC API + Java SDK tam Linux uyumlu. Kurulum: [KURULUM_REHBERI.md](KURULUM_REHBERI.md)

**S: KamuSM TSA bağlanamıyor, ne yapmalıyım?**
C: Şifre kontrolü. Production `appsettings.json` bilerek boş şifre ile gelir. Env var gerekli: `TsaProviders__Providers__kamusm__Password=<şifre>`. Detay: [KAMUSM_TSA_ENTEGRASYONU.md](KAMUSM_TSA_ENTEGRASYONU.md)

**S: Mevcut v1.x kodumu nasıl v2.0'a geçiririm?**
C: Migration rehberi: [CHANGELOG.md](CHANGELOG.md) ve [SDK_REFERANS.md Appendix D](SDK_REFERANS.md).

---

## Destek

- **GitHub Issues** (müşteri repo): `moreum-tech/MBox`
- **E-posta**: `info@moreum.com`
- **Teknik dokümantasyon PDF paketi**: MBox release asset'lerinde `digimr-docs.tar.gz`
