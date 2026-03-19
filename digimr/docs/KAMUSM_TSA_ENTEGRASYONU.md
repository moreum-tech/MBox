# Kamu SM Zaman Damgası Entegrasyonu

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
    new AuthenticationContext { Pin = "1234", Pkcs11LibraryPath = "eTPKCS11.dll" });

// B-T imza (zaman damgalı)
var result = await sdk.SignPdfAsync(pdfData, provider, new SignatureParameters
{
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"  // test ortamı
});
```

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
var tsClient = new TSClient(new TSSettings("http://tzd.kamusm.gov.tr", 7521, "şifre"));
int kalan = tsClient.requestRemainingCredit();
Console.WriteLine($"Kalan kredi: {kalan}");
```

```bash
# REST API üzerinden
curl -X POST https://localhost:7701/api/v1/timestamp/providers \
  -H "Content-Type: application/json"
```

## Çoklu TSA Desteği

DigiMR birden fazla TSA sağlayıcı destekler. `TsaProviderKey` ile istediğiniz sağlayıcıyı seçebilirsiniz:

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
