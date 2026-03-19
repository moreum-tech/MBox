# DigiMR Digital Signature API — Kullanım Kılavuzu

**Versiyon**: 2.0
**Son Güncelleme**: Mart 2026
**Base URL**: `http://localhost:7701/api/v1`

---

## İçindekiler

1. [Kurulum ve Yapılandırma](#1-kurulum-ve-yapılandırma)
2. [Genel Bilgiler](#2-genel-bilgiler)
3. [Provider Tipleri](#3-provider-tipleri)
4. [İmzalama API](#4-i̇mzalama-api)
5. [Toplu İmzalama API](#5-toplu-i̇mzalama-api)
6. [Doğrulama API](#6-doğrulama-api)
7. [Seviye Yükseltme API](#7-seviye-yükseltme-api)
8. [Zaman Damgası API](#8-zaman-damgası-api)
9. [EYP API](#9-eyp-api)
10. [KEP API](#10-kep-api)
11. [Provider Keşif API](#11-provider-keşif-api)
12. [Sağlık Kontrolü API](#12-sağlık-kontrolü-api)
13. [HSM Kullanımı](#13-hsm-kullanımı)
14. [Mobil İmza](#14-mobil-i̇mza)
15. [Token Oturum API](#15-token-oturum-api)
16. [Çoklu İmza](#16-çoklu-i̇mza-aynı-belgede-birden-fazla-i̇mza)
17. [İki Aşamalı İmzalama](#17-i̇ki-aşamalı-i̇mzalama-preparefinalize)
18. [Format Keşif API](#18-format-keşif-api)
19. [Agent Hub](#19-agent-hub-signalr)
20. [Hata Kodları ve Rate Limiting](#20-hata-kodları-ve-rate-limiting)

---

## 1. Kurulum ve Yapılandırma

### Gereksinimler

- .NET 8.0 Runtime
- Windows veya Linux
- PKCS#11 token kullanılacaksa token sürücüsü

### Çalıştırma

```bash
# Kaynak koddan
cd src/DigitalSignature.API
dotnet run

# Yayınlanmış binary'den
dotnet DigitalSignature.API.dll

# Varsayılan port: 7701
# http://localhost:7701/api/v1/health
```

### appsettings.json Yapılandırması

```json
{
  "Kestrel": {
    "Endpoints": {
      "Local":    { "Url": "http://127.0.0.1:7701" },
      "External": { "Url": "http://0.0.0.0:7701" }
    }
  },

  "DefaultTsaUrl": "http://tzd.kamusm.gov.tr",
  "TimestampServer": {
    "UserId":   7521,
    "Password": "3VilLuA8"
  },

  "TsaProviders": {
    "Default": "kamusm",
    "Providers": {
      "kamusm": {
        "Name":         "TÜBİTAK Kamu SM",
        "Url":          "http://tzd.kamusm.gov.tr",
        "AuthType":     "kamusm-identity",
        "UserId":       7521,
        "Password":     "3VilLuA8",
        "PolicyOid":    "2.16.792.1.2.1.1.5.7.3.1",
        "TimeoutSeconds": 30,
        "Enabled":      true
      },
      "turktrust": {
        "Name":     "TÜRKTRUST",
        "Url":      "http://zd.turktrust.com.tr",
        "AuthType": "http-basic",
        "Username": "",
        "Password": "",
        "Enabled":  false
      },
      "eguven": {
        "Name":     "E-Güven",
        "Url":      "http://ts384.e-guven.com",
        "AuthType": "http-basic",
        "Username": "",
        "Password": "",
        "Enabled":  false
      },
      "edmkep": {
        "Name":    "EDM KEP",
        "Url":     "",
        "Enabled": false
      },
      "eimzatr": {
        "Name":    "E-İmzaTR",
        "Url":     "",
        "Enabled": false
      }
    }
  },

  "TokenSession": {
    "IdleTimeoutMinutes":      5,
    "AcquireTimeoutSeconds":  30,
    "CleanupIntervalSeconds": 30
  },

  "Security": {
    "MaxDocumentSizeBytes":      104857600,
    "MaxCertificateSizeBytes":   10485760,
    "MaxBatchDocuments":         100,
    "MaxBatchTotalSizeBytes":    524288000,
    "RequestBodySizeLimitBytes": 524288000,
    "RateLimitPerMinute":        100
  }
}
```

**TSA AuthType değerleri:**

| Değer | Açıklama |
|---|---|
| `kamusm-identity` | TÜBİTAK KAMUSM'e özel PBKDF2+AES protokolü |
| `http-basic` | HTTP Basic Auth (`Authorization: Basic ...`) |
| `none` | Kimlik doğrulama yok (açık TSA'lar) |

**Yeni TSA sağlayıcısı etkinleştirmek:**
1. `appsettings.json`'da ilgili provider'a `"Enabled": true` yaz
2. `Username`/`Password` ya da `UserId`/`Password` gir
3. Uygulamayı yeniden başlat

---

## 2. Genel Bilgiler

### İmza Formatları

| Format | Açıklama | Kullanım |
|---|---|---|
| `PAdES` | PDF Advanced Electronic Signatures | PDF imzalama |
| `CAdES` | CMS Advanced Electronic Signatures | Her tür dosya, detached .p7s |
| `XAdES` | XML Advanced Electronic Signatures | XML dosyaları, e-Defter, e-Fatura |

### İmza Seviyeleri

| Seviye | Açıklama | TSA gerekli? |
|---|---|---|
| `B_B` | Temel imza | Hayır |
| `B_T` | İmza + zaman damgası | Evet |
| `B_LT` | B_T + CRL/OCSP revocation verisi | Evet |
| `B_LTA` | B_LT + arşiv zaman damgası | Evet |

### Veri Formatları

Tüm belgeler ve sertifikalar **base64** kodlanmış olarak gönderilir.

```bash
# Dosyayı base64'e çevir (Linux/macOS)
base64 -i belge.pdf -o belge_b64.txt

# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("belge.pdf"))
```

---

## 3. Provider Tipleri

Provider, imzada kullanılacak **özel anahtarın** nerede durduğunu belirtir.

### 3.1 Software (Yazılım Sertifikası)

PKCS#12 (.p12 / .pfx) dosyasından imzalama. Özel anahtar **sunucuya gönderilir**, üretim için önerilmez.

```json
{
  "Type": "software",
  "CertificateBase64": "<.p12 dosyasının base64>",
  "CertificatePassword": "p12_sifresi"
}
```

### 3.2 Token (USB e-İmza Token)

Lokal PKCS#11 token (AKiS, SafeNet, Gemalto vb.). **Özel anahtar token'da kalır**, sunucuya gönderilmez.

```json
{
  "Type": "token",
  "Pin": "1234",
  "Pkcs11LibraryPath": "/usr/lib/libakis.so",
  "SlotId": 0
}
```

> **PIN Modları** — `Metadata` ile ek kontrol:
>
> | PinMode | Davranış |
> |---|---|
> | `SessionStart` (varsayılan) | Oturum başında bir kez giriş |
> | `PerSignature` | Her imzada PIN sorulur |
> | `Cached` | PIN TTL kadar önbelleğe alınır (`PinCacheTtlSeconds`) |
> | `ProtectedPath` | PIN kart okuyucudan girilir |
> | `AutoDetect` | Token yeteneklerine göre otomatik |

```json
{
  "Type": "token",
  "Pin": "1234",
  "Pkcs11LibraryPath": "/usr/lib/libakis.so",
  "SlotId": 0,
  "Metadata": {
    "PinMode": "Cached",
    "PinCacheTtlSeconds": "300"
  }
}
```

### 3.3 HSM (Donanımsal Güvenlik Modülü)

Kurumsal HSM'ler için. Token provider ile aynı PKCS#11 arayüzünü kullanır, ek olarak oturum havuzu desteği sunar.

```json
{
  "Type": "hsm",
  "Pin": "hsm_pin",
  "Pkcs11LibraryPath": "/usr/lib/libCryptoki2_64.so",
  "SlotId": 0,
  "Metadata": {
    "HsmPoolSize": "5"
  }
}
```

### 3.4 Remote Token (Uzak Token Agent)

Özel anahtarın **başka bir makinedeki** fiziksel token'da olduğu durum. DigiMR Agent servisi aracılık eder.

```json
{
  "Type": "remote-token",
  "AgentUrl": "https://agent-sunucu:5555",
  "AgentApiKey": "agent_api_key",
  "Pin": "1234",
  "CertificateIndex": 0
}
```

**Token nerede durur?** Özel anahtar hiçbir zaman agent sunucusunu terk etmez. İmzalanacak hash agent'a gönderilir, imza geri döner.

### 3.5 Mobile (Mobil İmza)

Turkcell, Vodafone veya Türk Telekom üzerinden mobil imza. Operatör SMS/mobil uygulama ile kullanıcıdan onay alır.

```json
{
  "Type": "mobile",
  "MobileNumber": "5551234567",
  "Operator": "Turkcell"
}
```

> Mobil imza için operatörle ayrıca sözleşme yapılması gerekir.

---

## 4. İmzalama API

### POST /api/v1/sign

**İstek:**

```json
{
  "documentBase64": "<belge_base64>",
  "format": "PAdES | CAdES | XAdES | ASiC-S | ASiC-E | EYP | KEP",
  "provider": {
    "type": "software | token | hsm | remote-token | mobile",
    "...": "provider'a özel alanlar"
  },
  "parameters": {
    "level": "B_B | B_T | B_LT | B_LTA",
    "tsaUrl": "http://tzd.kamusm.gov.tr",
    "tsaProviderKey": "kamusm | turktrust | eguven",
    "reason": "Belge onayı",
    "location": "İstanbul, Türkiye",
    "contactInfo": "imza@sirket.com",
    "hashAlgorithm": "SHA256 | SHA384 | SHA512",
    "signaturePolicy": "P1 | P2 | P3 | P4",
    "productionPlace": {
      "city": "İstanbul",
      "stateOrProvince": "İstanbul",
      "postalCode": "34000",
      "countryName": "TR"
    },
    "signerRole": {
      "claimedRoles": ["Müdür", "Yetkili İmza"]
    },
    "commitmentType": "ProofOfApproval | ProofOfCreation | ProofOfReceipt",
    "visualSignature": { "...": "..." }
  }
}
```

**parameters Alan Referansı:**

| Alan | Tür | Açıklama |
|------|-----|----------|
| `level` | string | İmza seviyesi: `B_B` (temel), `B_T` (+zaman damgası), `B_LT` (+CRL/OCSP), `B_LTA` (+arşiv) |
| `tsaUrl` | string | Özel TSA URL override |
| `tsaProviderKey` | string | Yapılandırılmış TSA sağlayıcısı: `kamusm`, `turktrust`, `eguven` |
| `reason` | string | İmzalama nedeni (PDF ve XAdES meta) |
| `location` | string | İmzalama yeri |
| `contactInfo` | string | İmzalayan iletişim bilgisi |
| `hashAlgorithm` | string | `SHA256` (varsayılan), `SHA384`, `SHA512` |
| `signaturePolicy` | string | Türk BTK profili: `P1`–`P4` (PAdES/CAdES EPES için) |
| `productionPlace` | object | `city`, `stateOrProvince`, `postalCode`, `countryName` |
| `signerRole` | object | `{ claimedRoles: [...] }` — imzalayanın rolü |
| `commitmentType` | string | `ProofOfApproval`, `ProofOfCreation`, `ProofOfReceipt` vb. |
| `visualSignature` | object | PDF görünür imza ayarları (bkz. aşağısı) |

**Yanıt:**

```json
{
  "success": true,
  "signedDocumentBase64": "<imzali_belge_base64>",
  "message": "PAdES B_T imza başarılı"
}
```

---

### Senaryo: Yazılım Sertifikası ile PDF İmzalama (B-T)

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i belge.pdf)'",
    "Format": "PAdES",
    "Provider": {
      "Type": "software",
      "CertificateBase64": "'$(base64 -i sertifika.p12)'",
      "CertificatePassword": "p12_sifresi"
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr",
      "Reason": "Sözleşme onayı",
      "Location": "İstanbul"
    }
  }'
```

---

### Senaryo: USB Token ile PDF İmzalama

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i belge.pdf)'",
    "Format": "PAdES",
    "Provider": {
      "Type": "token",
      "Pin": "1234",
      "Pkcs11LibraryPath": "/usr/lib/libakis.so",
      "SlotId": 0
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr"
    }
  }'
```

---

### Senaryo: Mali Mühür ile XML İmzalama (e-Defter / XAdES-T)

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i edefter.xml)'",
    "Format": "XAdES",
    "Provider": {
      "Type": "token",
      "Pin": "1234",
      "Pkcs11LibraryPath": "/usr/lib/libakis.so",
      "SlotId": 0
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr"
    }
  }'
```

---

### Senaryo: Remote Token (Uzak Sunucudaki Token)

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i belge.pdf)'",
    "Format": "PAdES",
    "Provider": {
      "Type": "remote-token",
      "AgentUrl": "https://agent.sirket.local:5555",
      "AgentApiKey": "gizli_api_anahtari",
      "Pin": "1234",
      "CertificateIndex": 0
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr"
    }
  }'
```

---

### Senaryo: Görünür İmza (Stamp/Mühür) ile PDF

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i belge.pdf)'",
    "Format": "PAdES",
    "Provider": {
      "Type": "software",
      "CertificateBase64": "'$(base64 -i sertifika.p12)'",
      "CertificatePassword": "p12_sifresi"
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr",
      "VisualSignature": {
        "Mode": "StampOnly",
        "Page": 1,
        "X": 100,
        "Y": 100,
        "Width": 200,
        "Height": 80,
        "StampText": "ONAYLANDI",
        "StampStyle": "RoundedRectangle",
        "StampColor": "#003399"
      }
    }
  }'
```

**VisualSignature Mode değerleri:**

| Mode | Açıklama |
|---|---|
| `TextOnly` | Sadece metin |
| `ImageOnly` | Sadece görsel |
| `TextAndImage` | Metin + görsel |
| `StampOnly` | Mühür görünümü |

**StampStyle değerleri:** `Rectangle`, `RoundedRectangle`, `Oval`, `Badge`, `Ribbon`

---

## 5. Toplu İmzalama API

### POST /api/v1/sign/batch

Tek PIN girişiyle birden fazla belge imzalar.

**İstek:**

```json
{
  "Documents": [
    { "Id": "fatura-001", "DocumentBase64": "<belge1_base64>" },
    { "Id": "fatura-002", "DocumentBase64": "<belge2_base64>" },
    { "Id": "fatura-003", "DocumentBase64": "<belge3_base64>" }
  ],
  "Format": "PAdES",
  "Provider": {
    "Type": "token",
    "Pin": "1234",
    "Pkcs11LibraryPath": "/usr/lib/libakis.so"
  },
  "Parameters": {
    "Level": "B_T",
    "TsaUrl": "http://tzd.kamusm.gov.tr"
  }
}
```

**Yanıt:**

```json
{
  "Results": [
    {
      "Id": "fatura-001",
      "Success": true,
      "SignedDocumentBase64": "<imzali_base64>",
      "Error": null
    },
    {
      "Id": "fatura-002",
      "Success": false,
      "SignedDocumentBase64": null,
      "Error": "Belge bozuk"
    }
  ],
  "Summary": {
    "Total": 3,
    "Succeeded": 2,
    "Failed": 1,
    "ElapsedMs": 4521.3
  }
}
```

**Limitler:** Maksimum 100 belge, toplam 500 MB.

---

## 6. Doğrulama API

### POST /api/v1/verify

Format otomatik algılanır (PAdES, CAdES, XAdES, EYP, ASiC). Tam doğrulama sonucu döner.

**İstek:**

```json
{
  "documentBase64": "<imzali_belge_base64>",
  "validationOptions": {
    "gracePeriodSeconds": 0,
    "trustSigningTimeAttribute": false
  }
}
```

**Yanıt:**

```json
{
  "isValid": true,
  "format": "PAdES",
  "isDocumentIntact": true,
  "validationState": "MATURE",
  "isPremature": false,
  "signatures": [
    {
      "signerSubject": "CN=Ahmet Yılmaz, O=Şirket A.Ş., C=TR",
      "certificateSerial": "1A2B3C4D",
      "signatureTime": "2026-03-04T15:30:00Z",
      "isValid": true,
      "level": "B_T",
      "hashAlgorithm": "SHA256",
      "hasTimestamp": true,
      "timestampTime": "2026-03-04T15:30:01Z",
      "errors": [],
      "warnings": []
    }
  ],
  "errors": [],
  "warnings": []
}
```

**validationState Değerleri:**

| Değer | Açıklama |
|-------|----------|
| `MATURE` | Doğrulama kesinleşmiştir; iptal bilgisi yayılmış olmalı |
| `PREMATURE` | İmza çok yakın zamanda atılmış; iptal bilgisi henüz yayılmamış olabilir |
| `null` | `gracePeriodSeconds` yapılandırılmamış |

```bash
curl -X POST http://localhost:7701/api/v1/verify \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'$(base64 -i imzali.pdf)'"}'
```

---

### POST /api/v1/verify/check

Belgede belirli bir format/seviyenin mevcut olup olmadığını kontrol eder.

**İstek:**

```json
{
  "documentBase64": "<imzali_belge_base64>",
  "format": "PAdES",
  "level": "B_T"
}
```

**Yanıt:**

```json
{
  "hasMatch": true,
  "detectedFormat": "PAdES",
  "matchReason": "PAdES B-T imzası bulundu",
  "matchedSignatures": 1,
  "deficiencies": []
}
```

```bash
curl -X POST http://localhost:7701/api/v1/verify/check \
  -H "Content-Type: application/json" \
  -d '{ "documentBase64": "'$(base64 -i imzali.pdf)'", "format": "PAdES", "level": "B_T" }'
```

---

### POST /api/v1/verify/inspect

Belgede hangi imza tiplerinin bulunduğunu listeler (tam envanter).

**İstek:**

```json
{
  "documentBase64": "<imzali_belge_base64>"
}
```

**Yanıt:**

```json
{
  "detectedFormat": "PAdES",
  "validSignatureTypes": ["PAdES-BASELINE-B", "PAdES-BASELINE-T"],
  "signatureInventory": [
    {
      "index": 0,
      "signerSubject": "CN=Ahmet Yılmaz",
      "level": "B_T",
      "hasTimestamp": true
    }
  ],
  "deficiencyReport": [],
  "summary": "1 geçerli imza bulundu. En yüksek seviye: B-T"
}
```

---

### POST /api/v1/verify/kep

KEP paketi detaylı doğrulama.

### POST /api/v1/verify/eyp

EYP paketi detaylı doğrulama.

---

## 7. Seviye Yükseltme API

### POST /api/v1/upgrade

Mevcut imzanın seviyesini yükseltir: `B_B → B_T → B_LT → B_LTA`

**İstek:**

```json
{
  "DocumentBase64": "<imzali_belge_base64>",
  "TargetLevel": "B_T",
  "TsaUrl": "http://tzd.kamusm.gov.tr"
}
```

**Yanıt:**

```json
{
  "success": true,
  "upgradedDocumentBase64": "<yukseltilmis_base64>",
  "detectedFormat": "PAdES",
  "previousLevel": "B_B",
  "newLevel": "B_T",
  "sizeBytes": 125432
}
```

```bash
# B-B imzayı B-T'ye yükselt
curl -X POST http://localhost:7701/api/v1/upgrade \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i imzali_bb.pdf)'",
    "TargetLevel": "B_T",
    "TsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

---

## 8. Zaman Damgası API

Zaman damgası imzadan bağımsız olarak kullanılabilir. Merkezi KAMUSM hesabı tüm istekler için kullanılır.

### POST /api/v1/timestamp/token

Herhangi bir veri veya hash için RFC 3161 zaman damgası token'ı alır.

**İstek:**

```json
{
  "DataBase64": "<veri_base64>",
  "HashAlgorithm": "SHA256",
  "Provider": "kamusm"
}
```

veya önceden hash hesaplanmışsa:

```json
{
  "HashBase64": "<sha256_hash_base64>",
  "HashAlgorithm": "SHA256",
  "Provider": "kamusm"
}
```

**Yanıt:**

```json
{
  "success": true,
  "timestampTokenBase64": "<rfc3161_token_base64>",
  "provider": "kamusm",
  "providerName": "TÜBİTAK Kamu SM",
  "hashAlgorithm": "SHA256",
  "tokenSizeBytes": 3842,
  "genTime": "2026-03-04T15:30:01Z",
  "serialNumber": "1A2B3C4D5E6F"
}
```

```bash
# PDF dosyasına zaman damgası (imzasız)
curl -X POST http://localhost:7701/api/v1/timestamp/token \
  -H "Content-Type: application/json" \
  -d '{"DataBase64": "'$(base64 -i muhasebe_kaydi.pdf)'"}'

# Log dosyasına zaman damgası
curl -X POST http://localhost:7701/api/v1/timestamp/token \
  -H "Content-Type: application/json" \
  -d '{"DataBase64": "'$(base64 -i sistem_logu.txt)'"}'
```

**Zaman damgası nerede saklanır?**

- API **token'ı döndürür** — saklama sorumluluğu çağırana aittir
- Token ile orijinal verinin birlikte saklanması gerekir
- Doğrulama için her ikisi de gereklidir
- PDF'e gömülü damga için `/api/v1/sign` ile `Level: B_T` kullanın

### POST /api/v1/timestamp/verify

Token'ın orijinal veriyle eşleşip eşleşmediğini doğrular.

**İstek:**

```json
{
  "TimestampTokenBase64": "<token_base64>",
  "DataBase64": "<orijinal_veri_base64>",
  "HashAlgorithm": "SHA256"
}
```

**Yanıt:**

```json
{
  "isValid": true,
  "genTime": "2026-03-04T15:30:01Z",
  "serialNumber": "1A2B3C4D5E6F",
  "policyOid": "2.16.792.1.2.1.1.5.7.3.1",
  "hashAlgorithm": "SHA256",
  "tsaName": "TÜBİTAK KAMUSM TSA",
  "hasCertificates": true,
  "messageImprintBase64": "<hash_base64>"
}
```

### GET /api/v1/timestamp/providers

Aktif TSA sağlayıcıların listesi (kimlik bilgileri döndürülmez).

```bash
curl http://localhost:7701/api/v1/timestamp/providers
```

**Yanıt:**

```json
{
  "defaultProvider": "kamusm",
  "providers": [
    {
      "key": "kamusm",
      "name": "TÜBİTAK Kamu SM",
      "url": "http://tzd.kamusm.gov.tr",
      "authType": "kamusm-identity",
      "isDefault": true
    }
  ]
}
```

**Türkiye'de Yasal TSA Sağlayıcıları:**

| Key | Kurum | AuthType |
|---|---|---|
| `kamusm` | TÜBİTAK Kamu SM | `kamusm-identity` |
| `turktrust` | TÜRKTRUST | `http-basic` |
| `eguven` | E-Güven | `http-basic` |
| `edmkep` | EDM KEP | Belirlenmedi |
| `eimzatr` | E-İmzaTR | Belirlenmedi |

---

## 9. EYP API

EYP (Elektronik Yazışma Paketi) — resmi yazışma standardı.

### POST /api/v1/eyp/create

**İstek:**

```json
{
  "CoverDocument": {
    "FileName": "ustyazi.pdf",
    "ContentBase64": "<ustyazi_base64>",
    "MimeType": "application/pdf"
  },
  "Documents": [
    {
      "FileName": "ek-1.pdf",
      "ContentBase64": "<ek1_base64>",
      "MimeType": "application/pdf"
    }
  ],
  "Version": "V2_0",
  "SignatureLevel": "B-LT",
  "TsaUrl": "http://tzd.kamusm.gov.tr",
  "Metadata": {
    "DocumentNumber": "E-2026-001",
    "Subject": "Sözleşme Onayı",
    "Language": "tur"
  },
  "SignerProvider": {
    "Type": "token",
    "Pin": "1234",
    "Pkcs11LibraryPath": "/usr/lib/libakis.so"
  },
  "SealProvider": {
    "Type": "hsm",
    "Pin": "hsm_pin",
    "Pkcs11LibraryPath": "/usr/lib/libCryptoki2_64.so"
  }
}
```

**Yanıt:**

```json
{
  "success": true,
  "eypPackageBase64": "<eyp_base64>",
  "sizeBytes": 245678,
  "message": "EYP paketi oluşturuldu"
}
```

```bash
curl -X POST http://localhost:7701/api/v1/eyp/create \
  -H "Content-Type: application/json" \
  -d '{
    "CoverDocument": {
      "FileName": "ustyazi.pdf",
      "ContentBase64": "'$(base64 -i ustyazi.pdf)'",
      "MimeType": "application/pdf"
    },
    "Documents": [],
    "SignerProvider": {
      "Type": "software",
      "CertificateBase64": "'$(base64 -i sertifika.p12)'",
      "CertificatePassword": "p12_sifresi"
    },
    "SignatureLevel": "B-LT",
    "TsaUrl": "http://tzd.kamusm.gov.tr"
  }'
```

### POST /api/v1/eyp/verify

```bash
curl -X POST http://localhost:7701/api/v1/eyp/verify \
  -H "Content-Type: application/json" \
  -d '{"EypPackageBase64": "'$(base64 -i paket.eyp)'"}'
```

**Yanıt:**

```json
{
  "isValid": true,
  "isPackageStructureValid": true,
  "isPaketOzetiValid": true,
  "isNihaiOzetValid": true,
  "isImzaValid": true,
  "isMuhurValid": true,
  "areHashesValid": true,
  "imzaLevel": "B_LT",
  "muhurLevel": "B_LTA",
  "imzaciSubject": "CN=Ahmet Yılmaz, O=Kurum, C=TR",
  "errors": [],
  "warnings": []
}
```

### POST /api/v1/eyp/extract

```bash
curl -X POST http://localhost:7701/api/v1/eyp/extract \
  -H "Content-Type: application/json" \
  -d '{"EypPackageBase64": "'$(base64 -i paket.eyp)'"}'
```

**Yanıt:**

```json
{
  "packageId": "550e8400-e29b-41d4-a716-446655440000",
  "createdAt": "2026-03-04T10:00:00Z",
  "coverDocument": {
    "fileName": "ustyazi.pdf",
    "mimeType": "application/pdf",
    "sizeBytes": 123456,
    "contentBase64": "<base64>"
  },
  "attachments": [
    {
      "fileName": "ek-1.pdf",
      "mimeType": "application/pdf",
      "sizeBytes": 234567,
      "contentBase64": "<base64>"
    }
  ],
  "metadata": {
    "belgeNo": "E-2026-001",
    "konu": "Sözleşme Onayı",
    "tarih": "2026-03-04T10:00:00Z"
  }
}
```

---

## 10. KEP API

KEP (Kayıtlı Elektronik Posta) paketi oluşturma ve doğrulama.

### POST /api/v1/kep/create

```json
{
  "SenderAddress": "gonderen@kep-adresi.tr",
  "RecipientAddresses": ["alici1@kep.tr", "alici2@kep.tr"],
  "Subject": "Sözleşme Gönderimi",
  "Body": "Ekteki sözleşme onayınıza sunulmuştur.",
  "Type": "Icerik",
  "TsaUrl": "http://tzd.kamusm.gov.tr",
  "Attachments": [
    {
      "FileName": "sozlesme.pdf",
      "ContentBase64": "<base64>"
    }
  ],
  "Provider": {
    "Type": "token",
    "Pin": "1234",
    "Pkcs11LibraryPath": "/usr/lib/libakis.so"
  }
}
```

**Type değerleri:** `Gonderi`, `Kabul`, `Alindi`, `Icerik`

**Yanıt:**

```json
{
  "success": true,
  "kepPackageBase64": "<kep_zip_base64>",
  "kepId": "KEP-2026-00123",
  "sizeBytes": 345678,
  "attachmentCount": 1
}
```

### POST /api/v1/kep/verify

```bash
curl -X POST http://localhost:7701/api/v1/kep/verify \
  -H "Content-Type: application/json" \
  -d '{"KepPackageBase64": "'$(base64 -i paket.zip)'"}'
```

### POST /api/v1/kep/extract

```bash
curl -X POST http://localhost:7701/api/v1/kep/extract \
  -H "Content-Type: application/json" \
  -d '{"KepPackageBase64": "'$(base64 -i paket.zip)'"}'
```

---

## 11. Provider Keşif API

### GET /api/v1/providers

Desteklenen provider tiplerini listeler.

```bash
curl http://localhost:7701/api/v1/providers
```

### GET /api/v1/providers/tokens

Sunucuya bağlı lokal PKCS#11 token'ları keşfeder.

```bash
curl http://localhost:7701/api/v1/providers/tokens
```

**Yanıt:**

```json
[
  {
    "slotId": 0,
    "label": "AKiS Token",
    "manufacturerId": "Akis",
    "model": "AKiS v1.0",
    "serialNumber": "123456789",
    "library": { "path": "/usr/lib/libakis.so" }
  }
]
```

### POST /api/v1/providers/remote/tokens

Uzak agent'taki token'ları listeler.

```bash
curl -X POST http://localhost:7701/api/v1/providers/remote/tokens \
  -H "Content-Type: application/json" \
  -d '{
    "AgentUrl": "https://agent.sirket.local:5555",
    "ApiKey": "gizli_api_anahtari"
  }'
```

---

## 12. Sağlık Kontrolü API

### GET /api/v1/health

```bash
curl http://localhost:7701/api/v1/health
```

```json
{
  "status": "healthy",
  "timestamp": "2026-03-04T15:00:00Z",
  "version": "2.0.0"
}
```

### GET /api/v1/health/ready

TSA ve SDK bağımlılıklarını kontrol eder.

```bash
curl http://localhost:7701/api/v1/health/ready
```

```json
{
  "status": "ready",
  "timestamp": "2026-03-04T15:00:00Z",
  "checks": [
    { "Name": "TSA", "Status": "healthy", "Details": "kamusm reachable" },
    { "Name": "SDK",  "Status": "healthy" }
  ]
}
```

**HTTP 503** döndürürse bir bağımlılık sağlıksız demektir.

---

## 13. HSM Kullanımı

HSM (Hardware Security Module), kurumsal seviye özel anahtar saklama cihazıdır. SafeNet Luna, Thales, nCipher vb.

### appsettings.json'da HSM yapılandırması

```json
{
  "Hsm": {
    "LibraryPath": "/usr/lib/libCryptoki2_64.so",
    "SlotId": 0,
    "SessionPoolSize": 5
  }
}
```

### HSM ile İmzalama

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "DocumentBase64": "'$(base64 -i belge.pdf)'",
    "Format": "PAdES",
    "Provider": {
      "Type": "hsm",
      "Pin": "hsm_pin",
      "Pkcs11LibraryPath": "/usr/lib/libCryptoki2_64.so",
      "SlotId": 0,
      "Metadata": {
        "HsmPoolSize": "5",
        "PinMode": "SessionStart"
      }
    },
    "Parameters": {
      "Level": "B_T",
      "TsaUrl": "http://tzd.kamusm.gov.tr"
    }
  }'
```

**HSM ile Toplu İmzalama** — HSM, yüksek verimli toplu imzalama için oturum havuzu kullanır:

```bash
curl -X POST http://localhost:7701/api/v1/sign/batch \
  -H "Content-Type: application/json" \
  -d '{
    "Documents": [
      { "Id": "doc1", "DocumentBase64": "'$(base64 -i doc1.pdf)'" },
      { "Id": "doc2", "DocumentBase64": "'$(base64 -i doc2.pdf)'" }
    ],
    "Format": "PAdES",
    "Provider": {
      "Type": "hsm",
      "Pin": "hsm_pin",
      "Pkcs11LibraryPath": "/usr/lib/libCryptoki2_64.so",
      "Metadata": { "HsmPoolSize": "5" }
    },
    "Parameters": { "Level": "B_T", "TsaUrl": "http://tzd.kamusm.gov.tr" }
  }'
```

---

## 14. Mobil İmza

Türk mobil operatörleri **ETSI TS 102 204 MSSP** protokolü üzerinden imzalama yapar. Kullanıcı SIM kartına gelen mesajda PIN girerek onay verir. DigiMR tamamen sunucu taraflıdır — müşteri uygulamasına SDK kurulmaz.

**Desteklenen operatörler:** `Turkcell`, `Vodafone`, `TurkTelekom`

### provider Alanları (type=mobile)

| Alan | Zorunlu | Açıklama |
|------|---------|----------|
| `type` | ✅ | `"mobile"` |
| `msisdn` | ✅ | Telefon numarası — uluslararası format: `905XXXXXXXXX` |
| `mobileOperator` | ✅ | `"Turkcell"`, `"Vodafone"`, `"TurkTelekom"` |
| `displayText` | — | SIM ekranında gösterilecek metin (varsayılan: appsettings'ten) |

### Örnek — Turkcell ile PAdES B-T

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
      "displayText": "Kira sözleşmesi imzalanıyor"
    },
    "parameters": {
      "level": "B_T",
      "tsaProviderKey": "kamusm",
      "reason": "Sözleşme onayı",
      "location": "İstanbul"
    }
  }'
```

### Örnek — Vodafone ile PAdES B-B

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'$(base64 -i belge.pdf)'",
    "format": "PAdES",
    "provider": {
      "type": "mobile",
      "msisdn": "905421234567",
      "mobileOperator": "Vodafone"
    },
    "parameters": { "level": "B_B" }
  }'
```

> Mobil imza akışı asenkrondur — kullanıcı SIM'de onay verene kadar istek bekler (max 120 saniye).
> `mobileOperator` büyük-küçük harf duyarsızdır. Tam dokümantasyon: [MOBILE_SIGNATURE.md](MOBILE_SIGNATURE.md)

---

## 15. Token Oturum API

Token PIN'ini bir kez girerek `sessionId` alın. Sonraki tüm imzalama isteklerinde PIN yerine `sessionId` kullanın.

### GET /api/v1/token/slots

Sunucuya bağlı PKCS#11 slotlarını listeler (PIN gerekmez).

```bash
curl "http://localhost:7701/api/v1/token/slots?libraryPath=C:/Windows/System32/eTPKCS11.dll"
```

**Yanıt:**
```json
[
  { "slotId": 0, "label": "eToken", "manufacturerId": "SafeNet", "hasToken": true }
]
```

### GET /api/v1/token/slots/{slotId}/certificates

Sloттaki sertifikaları PIN girmeden listeler (yalnızca public).

```bash
curl "http://localhost:7701/api/v1/token/slots/0/certificates?libraryPath=C:/Windows/System32/eTPKCS11.dll"
```

### POST /api/v1/token/sessions

Oturum açar, `sessionId` döner. Sonraki imzalama isteklerinde kullanılır.

**İstek:**
```json
{
  "libraryPath": "C:/Windows/System32/eTPKCS11.dll",
  "pin": "123456",
  "slotId": 0
}
```

**Yanıt:**
```json
{
  "sessionId": "abc123",
  "expiresAt": "2026-03-07T10:05:00Z",
  "slotLabel": "eToken",
  "certificateCount": 1
}
```

### SessionId ile İmzalama

```json
{
  "documentBase64": "...",
  "format": "PAdES",
  "provider": {
    "type": "token",
    "sessionId": "abc123"
  },
  "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
}
```

### GET /api/v1/token/sessions

Aktif oturumları listeler.

### GET /api/v1/token/sessions/{sessionId}

Oturum detayı (süresi, sertifika sayısı).

### DELETE /api/v1/token/sessions/{sessionId}

Oturumu kapatır ve PIN'i bellekten temizler.

### POST /api/v1/token/pin/verify

PIN doğrular (oturum açmadan).

---

## 16. Çoklu İmza (Aynı Belgede Birden Fazla İmza)

PAdES, ISO 32000-2 uyumlu **incremental update** modu kullandığından aynı PDF'e birden fazla imza eklenebilir. Her imza önceki imzaları bozmaz — her biri kendi ByteRange'ini imzalar.

### Nasıl Çalışır?

İmzalı belgeyi bir sonraki imzaya girdi olarak verin:

```bash
# 1. İmzacı A imzalar
SIGNED_A=$(curl -s -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'"$(base64 -i belge.pdf)"'",
    "format": "PAdES",
    "provider": { "type": "software", "certificateBase64": "'"$(base64 -i imzaci_a.p12)"'", "certificatePassword": "sifre_a" },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm", "reason": "Birinci onay" }
  }' | jq -r '.signedDocumentBase64')

# 2. İmzacı B, A'nın imzası üzerine imzalar — A'nın imzası bozulmaz
SIGNED_AB=$(curl -s -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "'"$SIGNED_A"'",
    "format": "PAdES",
    "provider": { "type": "software", "certificateBase64": "'"$(base64 -i imzaci_b.p12)"'", "certificatePassword": "sifre_b" },
    "parameters": { "level": "B_T", "tsaProviderKey": "kamusm", "reason": "İkinci onay" }
  }' | jq -r '.signedDocumentBase64')

# 3. Tüm imzaları doğrula — her ikisi de geçerli olmalı
curl -X POST http://localhost:7701/api/v1/verify \
  -H "Content-Type: application/json" \
  -d '{"documentBase64": "'"$SIGNED_AB"'"}'
```

### Önemli Notlar

| Konu | Açıklama |
|------|----------|
| **Sıra** | Her adımda önceki adımın çıktı belgesi sonraki adımın girdisi olur |
| **Bağımsızlık** | Her imzacı kendi sertifikasını ve zaman damgasını kullanır |
| **Bütünlük** | Sonraki imza öncekini bozmaz (PDF incremental update / append mode) |
| **Sınır** | Teknik limit yoktur; kurumsal onay akışlarında 10+ imza desteklenir |
| **Format** | PAdES (PDF) için doğrudan desteklenir. CAdES ve XAdES için ayrı imza dosyaları oluşturulur |

---

## 17. İki Aşamalı İmzalama (Prepare/Finalize)


Hash'i dışarıda imzalamak isteyenler için — PIN sunucuya gönderilmez.

### POST /api/v1/sign/prepare

Belgeyi hazırlar ve imzalanacak hash'i döner.

**İstek:**
```json
{
  "documentBase64": "<base64 PDF>",
  "format": "PAdES",
  "certificateBase64": "<imzalayan sertifika DER base64>",
  "parameters": { "level": "B_T", "tsaProviderKey": "kamusm" }
}
```

**Yanıt:**
```json
{
  "jobId": "job-uuid-xxxx",
  "hashBase64": "<imzalanacak hash base64>",
  "expiresIn": 300
}
```

### POST /api/v1/sign/finalize

Ham imzayı (CMS/PKCS#7 raw signature) gönderir, imzalı belgeyi alır.

**İstek:**
```json
{
  "jobId": "job-uuid-xxxx",
  "rawSignatureBase64": "<ham RSA/ECDSA imza base64>"
}
```

**Yanıt:**
```json
{
  "success": true,
  "signedDocumentBase64": "<imzali belge base64>"
}
```

> `jobId` 5 dakika geçerliliğini kaybeder. Prepare sonrası 5 dakika içinde finalize çağrılmalıdır.

---

## 18. Format Keşif API

### GET /api/v1/sign/formats

Desteklenen tüm format, seviye, provider ve hash algoritmalarını listeler.

```bash
curl http://localhost:7701/api/v1/sign/formats
```

**Yanıt:**
```json
{
  "formats": ["PAdES", "CAdES", "XAdES", "ASiC-S", "ASiC-E", "EYP", "KEP"],
  "levels": ["B_B", "B_T", "B_LT", "B_LTA"],
  "providers": ["software", "token", "hsm", "remote-token", "mobile"],
  "hashAlgorithms": ["SHA256", "SHA384", "SHA512"]
}
```

---

## 19. Agent Hub (SignalR)

Uzak token agent'larını merkezi API'ye bağlamak için SignalR hub.

### POST /api/v1/agent-hub (SignalR)

Agent'lar buraya bağlanır. `AgentHubClient.cs` bağlantıyı otomatik yönetir.

### GET /api/v1/admin/agents

Bağlı agent'ları listeler.

```bash
curl http://localhost:7701/api/v1/admin/agents
```

### POST /api/v1/admin/agents/{agentId}/sign-hash

Belirli bir agent'a hash imzalatma isteği gönderir.

---

## 20. Hata Kodları ve Rate Limiting

### HTTP Durum Kodları

| Kod | Açıklama |
|---|---|
| 200 | Başarılı |
| 400 | Geçersiz istek (eksik alan, hatalı format vb.) |
| 429 | Rate limit aşıldı |
| 500 | Sunucu hatası (TSA ulaşılamıyor, token hatası vb.) |
| 503 | Servis hazır değil |

### Hata Yanıt Formatı

```json
{
  "error": "Sağlayıcı bulunamadı: 'xyz'. Mevcut sağlayıcılar: kamusm"
}
```

### Rate Limiting

Varsayılan: **100 istek/dakika/IP**. `appsettings.json`'da değiştirilebilir:

```json
"Security": {
  "RateLimitPerMinute": 100
}
```

Limit aşıldığında:

```json
{
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many requests. Please try again later."
}
```

---

## Changelog

### v3.0 (Mart 2026)
- **Token Oturum API** — `/api/v1/token/*`: slot listeleme, oturum açma (sessionId), PIN doğrulama
- **İki Aşamalı İmzalama** — `/api/v1/sign/prepare` + `/api/v1/sign/finalize`
- **Agent Hub (SignalR)** — uzak token agent merkezi yönetimi
- **Çoklu TSA Sağlayıcı** — `tsaProviderKey` ile Kamu SM, TÜRKTRUST, E-Güven seçimi
- **Doğrulama API genişletme** — `/verify/check`, `/verify/inspect`, ValidationState (PREMATURE/MATURE)
- **Tüm Türk operatörleri MSSP** — Vodafone ve Türk Telekom gerçek gateway implementasyonu (ETSI TS 102 204)
- **Mobil imza düzeltmesi** — `msisdn` + `mobileOperator` alan isimleri güncellendi
- **Format Keşif API** — `GET /api/v1/sign/formats`
- **parameters genişletme** — `tsaProviderKey`, `signerRole`, `commitmentType`, `signaturePolicy`, `productionPlace`, `contactInfo`
- **Swagger Örnekleri** — Swagger UI'da her endpoint için named request/response örnekleri

### v2.0 (Mart 2026)
- Zaman Damgası API eklendi (`/api/v1/timestamp`)
- TÜRKTRUST, E-Güven, EDM KEP, E-İmzaTR TSA desteği (yapılandırma hazır)
- KEP API eklendi
- HSM oturum havuzu desteği
- Mobil imza desteği
- Seviye yükseltme API eklendi (`/api/v1/upgrade`)
- KAMUSM entegrasyon testleri (gerçek sunucu)

### v1.0 (Şubat 2026)
- CAdES, PAdES, XAdES imza desteği
- EYP 1.3, 2.0, 2.1 desteği
- Toplu imzalama
- Provider soyutlama (Software, Token, RemoteToken, HSM)
- Kapsamlı doğrulama
- Rate limiting ve güvenlik denetimleri

---

*Copyright © 2026 Moreum Tech*
