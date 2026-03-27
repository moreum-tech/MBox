[English](README.md)

# MBox

Moreum ürünlerinin resmi dağıtım deposu. Derlenmiş binary dosyaları, Docker image ve SDK paketleri burada yayınlanır.

> Bu depoda kaynak kod bulunmaz. Sadece derlenmiş ve indirilebilir paketler yer alır.

---

## Ürünler

### MStore - Nesne Depolama

Rust ile yazılmış, yüksek performanslı, S3 uyumlu nesne depolama sistemi.

| Bileşen | Açıklama | Son Sürüm |
|---------|----------|-----------|
| [Sunucu](mstore/server/) | MStore sunucu dosyası | v0.1.0 |
| [CLI](mstore/cli/) | Komut satırı aracı (mstore-ctl) | v0.1.0 |
| [Docker](mstore/docker/) | Docker image ve compose | v0.1.0 |
| [SDK - Rust](mstore/sdk/rust/) | Rust istemci kütüphanesi | v0.1.0 |
| [SDK - C#](mstore/sdk/csharp/) | .NET istemci kütüphanesi | v0.1.0 |
| [SDK - Python](mstore/sdk/python/) | Python istemci kütüphanesi | v0.1.0 |
| [SDK - Go](mstore/sdk/go/) | Go istemci kütüphanesi | v0.1.0 |
| [SDK - Java](mstore/sdk/java/) | Java istemci kütüphanesi | v0.1.0 |
| [SDK - JS/TS](mstore/sdk/js/) | JavaScript/TypeScript kütüphanesi | v0.1.0 |

[Tüm MStore indirmelerini görüntüle](mstore/)

---

### DigiMR - Elektronik İmza

Türk elektronik imza platformu — CAdES, PAdES, XAdES, JAdES, ASiC-E. EYP 1.3/2.0, KEP, KamuSM TSA, mobil imza.

| Bileşen | Açıklama | Son Sürüm |
|---------|----------|-----------|
| [API Sunucu](digimr/api/) | REST API sunucu dosyası | v0.1.0 |
| [Token Agent](digimr/agent/) | Donanım token erişimi için yerel PKCS#11 köprüsü | v0.1.0 |
| [Docker](digimr/docker/) | Docker image ve compose | v0.1.0 |
| [SDK - C#](digimr/sdk/csharp/) | .NET istemci kütüphanesi | v0.1.0 |

[Tüm DigiMR indirmelerini görüntüle](digimr/)

---

## Moreum Hakkında

[Moreum](https://www.moreum.com), 2004 yılında kurulan, doküman tabanlı teknoloji çözümlerinde uzmanlaşmış bir yazılım ve danışmanlık şirketidir. Şirket, Avrupa ve Türkiye pazarlarında doküman yönetimi, iş akışı otomasyonu, form işleme ve kaynak planlama alanlarında müşteri odaklı ürün ve hizmetler sunmaktadır.

**İstanbul, Türkiye** | info@moreum.com | [www.moreum.com](https://www.moreum.com)

## Lisans

Aksi belirtilmedikçe tüm ürünler PolyForm Strict License 1.0.0 kapsamında dağıtılmaktadır.
