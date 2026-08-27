[English](README.md)

# MStore

Rust ile yazilmis yuksek performansli, S3 uyumlu nesne depolama sunucusu.

**Son surum:** [v0.3.1](https://github.com/moreum-tech/MBox/releases/tag/mstore-v0.3.1)

---

## Dokumantasyon

Tum urun dokumantasyonu [docs/](docs/README.tr.md) klasorundedir — **okumak icin hicbir sey kurmaniza gerek yok**.

| Belge | Kapsam |
|-------|--------|
| [Kurulum](docs/installation.md) | Ikili dosyalar, systemd, sifir yapilandirma modu, ilk bucket |
| [Yapilandirma](docs/configuration.md) | Tum `config.toml` alanlari, CLI secenekleri, ortam degiskenleri |
| [S3 API](docs/s3-api.md) | S3 HTTP API, presigned URL, yonetim API'si, AWS SDK ornekleri |
| [gRPC API](docs/grpc-api.md) | Dogal gRPC API — 18 RPC, streaming, batch, hata modeli |
| [SDK Referansi](docs/sdk-reference.md) | Rust, Python, Go, Java, .NET, JS/TS istemcileri ve CLI |
| [Docker kurulumu](docs/docker-deployment.md) | Container imaji, compose, birimler |
| [Tum belgeler →](docs/README.tr.md) | Coklu disk, coklu dugum, dagitim senaryolari, isletim |

## Indirmeler

| Dosya | Platform | Aciklama |
|-------|----------|----------|
| [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-linux-amd64.tar.gz) | Linux x86_64 | Sunucu + CLI binary (`mstore-server`, `mstore`) |
| [mstore-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-windows-amd64.zip) | Windows x86_64 | Sunucu + CLI binary (`mstore-server.exe`, `mstore.exe`) |
| [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-docker.tar.gz) | Docker / Podman | OCI imaji (`docker load` / `podman load`) |

```bash
# Indir (Linux)
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.3.1/mstore-docker.tar.gz
```

## Bilesenler

| Bilesen | Aciklama | Detay |
|---------|----------|-------|
| [Sunucu](server/README.tr.md) | S3 HTTP + gRPC endpoint | Kurulum, yapilandirma |
| [Docker](docker/README.tr.md) | Container imaji ve compose | Hizli baslangic |

## Ozellikler

- S3 API uyumlu — AWS SDK, CLI ve tum S3 araclariyla calisir
- Erasure coding — Reed-Solomon veri/parity parcalari
- Sifreleme — AES-256-GCM, ChaCha20-Poly1305
- Nesne versiyonlama ve delete marker
- Nesne kilitleme — WORM uyumlulugu (SEC 17a-4)
- Tam metin arama — bucket bazli Tantivy indeksleri
- IAM — kullanici/grup politikalari, LDAP, OIDC
- Site replikasyonu — gRPC uzerinden async/sync
- SFTP gecidi — SSH dosya erisimi
- Web konsol — tarayici tabanli yonetim arayuzu
- Toplu islemler — toplu silme/kopyalama/etiketleme
- S3 Select — CSV ve JSON sorgu

## Portlar

| Port | Protokol | Amac |
|------|----------|------|
| 9010 | HTTP | S3 uyumlu API |
| 9011 | gRPC | SDK, CLI, replikasyon |

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
