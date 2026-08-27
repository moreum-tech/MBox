[English](README.md)

# MStore Dokümantasyonu

MStore'un tam ürün dokümantasyonu — S3 uyumlu nesne depolama, gRPC API'si ve istemci SDK'ları.

> **Bunları okumak için hiçbir şey kurmanıza gerek yok.** Aşağıdaki belgelerin
> tamamı yayınlanan MStore v0.3.1 sürümünü anlatır; tek bir ikili dosya
> indirmeden API yüzeyini, yapılandırma seçeneklerini, dağıtım topolojilerini
> ve SDK şeklini inceleyebilirsiniz.

**Dil notu:** belgelerin çoğu iki dillidir (önce Türkçe, sonra İngilizce bölüm).
Tek dilli olanlar aşağıdaki tablolarda işaretlenmiştir.

---

## Buradan başlayın

| Ne yapmak istiyorsanız… | Okuyun |
|---|---|
| Beş dakikada çalıştırmak | [installation.md](installation.md) — ikili dosyalar, systemd, ilk bucket |
| Konteyner olarak çalıştırmak | [docker-deployment.md](docker-deployment.md) — Docker / Podman |
| Kullanım senaryonuza uyup uymadığına karar vermek | [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — segmentler, boyutlandırma, topolojiler |
| Mevcut bir S3 uygulamasını yöneltmek | [s3-api.md](s3-api.md) — S3 API'si ve AWS SDK örnekleri |
| Doğal API'ye karşı kod yazmak | [sdk-reference.md](sdk-reference.md) — altı dilde istemci SDK'ları |
| Bir yapılandırma anahtarına bakmak | [configuration.md](configuration.md) — tüm TOML alanları ve varsayılanları |

## Kurulum ve işletim

| Belge | Kapsam |
|---|---|
| [installation.md](installation.md) | İkili kurulum, systemd birimi, sıfır yapılandırma modu, ilk adımlar |
| [docker-deployment.md](docker-deployment.md) | Konteyner imajı, compose, birimler, sağlık kontrolleri |
| [configuration.md](configuration.md) | Tam `config.toml` referansı, komut satırı seçenekleri, ortam değişkenleri |
| [multidrive-setup.md](multidrive-setup.md) | Çoklu diskte erasure coding — K+M boyutlandırması *(TR)* |
| [multinode-setup.md](multinode-setup.md) | Çok düğümlü küme, yerleşim grupları, eşler |
| [deployment-scenarios.tr.md](deployment-scenarios.tr.md) | MStore'u kimler, hangi topolojiyle kullanır |
| [orphan-reclamation.md](orphan-reclamation.md) | Silme sonrası kalan veri dosyalarının asenkron geri kazanımı *(EN)* |

## Geliştirici referansı

| Belge | Kapsam |
|---|---|
| [s3-api.md](s3-api.md) | S3 HTTP API'si — bucket/nesne işlemleri, alt kaynaklar, presigned URL, tam metin arama, yönetim API'si, AWS SDK örnekleri |
| [grpc-api.md](grpc-api.md) | Doğal gRPC API'si — 18 RPC, mesaj alanları, streaming, batch, `loc_token`, hata modeli |
| [sdk-reference.md](sdk-reference.md) | İstemci SDK'ları — Rust, Python, Go, Java, .NET, JS/TS — ve `mstore` CLI |
| [mstore.proto](mstore.proto) | Protobuf sözleşmesinin kendisi — herhangi bir dilde stub üretin |

---

## İki API, iki port

| Port | Protokol | Kimlik doğrulama | Ne için |
|---|---|---|---|
| 9010 | HTTP(S) üzerinden S3 | AWS SigV4, IAM, presigned URL | Mevcut S3 uygulamaları, tarayıcılar, araçlar |
| 9011 | gRPC | **Yok — her istek root olarak çalışır** | Güvenilir ağda, yüksek verimli istemciler |

> `9011` portu istek başına kimlik doğrulaması yapmaz. Doğrudan internete asla
> açmayın — bkz. [grpc-api.md](grpc-api.md).

---

## İndirmeler

Güncel sürüm ve indirme bağlantıları için
[MStore ürün sayfasına](../README.tr.md) bakın.

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
