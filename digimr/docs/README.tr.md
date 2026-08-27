[English](README.md)

# DigiMR Dokümantasyonu

DigiMR'ın tüm ürün dokümantasyonu — elektronik imza motoru, REST/gRPC API ve SDK'lar.

> **Okumak için hiçbir şey kurmanıza gerek yok.** Aşağıdaki belgeler, her sürümle
> birlikte `digimr-docs.tar.gz` içinde dağıtılan metinlerin aynısıdır. Tek bir ikili
> dosya indirmeden DigiMR'ın ne yaptığını — API yüzeyi, imza formatları, sağlayıcılar,
> dağıtım topolojileri — inceleyebilirsiniz.

**Dil notu:** Türkçe belgeler asıl kaynaktır ve eksiksizdir. İngilizce içerik şu an
genel bakış ve sürüm geçmişi ile sınırlıdır; her İngilizce sayfa Türkçe karşılığına bağlanır.

---

## Buradan başlayın

| Amacınız | Belge |
|---|---|
| DigiMR nedir, neyi imzalar | [tr/GENEL_BAKIS.md](tr/GENEL_BAKIS.md) |
| Kullanım senaryonuza uyuyor mu | [tr/SENARYO_REHBERI.md](tr/SENARYO_REHBERI.md) — karar rehberi |
| Kurmadan önce gerçek kod görmek | [tr/ORNEKLER.md](tr/ORNEKLER.md) — 17 uçtan uca senaryo (C# + curl) |
| Bir uç noktayı aramak | [tr/API_REFERANS.md](tr/API_REFERANS.md) — REST API referansı |
| Bir SDK tipini/metodunu aramak | [tr/SDK_REFERANS.md](tr/SDK_REFERANS.md) — .NET SDK referansı |
| Kurmak ve çalıştırmak | [tr/KURULUM_REHBERI.md](tr/KURULUM_REHBERI.md) — dağıtım kılavuzu |

## Kurulum ve işletim

| Belge | İçerik |
|---|---|
| [tr/KURULUM_REHBERI.md](tr/KURULUM_REHBERI.md) | Dağıtım kılavuzu — ikili dosyalar, systemd, Docker, TLS, boyutlandırma |
| [tr/AGENT_KURULUM.md](tr/AGENT_KURULUM.md) | Desktop Agent — kullanıcı makinesinde akıllı kart / USB token ile imzalama |
| [tr/ARCHITECTURE.md](tr/ARCHITECTURE.md) | Sistem mimarisi, bileşen diyagramları, performans ölçümleri |
| [tr/CHANGELOG.md](tr/CHANGELOG.md) | Sürüm geçmişi ve göç notları |

## Geliştirici referansı

| Belge | İçerik |
|---|---|
| [tr/API_REFERANS.md](tr/API_REFERANS.md) | REST API — tüm uç noktalar, istek/yanıt şemaları, hata kodları |
| [tr/SDK_REFERANS.md](tr/SDK_REFERANS.md) | .NET SDK — tam tip ve metot referansı |
| [tr/ORNEKLER.md](tr/ORNEKLER.md) | 17 uçtan uca kullanım senaryosu |
| [tr/KULLANIM_KILAVUZU.md](tr/KULLANIM_KILAVUZU.md) | İlk imzadan itibaren adım adım kullanım kılavuzu |
| [tr/TABLET_RELAY_SIGN.md](tr/TABLET_RELAY_SIGN.md) | Tablet relay-sign API entegrasyonu |

## İmza kavramları

| Belge | İçerik |
|---|---|
| [tr/IMZA_FORMATLARI.md](tr/IMZA_FORMATLARI.md) | CAdES / XAdES / PAdES / ASiC formatları ve seviyeleri (B-B … B-LTA) |
| [tr/SAGLAYICILAR.md](tr/SAGLAYICILAR.md) | İmza sağlayıcıları — PKCS#11, PKCS#12, HSM, mobil, KEP |
| [tr/MOBIL_IMZA.md](tr/MOBIL_IMZA.md) | Mobil imza (operatör tabanlı) entegrasyonu |
| [tr/KEP_ENTEGRASYONU.md](tr/KEP_ENTEGRASYONU.md) | KEP (kayıtlı elektronik posta) entegrasyonu |
| [tr/KAMUSM_TSA_ENTEGRASYONU.md](tr/KAMUSM_TSA_ENTEGRASYONU.md) | Kamu SM zaman damgası (TSA) entegrasyonu |
| [tr/DOGRULAMA_ARACLARI.md](tr/DOGRULAMA_ARACLARI.md) | Üçüncü taraf çevrimiçi imza doğrulama araçları |
| [tr/TURK_MEVZUATI.md](tr/TURK_MEVZUATI.md) | Türkiye elektronik imza mevzuatı uyum analizi |

---

## İndirme

Dokümantasyon aynı zamanda sürüm eki olarak dağıtılır — güncel sürüm ve indirme
bağlantıları için [DigiMR ürün sayfasına](../README.tr.md) bakın.

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
