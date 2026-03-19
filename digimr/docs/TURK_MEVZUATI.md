# DigiMR - Türkiye Elektronik İmza Uyum Analizi

## Yasal Dayanak

| Mevzuat | Kapsam | Durum |
|---------|--------|-------|
| **5070 sayılı Elektronik İmza Kanunu** (OG: 25355, 23.01.2004) | Temel kanun, güvenli e-imza = el yazısı | ✅ Uyumlu |
| **2021 Değişikliği** (Elektronik Mühür eklendi) | Tüzel kişi mührü | ✅ Desteklenir |
| **BTK Uygulama Yönetmeliği** (06.01.2005) | Sertifika profilleri, ESHS yükümlülükleri | ✅ Uyumlu |
| **BTK Teknik Kriterler Tebliği** (2006/DK-77/353) | İmza formatları, hash algoritmaları | ✅ Uyumlu |
| **BTK Kurul Kararı 2012/DK-15/299** | P1-P4 profilleri rehberi | ✅ **Desteklendi** |
| **KEP Yönetmeliği** (OG: 28036, 25.08.2011) | 7201 Kanun, KEP servisi | ✅ Desteklenir |

---

## İmza Formatları

### Temel ETSI Formatları

| Format | Standart | Seviyeler | Durum |
|--------|----------|-----------|-------|
| **CAdES** | ETSI EN 319 122-1 | B-B, B-T, B-C, B-X, B-LT, B-LTA | ✅ Tam |
| **XAdES** | ETSI EN 319 132-1 | BES, T, XL, A | ✅ Tam |
| **PAdES** | ETSI EN 319 142-1 | B-B, B-T, B-LT, B-LTA | ✅ Tam |
| **JAdES** | ETSI TS 119 182-1 | B-B, B-T | ✅ Desteklenir |
| **ASiC-S** | ETSI EN 319 162-1 | CAdES icerigi | ✅ Tam |
| **ASiC-E** | ETSI EN 319 162-1 | Coklu belge | ✅ Tam |

### PAdES ETSI Uyumu

| Gereksinim | Durum |
|-----------|-------|
| SubFilter: `ETSI.CAdES.detached` | ✅ |
| `signing-certificate-v2` (ESSCertIDv2) | ✅ |
| `signing-time` | ✅ |
| `sig-policy-id` (EPES, P2/P3/P4 icin) | ✅ |

> **Not**: DSS Demo'da beklenen sonuc `PAdES-BASELINE-B` veya `PAdES-BASELINE-T`.

---

## Türkiye P1-P4 İmza Profilleri

**Kaynak**: BTK Kurul Kararı 2012/DK-15/299 — "Elektronik İmza Kullanım Profilleri Rehberi" (Kamu SM / TÜBİTAK BİLGEM)

### Profil Karşılaştırma Tablosu

| Profil | ETSI Formatı | Seviye | Policy OID | Revokasyon | Bekleme Süresi |
|--------|-------------|--------|-----------|-----------|----------------|
| **P1** | CAdES/XAdES/PAdES-BES | B-B | Yok (BES) | CRL veya OCSP | - |
| **P2** | CAdES/XAdES/PAdES-T | B-T | `2.16.792.1.61.0.1.5070.3.1.1` | CRL zorunlu | 24 saat |
| **P3** | CAdES/XAdES/PAdES-XL | B-LT | `2.16.792.1.61.0.1.5070.3.2.1` | CRL + gömülü | 24 saat |
| **P4** | CAdES/XAdES/PAdES-XL | B-LT | `2.16.792.1.61.0.1.5070.3.3.1` | OCSP + gömülü | Yok |

### P2/P3/P4 İçin Zorunlu İmza Bilgileri

P2, P3 ve P4 profilleri ETSI standartlarına uygun olarak şu bilgileri imzaya otomatik ekler:
- **İmza zamanı** (signing-time)
- **İmzalayan sertifika referansı** (signing-certificate-v2, ESSCertIDv2)
- **İmza politikası** (sig-policy-id, EPES) — profil OID'si ile

DigiMR bu attribute'ları `TurkishSignatureProfiles.CreateP2/P3/P4Parameters()` ile otomatik yönetir.

### Kullanım

```csharp
// P1 - Anlık doğrulama (politika yok)
var p1 = TurkishSignatureProfiles.CreateP1Parameters(SignatureFormat.PAdES);

// P2 - Kısa vadeli CRL (policy hash Kamu SM'den alınmalı)
var p2 = TurkishSignatureProfiles.CreateP2Parameters(
    policyDigestValue: kamusmP2PolicyHash,
    tsaUrl: TurkishSignatureProfiles.KamusmTsaUrl);

// P3 - Uzun vadeli CRL
var p3 = TurkishSignatureProfiles.CreateP3Parameters(policyDigestValue: kamusmP3PolicyHash);

// P4 - Uzun vadeli OCSP (anlık geçerli)
var p4 = TurkishSignatureProfiles.CreateP4Parameters(policyDigestValue: kamusmP4PolicyHash);
```

> **Önemli**: Policy hash değerleri (`policyDigestValue`) Kamu SM'nin yayımladığı resmi politika belgelerinden (PDF) SHA-256 hash alınarak hesaplanmalıdır. Bu hash'ler imzaya gömülür ve doğrulama sırasında politika belgesinin değişmediğini kanıtlar.

---

## Türkiye'ye Özgü Formatlar

### EYP — Elektronik Yazışma Paketi

| Versiyon | Standart | İmza | Mühür | Hash | Durum |
|----------|---------|------|-------|------|-------|
| EYP 1.3 | OPC (ECMA-376) | CAdES veya XAdES | Opsiyonel | SHA-256 | ✅ Desteklenir |
| EYP 2.0 | OPC (ECMA-376) | CAdES-B-LT (P3/P4) | Zorunlu CAdES-B-LTA | SHA-384 + SHA-512 | ✅ Desteklenir |
| EYP 2.1 | OPC (ECMA-376) | P3 profili desteği ek | Zorunlu | SHA-384 + SHA-512 | ✅ Desteklenir |

**Yönetim**: Cumhurbaşkanlığı Dijital Dönüşüm Ofisi (CBDDO)
**Uyum Değerlendirmesi**: TÜBİTAK BİLGEM Kamu SM ("EYP Uyum Değerlendirmesi")

### KEP — Kayıtlı Elektronik Posta

| Özellik | Standart | Durum |
|---------|---------|-------|
| Temel standart | ETSI TS 102 640 (BTK: 7201 Kanun) | ✅ Desteklenir |
| Paket yapısı | ZIP + META-INF/kep-bilgi.xml + imza.p7s | ✅ Desteklenir |
| İmza formatı | CAdES-T detached | ✅ Desteklenir |
| Teslim türleri | Gonderi, Kabul, Alindi, Icerik | ✅ Desteklenir |

**Lisanslı KEP Sağlayıcıları**: PTT A.Ş., TÜRKKEP, TNB KEP, EDM KEP

### e-Fatura / GİB Belgeleri (e-Arşiv, e-İrsaliye vb.)

| Özellik | Standart | Durum |
|---------|---------|-------|
| Format | UBL-TR 1.2 (OASIS UBL 2.1 adaptasyonu) | ⚠️ Kısmi |
| İmza formatı | XAdES enveloped | ✅ Desteklenir |
| Sertifika | Mali Mühür MÜS (Kamu SM exclusive) | ✅ Desteklenir |
| UBL-TR XML şablon | GİB'e özgü alanlar | ❌ Yok (harici bileşen gerekir) |

> **Not**: UBL-TR XML şablonu GİB'e özgüdür. Uygulamamız XAdES imzalama altyapısını sağlar; UBL-TR XML üretimi için GİB entegrasyon yazılımı (ör. EDEP, Logo, SAP modülü) veya özel geliştirme gerekir.

### Elektronik Mühür (e-Mühür)

| Özellik | Açıklama | Durum |
|---------|---------|-------|
| Yasal dayanak | 5070 Kanunu 2021 değişikliği | ✅ |
| Uygulama | CAdES/XAdES/PAdES, mühür sertifikasıyla | ✅ Desteklenir |
| EYP mühür | EypSealMode (WithSeal/WithoutSeal) | ✅ Desteklenir |
| Mali Mühür | GİB e-belgeler için Kamu SM exclusive | ✅ (XAdES ile) |

---

## Zaman Damgası (RFC 3161)

| Özellik | Durum |
|---------|-------|
| RFC 3161 uyumu | ✅ |
| KAMUSM TSA (üretim): `http://zd.kamusm.gov.tr` | ✅ |
| KAMUSM TSA (test): `http://tzd.kamusm.gov.tr` | ✅ |
| Özel kimlik doğrulama (PBKDF2/HMAC-SHA256) | ✅ |
| Çoklu TSA sağlayıcı desteği | ✅ |
| CMS unsigned attribute (signature-timestamp) | ✅ |
| Document timestamp (B-LTA, ETSI.RFC3161) | ✅ |

---

## Sertifika Türleri ve CA'lar

| Sertifika Türü | Kullanım | CA |
|----------------|---------|-----|
| NES (Nitelikli Elektronik Sertifika) | Güvenli e-imza | E-Güven, TÜRKTRUST, E-TUĞRA, Kamu SM, vb. |
| Mobil NES | SIM kart tabanlı mobil imza | Kamu SM + GSM operatörleri |
| Mali Mühür (MÜS) | e-Fatura, e-Arşiv, GİB belgeleri | **Sadece Kamu SM** |
| Kurumsal e-Mühür | EYP, genel kurumsal mühür | TÜRKTRUST, Kamu SM, diğer ESHS |

**BTK Onaylı CA'lar**: E-Güven, Kamu SM, TÜRKTRUST, E-TUĞRA, EGMSM, EİMZATR, Ayyıldız İmza, ARKİMZA

> **Adobe AATL Durumu**: Hiçbir Türk CA Adobe'nun AATL (Adobe Approved Trust List) listesinde değildir. Türkiye AB üyesi olmadığından Türk CA'lar EUTL'de de yer almaz. Bu nedenle Adobe DSS'de Türk CA'larla imzalanmış belgeler `INDETERMINATE/NO_CERTIFICATE_CHAIN_FOUND` gösterir — bu imzanın geçersizliği değil, doğrulama güven listesi eksikliğidir. **5070 sayılı kanun kapsamında imza hukuken tam geçerlidir.**

---

## DSS Demo Doğrulama Sonucu Yorumu

Mevcut kurulumda (`ETSI.CAdES.detached` SubFilter, ETSI zorunlu attribute'larla):

| Bileşen | Beklenen Sonuç | Açıklama |
|---------|---------------|----------|
| Signature format | `PAdES-BASELINE-B` | ✅ Artık PDF-NOT-ETSI değil |
| Qualification | `INDETERMINATE` | E-Güven EUTL'de yok |
| Sub indication | `NO_CERTIFICATE_CHAIN_FOUND` | Türk CA AATL/EUTL'de değil |
| Timestamp | `INDETERMINATE` | Test TSA kullanıldı |
| Kriptografik doğruluk | ✅ | İmza matematiksel olarak doğru |

Üretim ortamında (gerçek Kamu SM TSA, ETSI politika dokümanı):
- Timestamp: `PASSED` (Kamu SM TSA EUTL'de olabilir veya politika kabul edebilir)
- Signature: `INDETERMINATE` (E-Güven EUTL'de olmaya devam eder)

---

## Kesinleşme Süresi (Grace Period) Desteği

DigiMR, imza doğrulamada **PREMATURE/MATURE** kesinleşme süresi kontrolünü destekler.

| Durum | Açıklama |
|-------|----------|
| **PREMATURE** | İmza zamanından bu yana kesinleşme süresi henüz dolmamış. Sertifika iptal bilgisi henüz yayılmamış olabilir. |
| **MATURE** | Kesinleşme süresi dolmuş, sertifika durumu kesinleşmiş. |

P2 ve P3 profilleri **24 saat** kesinleşme süresi gerektirir (CRL yayılma süresi).
P4 profili OCSP kullandığından kesinleşme süresi gerekmez.

### API Kullanımı
```bash
curl -X POST http://localhost:7701/api/v1/verify/inspect \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "<base64>",
    "gracePeriodSeconds": 86400
  }'
# Yanıtta: "validationState": "PREMATURE" veya "MATURE"
```

---

## Uzun Vadeli Koruma (Long-Term Preservation)

ETSI TS 119 511/512 uyumlu arşiv zaman damgası yönetimi.

| Endpoint | Açıklama |
|----------|----------|
| `POST /api/v1/preservation/check` | Arşiv zaman damgası durumunu kontrol eder |
| `POST /api/v1/preservation/renew` | Arşiv zaman damgasını yenileyerek uzun vadeli geçerliliği uzatır |
| `POST /api/v1/preservation/evidence` | Evidence Record (zaman damgası zinciri) bilgisini çıkarır |

Kontrol sonucu hash algoritmasının zayıflığını (`IsHashAlgorithmWeak`), yenileme gerekliliğini ve mevcut arşiv zaman damgası sayısını raporlar.

---

## CSC API (Cloud Signature Consortium)

ETSI TS 119 432 uyumlu CSC API v2 desteği. Bulut tabanlı uzaktan imzalama senaryoları için standart arayüz.

| Endpoint | Açıklama |
|----------|----------|
| `POST /api/v1/csc/info` | Servis bilgisi |
| `POST /api/v1/csc/credentials/list` | Mevcut credential'lar |
| `POST /api/v1/csc/credentials/info` | Credential detayları |
| `POST /api/v1/csc/credentials/authorize` | PIN ile yetkilendirme (SAD token döner) |
| `POST /api/v1/csc/signatures/signHash` | Hash imzala |

---

## JAdES Uyum Durumu

JAdES (ETSI TS 119 182-1) JSON tabanlı imza formatı olarak desteklenmektedir.

| Özellik | Durum |
|---------|-------|
| B-B seviyesi | ✅ Desteklenir |
| B-T seviyesi | ✅ Desteklenir |
| JWS Compact Serialization | ✅ |
| JSON Serialization | ✅ |
| Türk P1-P4 profilleri | P1 (B-B) ve P2 (B-T) uyumlu |

> **Not**: JAdES özellikle REST API tabanlı sistemler ve mikroservis mimarileri için uygundur.

---

## Önerilen Geliştirmeler (Gelecek)

| Konu | Öncelik | Açıklama |
|------|---------|----------|
| P2/P3/P4 policy hash'lerinin Kamu SM'den alınması | Yüksek | Resmi politika belgeleri indirilip SHA-256 hash'leri hesaplanmalı |
| UBL-TR XML şablonu | Orta | e-Fatura/e-Arşiv için GİB UBL-TR format üretimi |
| ETSI EN 319 532 (yeni KEP standardı) | Düşük | KEP güncellemesi (ETSI TS 102 640'ın yerini alıyor) |
| TÜBİTAK BİLGEM e-imza uyum değerlendirmesi | Yüksek | Kamu kurumlarına satış için zorunlu |

---

*Kaynak: BTK "Elektronik İmza Kullanım Profilleri Rehberi" (2012/DK-15/299), ETSI EN 319 serisi, 5070 sayılı kanun ve değişiklikleri.*
