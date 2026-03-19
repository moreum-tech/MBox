# DigiMR Dokümantasyon

## Hızlı Başlangıç

| Hedef | Doküman |
|-------|---------|
| **Kullanmayı öğren** | [KULLANIM_KILAVUZU.md](KULLANIM_KILAVUZU.md) |
| **Örnek kodlar** | [ORNEKLER.md](ORNEKLER.md) |
| **Ne zaman ne kullan** | [SENARYO_REHBERI.md](SENARYO_REHBERI.md) |

## Hedef Kitleye Göre

### Geliştirici (SDK / API Entegrasyonu)
1. [KULLANIM_KILAVUZU.md](KULLANIM_KILAVUZU.md) — SDK metotları + REST API endpoint'leri
2. [ORNEKLER.md](ORNEKLER.md) — 12 senaryo (C# + curl)
3. [SDK_REFERANS.md](SDK_REFERANS.md) — SDK tam API referansı (interface, model, enum)
4. [API_REFERANS.md](API_REFERANS.md) — REST API tam referansı (endpoint, schema, hata kodları)
5. [SAGLAYICILAR.md](SAGLAYICILAR.md) — 8 imza sağlayıcı tipi (Token, HSM, CloudHSM, Mobile, ESeal...)

### Operasyon / DevOps
1. [KURULUM_REHBERI.md](KURULUM_REHBERI.md) — Docker, Kubernetes, Windows/Linux sunucu kurulumu
2. [AGENT_KURULUM.md](AGENT_KURULUM.md) — Agent kurulumu (masaüstü, tray, port, SSL)

### Entegrasyon (Özel Konular)
1. [KAMUSM_TSA_ENTEGRASYONU.md](KAMUSM_TSA_ENTEGRASYONU.md) — Kamu SM zaman damgası entegrasyonu
2. [KEP_ENTEGRASYONU.md](KEP_ENTEGRASYONU.md) — Kayıtlı Elektronik Posta entegrasyonu
3. [MOBIL_IMZA.md](MOBIL_IMZA.md) — Mobil imza (Turkcell/Vodafone/TTNet MSSP)
4. [IMZA_FORMATLARI.md](IMZA_FORMATLARI.md) — İmza formatları detayı (CAdES/PAdES/XAdES/JAdES/ASiC)

### Uyum / Hukuk
1. [TURK_MEVZUATI.md](TURK_MEVZUATI.md) — Türk mevzuatı (5070, BTK, P1-P4 profilleri)
2. [MA3_KARSILASTIRMA.md](MA3_KARSILASTIRMA.md) — MA3 API ile karşılaştırma ve göç rehberi

### Test / Kalite
1. [DOGRULAMA_ARACLARI.md](DOGRULAMA_ARACLARI.md) — DSS Demo ve Türk doğrulayıcıları

## Desteklenen Formatlar

| Format | Standart | Seviyeler |
|--------|----------|-----------|
| **PAdES** | ETSI EN 319 142 | B-B, B-T, B-LT, B-LTA |
| **CAdES** | ETSI EN 319 122 | B-B, B-T, B-C, B-X, B-LT, B-LTA |
| **XAdES** | ETSI EN 319 132 | BES, T, XL, A |
| **JAdES** | ETSI TS 119 182 | B-B, B-T |
| **ASiC** | ETSI EN 319 162 | ASiC-S, ASiC-E |
| **EYP** | CBDDO 2.0/2.1 | OPC + CAdES |
| **KEP** | 7201 sayılı Kanun | CAdES-T + ZIP |
| **UETS** | PTT UETS | CAdES-T + ZIP |
