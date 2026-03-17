[English](README.md)

# MStore

Rust ile yazılmış, yüksek performanslı, S3 uyumlu nesne depolama sistemi.

---

## İndirmeler

| Bileşen | Tür | Açıklama | Son Sürüm |
|---------|-----|----------|-----------|
| [Sunucu](server/) | Servis | MStore sunucu dosyası | v0.1.0 |
| [CLI](cli/) | Araç | Komut satırı aracı (mstore-ctl) | v0.1.0 |
| [Docker](docker/) | Konteyner | Docker image ve compose | v0.1.0 |
| [Rust SDK](sdk/rust/) | SDK | Rust istemci kütüphanesi | v0.1.0 |
| [C# SDK](sdk/csharp/) | SDK | .NET istemci kütüphanesi | v0.1.0 |
| [Python SDK](sdk/python/) | SDK | Python istemci kütüphanesi | v0.1.0 |
| [Go SDK](sdk/go/) | SDK | Go istemci kütüphanesi | v0.1.0 |
| [Java SDK](sdk/java/) | SDK | Java istemci kütüphanesi | v0.1.0 |
| [JS/TS SDK](sdk/js/) | SDK | JavaScript/TypeScript kütüphanesi | v0.1.0 |

---

## Özellikler

- S3 API uyumlu - AWS SDK'ları, CLI ve tüm S3 araçlarıyla çalışır
- Yerel SDK'lar - Rust, C#, Python, Go, Java, JS/TS
- Şifreleme - SSE-S3 (AES-256-GCM), SSE-KMS, SSE-C
- Erasure coding - Reed-Solomon veri/parite parçaları
- Sıfır kopyalama akışı - büyük nesneler için (>4 MiB)
- Sürümleme - silme işaretçileriyle tam nesne sürümlemesi
- Nesne kilidi - WORM uyumluluğu (SEC 17a-4)
- Web konsolu - tarayıcı tabanlı yönetim arayüzü
- Tam metin arama - bucket bazında Tantivy indeksleri
- Olay sistemi - Webhook, Kafka, AMQP, NATS, SQS, SNS
- Site replikasyonu - gRPC üzerinden asenkron/senkron
- SFTP geçidi - SSH dosya aktarımı erişimi
- IAM - LDAP, OIDC entegrasyonu

## Portlar

| Port | Protokol | Amaç |
|------|----------|------|
| 9010 | HTTP | S3 uyumlu API |
| 9011 | gRPC | MStore SDK, CLI, replikasyon |

---

info@moreum.com | [www.moreum.com](https://www.moreum.com)
