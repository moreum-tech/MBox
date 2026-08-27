[English](README.md)

# MBox

[Moreum](https://www.moreum.com) urunlerinin resmi dagitim deposu.

Derlenmis binary dosyalar, Docker imajlari ve SDK paketleri [GitHub Releases](https://github.com/moreum-tech/MBox/releases) uzerinden yayinlanir.

---

## Urunler

### MStore — Nesne Depolama

Rust ile yazilmis yuksek performansli, S3 uyumlu nesne depolama sunucusu.

| Bilesen | Son Surum | Indir |
|---------|-----------|-------|
| [Sunucu + CLI](mstore/server/README.tr.md) | v0.4.0 | [Linux amd64](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-linux-amd64.tar.gz) · [Windows amd64](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-windows-amd64.zip) |
| [Docker / Podman](mstore/docker/README.tr.md) | v0.4.0 | [Imaj](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-docker.tar.gz) |

**Dokumantasyon:** [mstore/docs/](mstore/docs/README.tr.md) — kurulum, yapilandirma, S3 ve gRPC API, SDK referansi. Okumak icin kurulum gerekmez.

[Tum MStore surumleri](https://github.com/moreum-tech/MBox/releases?q=mstore)

---

### DigiMR — Elektronik Imza

Turk elektronik imza platformu — CAdES, PAdES, XAdES, JAdES, ASiC-E. EYP 1.3/2.0, KEP, KamuSM TSA, mobil imza.

| Bilesen | Son Surum | Indir |
|---------|-----------|-------|
| [API Sunucu](digimr/api/README.tr.md) | v2.3.2 | [Linux amd64](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-linux-amd64.tar.gz) |
| [Docker](digimr/docker/README.tr.md) | v2.3.2 | [Imaj](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-docker.tar.gz) |
| [Java SDK](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.3.2/digimr-sdk-1.0.0-all.jar) | 1.0.0 | JAR |

**Dokumantasyon:** [digimr/docs/](digimr/docs/README.tr.md) — kurulum rehberi, REST API referansi, .NET SDK referansi, imza formatlari. Okumak icin kurulum gerekmez.

[Tum DigiMR surumleri](https://github.com/moreum-tech/MBox/releases?q=digimr)

---

## Hakkinda

[Moreum](https://www.moreum.com), 2004 yilinda kurulan, belge yonetimi, is akisi otomasyonu ve dijital imza cozumlerinde uzmanlasmis bir yazilim ve danismanlik firmasidir.

**Istanbul, Turkiye** · info@moreum.com · [www.moreum.com](https://www.moreum.com)

## Lisans

Tum urunler aksi belirtilmedikce PolyForm Strict License 1.0.0 kapsaminda dagitilir.
