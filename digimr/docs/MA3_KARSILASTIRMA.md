# MA3 API vs DigiMR — Karşılaştırma Raporu

> Kaynak: TÜBİTAK BİLGEM MA3 API Kullanım Kılavuzu (129 sayfa) + DigiMR kaynak kodu ve dokümantasyonu.
> Tarih: 2026-03-07

---

## 1. Genel Bakış

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| Üretici | TÜBİTAK BİLGEM | Moreum |
| Hedef kitle | Türkiye kamu/özel kurumları | Platform-agnostik kurumsal müşteriler |
| Mimari | Kütüphane (JAR / DLL) | REST API + gRPC + SignalR + SDK |
| Teknoloji | Java + .NET (C#) | .NET 8 |
| Entegrasyon yöntemi | Doğrudan kütüphane çağrısı (koda gömülü) | HTTP REST / gRPC / SignalR (dil-bağımsız) |
| Lisans | Ücretli lisans dosyası (lisans.xml, bakım sözleşmesi) | Özel / tescilli |
| Belgeleme | PDF kılavuz | Swagger/OpenAPI + Markdown docs |
| Kaynak kod | Kapalı (örnek kodlar açık) | Kapalı |

---

## 2. İmza Formatları

### 2.1 CAdES (.p7s)

| Format / Seviye | MA3 API | DigiMR |
|-----------------|---------|--------|
| BES (B-B) | ✅ | ✅ |
| EPES | ✅ | ✅ (P2-P4 profilleri) |
| EST / T (B-T) | ✅ | ✅ |
| C | ✅ | — |
| X-Long / LT (B-LT) | ✅ (ESXLong) | ✅ |
| X-Type 1 / X-Type 2 | ✅ | — |
| X-Long-Type 1/2 | ✅ | — |
| A / LTA (B-LTA) | ✅ (ESA) | ✅ |
| Paralel imza | ✅ | ✅ |
| Seri imza (counter-signature) | ✅ | ✅ |
| Ayrık imza (detached) | ✅ | ✅ |
| Bütünleşik imza (enveloping) | ✅ | ✅ |
| Standart referansı | ETSI TS 101 733 | ETSI EN 319 122 (güncel) |

> MA3 TS 101 733 (eski adlandırma: BES/EST/ESXLong/ESA), DigiMR EN 319 122 (yeni adlandırma: B-B/B-T/B-LT/B-LTA) kullanır.

### 2.2 XAdES (.xml)

| Format / Seviye | MA3 API | DigiMR |
|-----------------|---------|--------|
| BES (B-B) | ✅ | ✅ |
| EPES | ✅ | ✅ |
| T (B-T) | ✅ | ✅ |
| C | ✅ | — |
| X | ✅ | — |
| XL / LT (B-LT) | ✅ | ✅ |
| A / LTA (B-LTA) | ✅ | ✅ |
| Enveloping | ✅ | ✅ |
| Enveloped | ✅ | ✅ |
| Detached | ✅ | ✅ |
| Paralel imza | ✅ | ✅ |
| Seri imza | ✅ | ✅ |

### 2.3 PAdES (.pdf)

| Format / Seviye | MA3 API | DigiMR |
|-----------------|---------|--------|
| BES (B-B) | ✅ Java only | ✅ (C# + .NET) |
| EPES | ✅ Java only | ✅ |
| T (B-T) | ✅ Java only | ✅ |
| XL / LT (B-LT) | ✅ Java only | ✅ |
| A / LTA (B-LTA) | ✅ Java only (LTV olarak) | ✅ |
| Platform | **Sadece Java** | .NET (tüm platform) |
| Altyapı kütüphanesi | iText (ayrı lisans gerekir) | iTextSharp (dahili) |

### 2.4 ASiC

| Format | MA3 API | DigiMR |
|--------|---------|--------|
| ASiC-S (CAdES veya XAdES) | ✅ | ✅ |
| ASiC-E (CAdES veya XAdES) | ✅ | ✅ |
| Paket doğrulama | ✅ | ✅ |
| İmza geliştirme (upgrade) | ✅ | ✅ |

### 2.5 Türkiye'ye Özgü Formatlar

| Format | MA3 API | DigiMR |
|--------|---------|--------|
| EYP 2.0 (CBDDO Elektronik Yazışma Paketi) | — | ✅ |
| KEP (Kayıtlı Elektronik Posta) | — | ✅ |

---

## 3. Türk Yasal Uyumu (BTK İmza Profilleri)

| Profil | Tanım | MA3 API | DigiMR |
|--------|-------|---------|--------|
| P1 | Anlık — BES, zaman damgasız | ✅ | ✅ |
| P2 | Kısa Süreli — SİL, ES-T | ✅ | ✅ |
| P3 | Uzun Süreli — SİL, ES-XL | ✅ | ✅ |
| P4 | Uzun Süreli — OCSP, ES-XL | ✅ | ✅ |
| Policy OID gömme | ✅ | ✅ |
| 5070 sayılı Kanun uyumu | ✅ | ✅ |
| ETSI EN 319-1xx uyumu | Kısmi (TS 101 733 tabanlı) | ✅ (EN 319 122/132/132) |

---

## 4. Zaman Damgası (TSA)

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| RFC 3161 | ✅ | ✅ |
| Kamu SM desteği | ✅ (http://tzd.kamusm.gov.tr) | ✅ |
| TÜRKTRUST | Yapılandırılabilir | ✅ (config) |
| E-Güven | Yapılandırılabilir | ✅ (config) |
| Birden fazla TSA sağlayıcı | Yapılandırılabilir | ✅ (TsaProviders yapılandırması) |
| Kontör sorgulama | ✅ (TSClient.requestRemainingCredit) | ✅ (GET /api/v1/admin/tsa-credit + MA3Compat TSClient) |
| Arşiv zaman damgası | ✅ | ✅ |

---

## 5. Sertifika & İptal Doğrulama

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| CRL (SİL) doğrulama | ✅ (HTTP, LDAP, dosya) | ✅ |
| OCSP (ÇİSDUP) doğrulama | ✅ | ✅ |
| Lokal CRL deposu | ✅ (SVT formatı, dosya sistemi) | ✅ (SQLite WAL, diff-sync) |
| CRL otomatik yenileme | Manuel / politika dosyasıyla | ✅ (BackgroundService, 12h) |
| İmza tarihine göre iptal kontrolü | ✅ (P_SIGNING_TIME parametresi) | ✅ (CheckRevocationStatusAtAsync) |
| Grace period (kesinleşme süresi) | ✅ (varsayılan 24 saat, P_GRACE_PERIOD) | Yapılandırılabilir |
| Ön doğrulama (PREMATURE/MATURE) | ✅ | — |
| Doğrulama politika dosyası | ✅ XML tabanlı (policy.xml) | — (kod tabanlı) |
| Delta CRL desteği | ✅ | — |
| Çapraz sertifika | ✅ | — |
| OCSP gömme (B-LT/LTA) | ✅ | ✅ |

---

## 6. Akıllı Kart / Token Desteği

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| PKCS#11 | ✅ | ✅ |
| PKCS#12 (.pfx / .p12) | ✅ | ✅ |
| AKIS kart | ✅ (AkisCIF, APDU) | ✅ (PKCS#11 üzerinden) |
| eToken / SafeNet | Yapılandırılabilir | ✅ |
| SoftHSM2 | — | ✅ |
| HSM (Cloud/Network) | — | ✅ (adapter pattern) |
| ATR bazlı kart tipi tespiti | ✅ | — |
| APDU erişimi | ✅ (APDUSmartCard) | — |
| Çoklu kart / sertifika seçimi | ✅ (SmartCardManager) | ✅ (slot API) |
| PIN oturum yönetimi | Manuel (logout/login) | ✅ (TokenSessionManager, sessionId) |
| PIN keşleme (session token) | — | ✅ (TokenApiSessionService) |
| Uzak token agent | — | ✅ (SignalR Agent Hub) |

---

## 7. Mobil İmza

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| Mobil imza desteği | ✅ (v1.4.2+, MSSP protokolü) | ✅ (ETSI TS 102 204 MSSP) |
| İstemci API | ✅ (MobileSigner) | ✅ (MobileSigningProvider) |
| Sunucu API | ✅ (EMSSPRequestHandler) | ✅ (MsspGatewayBase SOAP) |
| Turkcell desteği | ✅ | ✅ (TurkcellGateway — test edildi) |
| Vodafone desteği | Yapılandırılabilir | ✅ (VodafoneGateway — kodlandı, operatör testi bekliyor) |
| Türk Telekom desteği | Yapılandırılabilir | ✅ (TurkTelekomGateway — kodlandı, operatör testi bekliyor) |
| Gerçek MSSP bağlantısı | ✅ | ✅ (Turkcell test edildi, Vodafone/TTNet operatör sözleşmesi gerekli) |

---

## 8. Uygulama Mimarisi

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| Entegrasyon modeli | Kütüphane (koda gömülü) | REST API / SDK / gRPC |
| Dil desteği | Java, .NET | Tüm diller (HTTP) |
| REST API | — | ✅ |
| gRPC | — | ✅ |
| SignalR (WebSocket) | — | ✅ (Agent Hub) |
| SDK | — | ✅ (DigitalSignatureSDK) |
| Swagger / OpenAPI | — | ✅ |
| Rate limiting | — | ✅ (.NET 8 RateLimiter) |
| Audit logging | — | ✅ |
| Correlation ID | — | ✅ |
| İki aşamalı imzalama (prepare/finalize) | — | ✅ (hash dışarıda imzalanabilir) |
| Batch imzalama | — | ✅ |

---

## 9. İmza Doğrulama

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| CAdES doğrulama | ✅ | ✅ |
| XAdES doğrulama | ✅ | ✅ |
| PAdES doğrulama | ✅ | ✅ |
| ASiC doğrulama | ✅ | ✅ |
| EYP doğrulama | — | ✅ |
| KEP doğrulama | — | ✅ |
| Paralel imza doğrulama | ✅ | ✅ |
| Seri imza doğrulama | ✅ | ✅ |
| Ayrık imza doğrulama | ✅ | ✅ |
| İmza seviyesi belirleme | ✅ (SignatureStatus) | ✅ (VerifyController: /check + /inspect) |
| Detaylı doğrulama sonucu | ✅ (SignedDataValidationResult) | ✅ (JSON raporu) |
| İmzacı kimlik bilgileri | ✅ (CN, TC Kimlik No) | ✅ |
| Format formatı tespiti | — | ✅ (/api/v1/verify/inspect) |

---

## 10. Yapılandırma & Operasyon

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| Yapılandırma yöntemi | XML politika dosyaları + kod parametreleri | appsettings.json |
| Politika dosyası güvenliği | ✅ (imzalı, şifreli, hash kontrollü) | — |
| Çalışma zamanı politika güncellemesi | ✅ | — |
| Proxy desteği | ✅ (XML imza için) | Sistem proxysinden otomatik |
| Log sistemi | SLF4J (Java) / log4net (.NET) | .NET ILogger |
| Docker/container desteği | — | ✅ |
| Background servisler | — | ✅ (CrlRefreshService) |

---

## 11. Android

| Özellik | MA3 API | DigiMR |
|---------|---------|--------|
| Android desteği | ✅ (BES imza, ACS kart okuyucu) | — |
| İmza türleri | Sadece BES | — |
| Desteklenen kart okuyucu | ACS (android kütüphanesi) | — |

---

## 12. Özet Karşılaştırma Tablosu

| Kategori | MA3 API | DigiMR | Kazanan |
|----------|---------|--------|---------|
| CAdES seviyeleri | B-B/T/C/X/LT/LTA + eski Type 1/2 | B-B/T/C/X/LT/LTA (güncel EN 319 122) | Berabere |
| PAdES platformu | Java only | .NET (tüm platform) | DigiMR |
| XAdES seviyeleri | BES/T/C/X/XL/A | BES/T/XL/A (güncel EN 319 132) | Berabere |
| JAdES (JSON imza) | — | ✅ (ETSI TS 119 182) | DigiMR |
| Türkiye'ye özgü (EYP, KEP, UETS) | — | ✅ | DigiMR |
| Entegrasyon kolaylığı | Kütüphane (aynı dil şart) | REST API + gRPC + SDK | DigiMR |
| Akıllı kart | PKCS#11 + ATR + APDU | PKCS#11 + HSM pool | Berabere |
| Mobil imza (MSSP) | ✅ Turkcell/Vodafone/TTNet | ✅ Turkcell (test edildi) / Vodafone, TTNet (kodlandı) | Berabere |
| İki aşamalı imza (prepare/finalize) | — | ✅ PAdES + CAdES + XAdES | DigiMR |
| Uzak token agent | — | ✅ Agent Hub + Web UI | DigiMR |
| CSC API (bulut imza) | — | ✅ (ETSI TS 119 432) | DigiMR |
| Grace period (PREMATURE/MATURE) | ✅ | ✅ | Berabere |
| CRL lokal depo | Dosya bazlı | SQLite + otomatik sync | DigiMR |
| Doğrulama politika | XML dosya | Config profil + XML okuyucu | Berabere |
| TSA doğrulama + retry | — | ✅ (imza + EKU + backoff) | DigiMR |
| Long-Term Preservation | — | ✅ (arşiv damga yenileme) | DigiMR |
| Trusted List (EU) | — | ✅ (ETSI TS 119 612 v6) | DigiMR |
| EUDI Wallet | — | ✅ (endpoint yapısı) | DigiMR |
| SHA-3 (PQC hazırlık) | — | ✅ (SHA3-256/384/512) | DigiMR |
| XML güvenlik (XXE/XSW/SSRF) | — | ✅ (SecureXmlFactory) | DigiMR |
| API dokümantasyonu | PDF kılavuz | Swagger + Markdown + Türkçe kılavuz | DigiMR |
| Çok dilli platform | Java + .NET | .NET + tüm HTTP dilleri | DigiMR |

---

## 13. Temel Farklılıklar

### MA3 API Tek Avantajı
1. **Android ACS kart okuyucu (APDU)** — Akıllı kart okuyucu ile Android cihazda imzalama. DigiMR'de MAUI Android var ama APDU kart okuyucu desteği henüz yok.

> Not: MA3'ün eski ETSI TS 101 733 ara formatları (ES-X Type 2, ES-X-Long Type 2) ve Delta CRL desteği güncel standartlarda (EN 319 122) artık gerekli değildir. DigiMR güncel standartları takip eder.

### DigiMR Avantajları
1. **REST API + gRPC + SDK mimarisi** — Herhangi bir dil/platform HTTP üzerinden kullanabilir.
2. **JAdES (JSON imza)** — eIDAS 2.0 kapsamında 4. resmi imza formatı. MA3'te yok.
3. **CSC API (bulut imza)** — ETSI TS 119 432 standart uzak imzalama protokolü. EUDI Wallet uyumlu.
4. **EYP 2.0 + KEP + UETS** — Türkiye'ye özgü paket formatları.
5. **İki aşamalı imzalama** — PAdES + CAdES + XAdES prepare/finalize.
6. **Uzak Agent + Web UI** — Plugin model, tarayıcıdan indirip kurulabilir.
7. **Long-Term Preservation** — Arşiv zaman damgası yenileme + evidence record.
8. **Trusted List v6** — AB güvenilir servis sağlayıcı listesi entegrasyonu.
9. **SHA-3 hazırlık** — Post-quantum geçiş için SHA3-256/384/512.
10. **TSA doğrulama** — Her TSA yanıtı imza + EKU + zaman kontrolü.
11. **XML güvenlik** — XXE, XSW, SSRF saldırılarına karşı korumalı.
12. **Grace period** — PREMATURE/MATURE imza olgunlaşma durumu.
13. **SQLite CRL deposu** — Otomatik 12 saatlik sync, offline çalışma.
14. **Modern altyapı** — Swagger, rate limiting, audit logging, CI/CD pipeline.

---

## 14. Kullanım Senaryolarına Göre Öneri

| Senaryo | Önerilen |
|---------|----------|
| Mevcut Java projesine imza entegrasyonu | MA3 API |
| Mevcut .NET projesine imza entegrasyonu | DigiMR SDK |
| Mikroservis / çok dilli mimari | DigiMR REST API |
| Android ACS kart okuyucu | MA3 API (tek seçenek) |
| Mobil imza (Turkcell/Vodafone) | DigiMR veya MA3 |
| EYP 2.0 / KEP / UETS paketi | DigiMR (tek seçenek) |
| JSON tabanlı imza (JAdES) | DigiMR (tek seçenek) |
| Bulut imza (CSC API) | DigiMR (tek seçenek) |
| Hash-only imzalama (HSM) | DigiMR (prepare/finalize) |
| Uzak token yönetimi | DigiMR Agent |
| Uzun süreli arşivleme | DigiMR (Preservation) |
| AB pazarı (eIDAS 2.0) | DigiMR (Trusted List + EUDI) |
| Kamu SM entegrasyonu | Her ikisi de |
| BTK P1-P4 profil uyumu | Her ikisi de |

---

## 15. Teknik Notlar

- **Standart referansı**: MA3 eski ETSI TS 101 733 kullanır. DigiMR güncel ETSI EN 319 122/132/142/162/182 standartlarını takip eder.
- **PAdES lisansı**: MA3 iText ticari lisansı gerektirir. DigiMR iTextSharp (LGPL v2) kullanır — ek lisans gerekmez.
- **PIN yönetimi**: MA3'te her imzada PIN girilir. DigiMR'de PIN bir kez girilir, sessionId ile devam edilir.
- **CRL deposu**: MA3'te manuel yapılandırma. DigiMR'de sertifika CDP'sinden otomatik keşif + SQLite depolama.
- **MA3 uyumluluk**: DigiMR `DigitalSignature.MA3Compat` katmanı ile MA3 API sınıf/metot isimlerini destekler (108/108 test geçiyor).
