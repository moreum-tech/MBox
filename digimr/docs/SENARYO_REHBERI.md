# DigiMR Senaryo Rehberi — Hangi Durumda Ne Yapilir

Bu rehber, karar agaci formatinda hangi durumda hangi arac/endpoint/seviyeyi kullanmaniz gerektigini gosterir.

---

## "PDF belgesini imzalamam lazim"

```
PDF imzalama gerekiyor
│
├── Token nerede?
│   ├── Sunucuda (lokal token) ──────────────────► SDK veya REST API
│   │   ├── .NET projesi mi? ────────────────────► SDK: SignPdfWithProviderAsync
│   │   └── Baska dil (Python/Java/JS) mi? ─────► REST API: POST /api/v1/sign
│   │
│   ├── Kullanicinin bilgisayarinda ─────────────► Agent + imza-app Web UI
│   │   ├── Agent kurulu mu? ────────────────────► http://localhost:5555
│   │   └── Agent yok mu? ──────────────────────► Agent Lite indir (2.9 MB)
│   │
│   ├── Baska bir makinede ──────────────────────► Remote Token (Agent Hub)
│   │   └── provider.type = "remote-token"
│   │   └── veya SignalR Agent Hub baglantisi
│   │
│   ├── Yazilim sertifikasi (.pfx) ─────────────► provider.type = "software"
│   │   (UYARI: Ozel anahtar sunucuya gonderilir, uretim icin onerilmez)
│   │
│   └── Cep telefonunda (SIM kart) ─────────────► provider.type = "mobile"
│       └── Operator: Turkcell / Vodafone / TurkTelekom
│
├── Hangi seviye?
│   ├── Anlik islem, arsiv gerekmez ─────────────► B-B (TSA gerekmez)
│   ├── Zaman damgasi isteniyor ─────────────────► B-T (TSA gerekli)
│   ├── Uzun donemli dogrulanabilir olmali ──────► B-LT (CRL/OCSP gomulu)
│   └── Arsivleme, uzun sureli saklama ─────────► B-LTA (arsiv zaman damgasi)
│
└── Gorunur imza gerekli mi?
    ├── Evet ──► parameters.visualSignature ekle
    │   ├── Metin → mode: "TextOnly"
    │   ├── Resim → mode: "ImageOnly"
    │   ├── Muhur/damga → mode: "StampOnly"
    │   └── Metin+resim → mode: "TextAndImage"
    └── Hayir ──► Gorunmez imza (varsayilan)
```

---

## "Imzali belgeyi dogrulamam lazim"

```
Dogrulama gerekiyor
│
├── Format biliniyor mu?
│   ├── Evet, belirli format/seviye kontrolu ───► POST /api/v1/verify/check
│   │   Body: { documentBase64, format: "PAdES", level: "B_T" }
│   │   Yanit: { hasMatch: true/false, deficiencies: [...] }
│   │
│   └── Hayir, ne oldugu bilinmiyor ────────────► POST /api/v1/verify/inspect
│       Body: { documentBase64 }
│       Yanit: { detectedFormat, validSignatureTypes, signatureInventory }
│
├── Tam dogrulama sonucu gerekli mi?
│   └── Evet ───────────────────────────────────► POST /api/v1/verify
│       Yanit: { isValid, format, signatures: [...], errors, warnings }
│
├── EYP paketi mi?
│   ├── Dogrulama ──────────────────────────────► POST /api/v1/verify/eyp
│   │   veya POST /api/v1/eyp/verify
│   └── Icerik cikarma ─────────────────────────► POST /api/v1/eyp/extract
│
├── KEP paketi mi?
│   ├── Dogrulama ──────────────────────────────► POST /api/v1/verify/kep
│   │   veya POST /api/v1/kep/verify
│   └── Icerik cikarma ─────────────────────────► POST /api/v1/kep/extract
│
├── ASiC konteyneri mi?
│   ├── Dogrulama ──────────────────────────────► POST /api/v1/asic/verify
│   └── Icerik cikarma ─────────────────────────► POST /api/v1/asic/extract
│
├── JAdES (JSON) mi?
│   └── Dogrulama ──────────────────────────────► POST /api/v1/jades/verify
│
└── SDK kullaniyorsam?
    └── sdk.ValidateDocumentAsync(data)
        Format otomatik algilanir (magic bytes ile)
```

---

## "EYP paketi hazirlamam lazim"

```
EYP paketi gerekiyor
│
├── Hangi versiyon?
│   ├── Yeni proje / kamu yazismasi ────────────► EYP 2.0 veya 2.1
│   │   └── SDK: CreateEypPackageV2Async
│   │   └── API: POST /api/v1/eyp/create
│   └── Eski sistemle uyumluluk ────────────────► EYP 1.3
│       └── SDK: CreateEypAsync (legacy)
│
├── e-Muhur gerekli mi?
│   ├── Evet (zorunlu: EYP 2.0/2.1) ───────────► sealProvider parametresi
│   │   └── HSM veya e-Seal sertifikasi kullan
│   │   └── MuhurLevel: B_LTA (zorunlu)
│   └── Hayir (opsiyonel: EYP 1.3) ────────────► sealProvider = null
│
├── Imza seviyesi?
│   ├── EYP 2.0/2.1 standart ──────────────────► ImzaLevel: B_LT, MuhurLevel: B_LTA
│   └── Ozel gereksinim ────────────────────────► B_T de kabul edilebilir (P2 profili)
│
├── Guvenlik kodu?
│   ├── Genel erisim ──────────────────────────► GuvenlikKodu: "YOK"
│   ├── Hizmete ozel ──────────────────────────► GuvenlikKodu: "HSY"
│   ├── Gizli ─────────────────────────────────► GuvenlikKodu: "GZL"
│   └── Cok gizli ─────────────────────────────► GuvenlikKodu: "CEK"
│
└── Paket yapisi (EYP 2.0):
    EYP-Paketi.eyp
    ├── Ustveri/Ustveri.xml         ← Metadata
    ├── UstYazi/ustyazi.pdf         ← Ana belge
    ├── Ekler/ek-1.pdf              ← Ekler
    ├── PaketOzeti/PaketOzeti.xml   ← Hash'ler (SHA-384 + SHA-512)
    ├── Imzalar/ImzaCades.imz       ← CAdES detached imza
    ├── NihaiUstveri/NihaiUstveri.xml
    ├── NihaiOzet/NihaiOzet.xml
    └── Muhur/MuhurCades.imz        ← CAdES-A muhur
```

---

## "Mevcut imzayi guclendirmem lazim"

```
Seviye yukseltme
│
├── B-B → B-T
│   └── Gerekli: TSA URL
│   └── Ne eklenir: Zaman damgasi (signature-time-stamp)
│   └── POST /api/v1/upgrade { targetLevel: "B_T", tsaUrl: "..." }
│   └── SDK: UpgradeSignatureAsync(data, B_T, tsaUrl)
│
├── B-T → B-LT
│   └── Gerekli: TSA + CRL/OCSP
│   └── Ne eklenir: Revokasyon bilgisi (CRL + OCSP response gomulur)
│   └── POST /api/v1/upgrade { targetLevel: "B_LT" }
│
├── B-LT → B-LTA
│   └── Gerekli: TSA
│   └── Ne eklenir: Arsiv zaman damgasi (archive-time-stamp-v3)
│   └── POST /api/v1/upgrade { targetLevel: "B_LTA" }
│
├── JAdES B-B → B-T
│   └── POST /api/v1/jades/upgrade { targetLevel: "B_T" }
│
└── Atlama: B-B → B-LTA
    └── Tek adimda: targetLevel: "B_LTA" (ara seviyeler otomatik)
```

---

## "Uzun sureli saklayacagim"

```
Arsiv senaryosu
│
├── Imzalama seviyesi
│   └── B-LTA ile imzala (en yuksek seviye)
│       └── CRL/OCSP + arsiv zaman damgasi gomulu
│       └── Belge, sertifika suresi dolduktan sonra bile dogrulanabilir
│
├── Periyodik yenileme (her 5 yilda bir)
│   ├── Durum kontrolu ──────────────────────► POST /api/v1/preservation/check
│   │   Yanit: { needsRenewal: true, recommendation: "..." }
│   │
│   ├── Yenileme ────────────────────────────► POST /api/v1/preservation/renew
│   │   Yanit: { renewedDocumentBase64: "..." }
│   │
│   └── Kanit zinciri izleme ────────────────► POST /api/v1/preservation/evidence
│       Yanit: { chainLength, timestamps: [...] }
│
├── Hash algoritmasi zayifladi mi?
│   └── check yaniti: isHashAlgorithmWeak: true
│   └── Daha guclu algoritma ile yeniden damgala
│
└── Zamanlama
    ├── 0-5 yil: Yenileme gerekmez
    ├── 5 yil: Ilk arsiv zaman damgasi yenileme
    ├── 10 yil: Ikinci yenileme
    └── ... (belge saklama suresi boyunca devam)
```

---

## "Bulut imza kullanacagim"

```
Bulut imza senaryosu
│
├── CSC API (Cloud Signature Consortium)
│   └── ETSI TS 119 432 uyumlu
│   └── Akis:
│       1. POST /api/v1/csc/info                    ← Servis bilgisi
│       2. POST /api/v1/csc/credentials/list         ← Credential listele
│       3. POST /api/v1/csc/credentials/info         ← Detay
│       4. POST /api/v1/csc/credentials/authorize    ← PIN → SAD token
│       5. POST /api/v1/csc/signatures/signHash      ← Hash imzala
│
├── EUDI Wallet (AB Dijital Kimlik Cuzdani)
│   └── Prepare/Finalize pattern
│   └── Akis:
│       1. POST /api/v1/eudi/request-signing   ← Imzalama talebi
│       2. Wallet hash'i imzalar
│       3. POST /api/v1/eudi/callback          ← Imzali hash gonder
│       4. GET  /api/v1/eudi/status/{id}       ← Durum kontrol
│
└── Iki asamali imzalama (genel)
    └── Herhangi bir harici imzalama sistemi icin
    └── POST /api/v1/sign/prepare  → hashBase64
    └── [harici imzalama]
    └── POST /api/v1/sign/finalize → signedDocumentBase64
```

---

## "Entegrasyon yapacagim"

```
Entegrasyon senaryosu
│
├── .NET projesi
│   └── SDK NuGet paketi
│   └── dotnet add package DigiMR.SDK
│   └── var sdk = new DigitalSignatureSDK();
│   └── In-process, en yuksek performans
│
├── Java / Python / Node.js / Go / diger diller
│   └── REST API
│   └── Base URL: http://localhost:7701/api/v1
│   └── Swagger UI: http://localhost:7701/swagger
│   └── JSON in/out, base64 encoded belgeler
│
├── Browser tabanli uygulama
│   └── Agent + imza-app
│   └── Agent: http://localhost:5555 (lokal token erisimi)
│   └── imza-app: Web UI, postMessage ile iletisim
│   └── Token algilama: fetch('http://localhost:5555/api/agent/status')
│
├── iframe icinden imza
│   └── postMessage API
│   └── iframe.contentWindow.postMessage({ action: 'sign', ... }, '*')
│   └── window.addEventListener('message', handler)
│
├── Mikro servis mimarisi
│   └── API sunucusunu ayri bir container'da calistir
│   └── docker run -p 7701:7701 digimr
│   └── Health check: GET /api/v1/health/ready
│
├── Toplu islem / batch
│   └── POST /api/v1/sign/batch (maks 100 belge, 500 MB)
│   └── Tek PIN/oturum ile tum belgeler
│
└── Uzun sureli entegrasyon
    └── Token oturum API ile session yonetimi
    └── POST /api/v1/token/sessions → sessionId
    └── Sonraki tum isteklerde sessionId kullan
    └── Oturum idle timeout: 5 dakika (yapilandiriilabilir)
```

---

## Format Secimi Tablosu

| Senaryo | Onerilen Format | Seviye | Neden |
|---------|----------------|--------|-------|
| PDF sozlesme imzasi | PAdES | B-T | Zaman damgali, PDF icinde gomulu |
| e-Fatura / e-Arsiv | XAdES | B-T | GIB UBL-TR uyumu |
| Genel dosya imzasi | CAdES | B-T | Her dosya tipi desteklenir |
| JSON API imzasi | JAdES | B-B/B-T | REST API entegrasyonu |
| Kamu yazismasi | EYP 2.0 | B-LT + B-LTA | CBDDO standardi |
| Kayitli posta | KEP | B-T | 7201 Kanun zorunlulugu |
| Coklu belge paketi | ASiC-E | B-T | ETSI konteyner standardi |
| Tek belge arsivi | ASiC-S | B-T | Basit konteyner |
| Arsiv belgesi | PAdES/CAdES | B-LTA | Uzun vadeli saklama |
| Anlik onay | PAdES | B-B | TSA gereksiz, hizli |

---

## Turk Profili Secimi

| Gereksinim | Profil | Aciklama |
|-----------|--------|----------|
| Anlik dogrulama yeterli | P1 (B-B) | Politika yok, en basit |
| CRL ile kisa vadeli dogrulama | P2 (B-T) | 24 saat bekleme, CRL zorunlu |
| Uzun vadeli CRL ile dogrulama | P3 (B-LT) | CRL gomulu, uzun donemli |
| Anlik OCSP ile dogrulama | P4 (B-LT) | OCSP gomulu, bekleme yok |
| EYP 2.0/2.1 standardi | P3/P4 (B-LT) | CBDDO zorunlulugu |

---

## Sorun Giderme — Hizli Referans

| Sorun | Cozum |
|-------|-------|
| DSS'de INDETERMINATE | Normal — Turk CA'lar EUTL'de degil |
| TSA baglanti hatasi | TSA URL ve kimlik bilgilerini kontrol et |
| Token bulunamadi | PKCS#11 kutuphane yolu ve slot kontrolu |
| PIN hatasi | Kalan deneme hakki var mi? (3 yanlis → kilitlenir) |
| Belge boyutu limiti | Security:MaxDocumentSizeBytes (varsayilan 100 MB) |
| Rate limit | Security:RateLimitPerMinute (varsayilan 100/dk) |
| Oturum suresi doldu | Token session idle timeout: 5 dakika |
| prepare/finalize timeout | jobId 5 dakika gecerli |

---

*Ornek kodlar: [ORNEKLER.md](ORNEKLER.md)*
*Kullanim kilavuzu: [KULLANIM_KILAVUZU.md](KULLANIM_KILAVUZU.md)*
*API referansi: [API_DOCUMENTATION.md](API_DOCUMENTATION.md)*

---

*Copyright 2026 Moreum Tech*
