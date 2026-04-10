[English](README.md)

# MStore

Rust ile yazilmis yuksek performansli, S3 uyumlu nesne depolama sunucusu.

**Son surum:** [v0.2.0](https://github.com/moreum-tech/MBox/releases/tag/mstore-v0.2.0)

---

## Indirmeler

| Dosya | Aciklama |
|-------|----------|
| [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz) | Sunucu + CLI binary (Linux x86_64) |
| [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz) | Docker imaji (`docker load`) |

```bash
# Tumu indir
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz
```

## Bilesenler

| Bilesen | Aciklama | Detay |
|---------|----------|-------|
| [Sunucu](server/) | S3 HTTP + gRPC endpoint | Kurulum, yapilandirma |
| [Docker](docker/) | Container imaji ve compose | Hizli baslangic |

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
