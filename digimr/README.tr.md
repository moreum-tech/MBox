[English](README.md)

# DigiMR

Turk elektronik imza platformu. CAdES, PAdES, XAdES, JAdES, ASiC-E imzalama ve dogrulama icin bagimsiz REST API.

**Son surum:** [v2.3.1](https://github.com/moreum-tech/MBox/releases/tag/digimr-v2.3.1)

> ⚠️ **v2.3.1 geriye donuk uyumsuz degisiklikler icerir** (v1.x'e gore) — string-path API kaldirildi, tek `byte[]` tabanli yuzey kaldi. Migration icin `digimr-docs.tar.gz` icindeki `CHANGELOG.md` dosyasina bakin.

---

## Dokumantasyon

Tum urun dokumantasyonu [docs/](docs/) klasorundedir — **okumak icin hicbir sey kurmaniza gerek yok**. Icerik `digimr-docs.tar.gz` ile birlikte gelen metnin aynisidir.

| Belge | Kapsam |
|-------|--------|
| [Genel bakis](docs/tr/GENEL_BAKIS.md) | DigiMR nedir, neleri imzalayabilir |
| [Kurulum rehberi](docs/tr/KURULUM_REHBERI.md) | Ikili dosyalar, systemd, Docker, TLS, boyutlandirma |
| [REST API referansi](docs/tr/API_REFERANS.md) | Tum endpoint'ler, semalar ve hata kodlari |
| [.NET SDK referansi](docs/tr/SDK_REFERANS.md) | Tam tip ve metot referansi |
| [Ornekler](docs/tr/ORNEKLER.md) | 17 uctan uca senaryo (C# + curl) |
| [Imza formatlari](docs/tr/IMZA_FORMATLARI.md) | CAdES / XAdES / PAdES / ASiC, B-B … B-LTA seviyeleri |
| [Tum belgeler →](docs/README.tr.md) | Agent kurulumu, saglayicilar, KEP, TSA, mobil imza, mevzuat |

## Indirmeler

| Dosya | Aciklama |
|-------|----------|
| [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-linux-amd64.tar.gz) | API sunucu binary (Linux x86_64) |
| [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-docker.tar.gz) | Docker imaji (`docker load`) |
| [digimr-sdk-1.0.0-all.jar](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-sdk-1.0.0-all.jar) | Java SDK fat JAR |

```bash
# Tumu indir
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-docker.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-sdk-1.0.0-all.jar
```

### Dokumantasyon ve ornekler (release asset)

| Dosya | Aciklama |
|-------|----------|
| [digimr-docs.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-docs.tar.gz) | Gelistirici dokumantasyonu (API referansi, kurulum, imza formatlari, CHANGELOG) |
| [digimr-dotnet-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-dotnet-examples.tar.gz) | .NET ornek projeleri |
| [digimr-java-examples.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/digimr-java-examples.tar.gz) | Java ornek programlari |
| [docker-compose.demo.yml](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.1/docker-compose.demo.yml) | Hazir calistir compose yapilandirmasi |

## Bilesenler

| Bilesen | Aciklama | Detay |
|---------|----------|-------|
| [API Sunucu](api/) | Port 7701 uzerinde REST API | Kurulum, yapilandirma, endpoint'ler |
| [Token Agent](agent/) | Yerel PKCS#11 koprusu (port 5555) | Donanim token erisimi |
| [Docker](docker/) | Container imaji ve compose | Hizli baslangic |

## Ozellikler

- Imza formatlari — CAdES, PAdES, XAdES, JAdES, ASiC-E/S
- Imza seviyeleri — BES, T, LT, LTA (AdES baseline)
- EYP — 1.3 ve 2.0 arsiv paketleri
- KEP — Kayitli elektronik posta paketleri
- TSA — KamuSM ucretsiz/ucretli katman, yapilandirilabilir
- Mobil imza — Turkcell, Vodafone, Turk Telekom
- Token destegi — Yerel agent uzerinden PKCS#11 (AKiS, SafeNet, Gemalto)
- HSM destegi — coklu slot oturum havuzu
- Toplu imzalama — tek cagrida birden fazla belge imzalama
- Dogrulama — CRL/OCSP kontrollu imza dogrulamasi
- Swagger UI — `/swagger` adresinde interaktif API dokumantasyonu

## Port

| Port | Protokol | Amac |
|------|----------|------|
| 7701 | HTTP | REST API ve Swagger UI |

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
