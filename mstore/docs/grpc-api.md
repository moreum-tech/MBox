# MStore gRPC API Referansı / gRPC API Reference

> Sürüm / Version: **v0.4.0** · Varsayılan uç nokta / Default endpoint: `http://<host>:9011`
> Proto paketi / package: `mstore.v1` · Servis / service: `ObjectStorage`

## İçindekiler / Table of Contents

- [Türkçe](#türkçe)
- [English](#english)

---

# Türkçe

MStore, S3 HTTP API'sinin yanında `9011` portunda ikili bir gRPC arayüzü sunar.
Aynı `ObjectStorage` motoruna bağlanır; protokol katmanında iş mantığı yoktur —
yani gRPC ve S3 üzerinden yazılan veri birebir aynıdır.

**gRPC'yi ne zaman tercih etmelisiniz:** yüksek hacimli yazma/okuma, düşük
gecikme gereksinimi, gerçek streaming (zero-copy), toplu (batch) işlemler ve
`loc_token` ile metadata aramasını tamamen atlayan hızlı GET.

---

## ⚠️ Güvenlik: gRPC portu kimlik doğrulaması yapmaz

> **`9011` portuna ulaşan her istek root yetkisiyle çalışır.** gRPC katmanında
> istek başına kimlik doğrulama (SigV4, token, kullanıcı) **yoktur**; bu port
> güvenilir bir taşıma katmanı varsayar.
>
> Sonuçları:
> - `9011` portunu **asla** doğrudan internete açmayın.
> - Erişimi güvenlik duvarı, özel ağ veya VPN ile sınırlayın.
> - Uygulama sunucularınız dışında kimseye açmayın.
> - Karşılıklı doğrulama gerekiyorsa taşıma katmanında **mTLS** kullanın.
>
> Kimlik doğrulaması, IAM politikaları, kullanıcı bazlı yetkilendirme ve denetim
> kaydı gereken istemciler **S3 HTTP API'sini (`9010`)** kullanmalıdır.
> Bkz. [s3-api.md](s3-api.md).

---

## 1. Bağlanma

```bash
# Proto dosyası bu klasörde: mstore.proto
# Herhangi bir dilde stub üretebilirsiniz:
protoc --proto_path=. --python_out=. --grpc_python_out=. mstore.proto
```

`grpcurl` ile hızlı deneme:

```bash
grpcurl -plaintext -proto mstore.proto \
  -d '{"bucket":"test"}' \
  127.0.0.1:9011 mstore.v1.ObjectStorage/CreateBucket

grpcurl -plaintext -proto mstore.proto \
  -d '{}' 127.0.0.1:9011 mstore.v1.ObjectStorage/ListBuckets
```

| Ayar | Değer |
|---|---|
| Bind adresi | `0.0.0.0:9011` (`MSTORE_GRPC_BIND` ile değiştirilir) |
| Maks. mesaj boyutu | 256 MiB (gönderme ve alma) |
| Şifreleme | Yok (varsayılan). mTLS taşıma katmanında yapılandırılır. |

## 2. Servis: `mstore.v1.ObjectStorage`

### Küçük nesneler (< 4 MB)

| RPC | İstek | Yanıt |
|---|---|---|
| `PutObject` | `PutObjectRequest` | `PutObjectResponse` |
| `GetObject` | `GetObjectRequest` | `GetObjectResponse` |
| `HeadObject` | `HeadObjectRequest` | `ObjectInfo` |
| `DeleteObject` | `DeleteObjectRequest` | `DeleteObjectResponse` |
| `ListObjects` | `ListObjectsRequest` | `ListObjectsResponse` |

### Büyük nesneler — streaming (≥ 4 MB)

| RPC | Yön | İstek | Yanıt |
|---|---|---|---|
| `PutObjectStream` | istemci → sunucu | `stream PutChunk` | `PutObjectResponse` |
| `GetObjectStream` | sunucu → istemci | `GetObjectRequest` | `stream GetChunk` |

İlk `PutChunk` mesajı `meta` alanını (`PutChunkMeta`) taşır; sonraki mesajlar
yalnızca `data` içerir. `GetChunk` için de aynı desen geçerlidir.

### Multipart

| RPC | İstek | Yanıt |
|---|---|---|
| `CreateMultipartUpload` | `CreateMultipartRequest` | `CreateMultipartResponse` |
| `UploadPart` | `stream UploadPartChunk` | `UploadPartResponse` |
| `CompleteMultipartUpload` | `CompleteMultipartRequest` | `PutObjectResponse` |
| `AbortMultipartUpload` | `AbortMultipartRequest` | `Empty` |

### Toplu (batch) işlemler

| RPC | Yön | İstek | Yanıt |
|---|---|---|---|
| `DeleteObjects` | tek | `DeleteObjectsRequest` | `DeleteObjectsResponse` |
| `PutObjects` | çift yönlü | `stream PutObjectsChunk` | `stream PutObjectResult` |
| `GetObjects` | sunucu akışı | `GetObjectsRequest` | `stream GetObjectResult` |

`PutObjects` çift yönlüdür: istemci nesneleri akış hâlinde gönderirken sunucu
tamamlananları anında geri bildirir — binlerce küçük dosyayı tek bağlantı
üzerinden pipeline'layarak yazmak için tasarlanmıştır.

### Bucket

| RPC | İstek | Yanıt |
|---|---|---|
| `CreateBucket` | `CreateBucketRequest` | `Empty` |
| `DeleteBucket` | `DeleteBucketRequest` | `Empty` |
| `ListBuckets` | `ListBucketsRequest` | `ListBucketsResponse` |
| `HeadBucket` | `HeadBucketRequest` | `BucketInfo` |

## 3. Önemli Mesajlar

### `PutObjectRequest`

| Alan | Tip | Açıklama |
|---|---|---|
| `bucket` | string | Bucket adı |
| `key` | string | Nesne anahtarı |
| `data` | bytes | Nesnenin tamamı (≤ 4 MB) |
| `content_type` | string | MIME tipi |
| `storage_class` | string | `STANDARD` \| `REDUCED_REDUNDANCY` |
| `metadata` | map<string,string> | Kullanıcı metadata'sı (arama indeksine girer) |
| `checksum_blake3` | bytes | Opsiyonel istemci tarafı BLAKE3 — sunucu doğrular |

### `PutObjectResponse`

| Alan | Tip | Açıklama |
|---|---|---|
| `version_id` | string | UUID v7 |
| `etag` | string | Hex kodlu BLAKE3 |
| `request_id` | string | İzleme kimliği |
| `loc_token` | string | **Sıfır-arama tokenı** — sonraki GET'lerde metadata aramasını atlar (inline nesnelerde boş) |

### `GetObjectRequest`

| Alan | Tip | Açıklama |
|---|---|---|
| `bucket`, `key` | string | Hedef nesne |
| `version_id` | string | Boş = en güncel sürüm |
| `range_start`, `range_end` | int64 | Kısmi okuma (0 = baştan / sona kadar) |
| `loc_token` | string | Doluysa metadata araması atlanır — en düşük gecikmeli GET yolu |

### `ObjectInfo`

`bucket`, `key`, `version_id`, `etag`, `size`, `content_type`, `last_modified`
(Unix saniye), `storage_class`, `metadata`, `is_delete_marker`.

### `ListObjectsRequest` / `ListObjectsResponse`

İstek: `bucket`, `prefix`, `delimiter`, `max_keys` (varsayılan 1000, en fazla
1000), `page_token`.
Yanıt: `objects`, `common_prefixes`, `next_page_token`, `is_truncated`,
`request_id`.

### `DeleteObjectsResponse`

Kısmi başarıyı ayrıştırır: `deleted` (başarılı) ve `errors` (`key`, `code`,
`message`) listeleri ayrı döner — bir nesnenin başarısız olması diğerlerini
etkilemez.

### `PutObjectResult` (batch)

`key`, `version_id`, `etag`, `loc_token` ve hata durumunda `error_code` /
`error_msg`. Boş `error_code` başarı demektir.

## 4. `loc_token` — sıfır-arama GET

`PutObject`, `PutObjectStream` ve batch yazma işlemleri bir `loc_token` döner.
Bu tokenı saklayıp sonraki `GetObject` / `GetObjects` çağrılarında verirseniz
sunucu metadata aramasını tamamen atlar ve doğrudan veriye gider.

Kendi veritabanınızda nesne referansı tutan uygulamalar için: `loc_token`'ı
kendi kaydınızda `version_id` ile birlikte saklayın.

## 5. Hata Modeli

Standart gRPC status kodları kullanılır:

| gRPC kodu | Anlamı |
|---|---|
| `NOT_FOUND` | Bucket veya nesne yok |
| `ALREADY_EXISTS` | Bucket zaten var |
| `FAILED_PRECONDITION` | Bucket boş değil (`force_empty` olmadan silme) |
| `INVALID_ARGUMENT` | Geçersiz istek alanı, checksum uyuşmazlığı |
| `RESOURCE_EXHAUSTED` | Kota veya eşzamanlılık sınırı |
| `INTERNAL` | Sunucu tarafı hata |

Toplu işlemlerde tekil hatalar RPC'yi düşürmez; yanıt içindeki `errors` /
`error_code` alanlarında döner.

## 6. Proto Dosyası

Tam sözleşme bu klasördedir: [mstore.proto](mstore.proto)

```bash
# Python
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. mstore.proto

# Go
protoc -I. --go_out=. --go-grpc_out=. mstore.proto

# Java
protoc -I. --java_out=. --grpc-java_out=. mstore.proto

# C#
protoc -I. --csharp_out=. --grpc_out=. --plugin=protoc-gen-grpc=grpc_csharp_plugin mstore.proto
```

Hazır istemciler için: [sdk-reference.md](sdk-reference.md)

## İlgili Belgeler

- [s3-api.md](s3-api.md) — S3 HTTP API (kimlik doğrulamalı)
- [sdk-reference.md](sdk-reference.md) — 6 dilde hazır istemci
- [installation.md](installation.md) — Kurulum

---

# English

Alongside the S3 HTTP API, MStore exposes a binary gRPC interface on port
`9011`. It talks to the same `ObjectStorage` engine — no business logic lives in
the protocol layer — so data written over gRPC and over S3 is identical.

**When to prefer gRPC:** high-volume writes/reads, low-latency requirements,
true streaming (zero-copy), batch operations, and fast GETs that skip the
metadata lookup entirely via `loc_token`.

---

## ⚠️ Security: the gRPC port does not authenticate

> **Every request that reaches port `9011` runs with root identity.** There is
> **no** per-request authentication (SigV4, token, user) at the gRPC layer; the
> port assumes a trusted transport.
>
> Consequences:
> - **Never** expose `9011` directly to the internet.
> - Restrict access with a firewall, a private network, or a VPN.
> - Reach it only from your own application servers.
> - Use **mTLS** at the transport layer if you need mutual authentication.
>
> Clients that need authentication, IAM policies, per-user authorization or
> audit logging must use the **S3 HTTP API on `9010`**. See [s3-api.md](s3-api.md).

---

## 1. Connecting

```bash
# The proto file is in this folder: mstore.proto
# Generate stubs for any language:
protoc --proto_path=. --python_out=. --grpc_python_out=. mstore.proto
```

Quick check with `grpcurl`:

```bash
grpcurl -plaintext -proto mstore.proto \
  -d '{"bucket":"test"}' \
  127.0.0.1:9011 mstore.v1.ObjectStorage/CreateBucket

grpcurl -plaintext -proto mstore.proto \
  -d '{}' 127.0.0.1:9011 mstore.v1.ObjectStorage/ListBuckets
```

| Setting | Value |
|---|---|
| Bind address | `0.0.0.0:9011` (override with `MSTORE_GRPC_BIND`) |
| Max message size | 256 MiB (both encode and decode) |
| Encryption | None by default. mTLS is configured at the transport layer. |

## 2. Service: `mstore.v1.ObjectStorage`

### Small objects (< 4 MB)

| RPC | Request | Response |
|---|---|---|
| `PutObject` | `PutObjectRequest` | `PutObjectResponse` |
| `GetObject` | `GetObjectRequest` | `GetObjectResponse` |
| `HeadObject` | `HeadObjectRequest` | `ObjectInfo` |
| `DeleteObject` | `DeleteObjectRequest` | `DeleteObjectResponse` |
| `ListObjects` | `ListObjectsRequest` | `ListObjectsResponse` |

### Large objects — streaming (≥ 4 MB)

| RPC | Direction | Request | Response |
|---|---|---|---|
| `PutObjectStream` | client → server | `stream PutChunk` | `PutObjectResponse` |
| `GetObjectStream` | server → client | `GetObjectRequest` | `stream GetChunk` |

The first `PutChunk` carries the `meta` field (`PutChunkMeta`); subsequent
messages carry only `data`. `GetChunk` follows the same pattern.

### Multipart

| RPC | Request | Response |
|---|---|---|
| `CreateMultipartUpload` | `CreateMultipartRequest` | `CreateMultipartResponse` |
| `UploadPart` | `stream UploadPartChunk` | `UploadPartResponse` |
| `CompleteMultipartUpload` | `CompleteMultipartRequest` | `PutObjectResponse` |
| `AbortMultipartUpload` | `AbortMultipartRequest` | `Empty` |

### Batch operations

| RPC | Direction | Request | Response |
|---|---|---|---|
| `DeleteObjects` | unary | `DeleteObjectsRequest` | `DeleteObjectsResponse` |
| `PutObjects` | bidirectional | `stream PutObjectsChunk` | `stream PutObjectResult` |
| `GetObjects` | server stream | `GetObjectsRequest` | `stream GetObjectResult` |

`PutObjects` is bidirectional: the client streams objects while the server
reports completions as they land — designed for pipelining thousands of small
files over a single connection.

### Bucket

| RPC | Request | Response |
|---|---|---|
| `CreateBucket` | `CreateBucketRequest` | `Empty` |
| `DeleteBucket` | `DeleteBucketRequest` | `Empty` |
| `ListBuckets` | `ListBucketsRequest` | `ListBucketsResponse` |
| `HeadBucket` | `HeadBucketRequest` | `BucketInfo` |

## 3. Key Messages

### `PutObjectRequest`

| Field | Type | Description |
|---|---|---|
| `bucket` | string | Bucket name |
| `key` | string | Object key |
| `data` | bytes | Entire object (≤ 4 MB) |
| `content_type` | string | MIME type |
| `storage_class` | string | `STANDARD` \| `REDUCED_REDUNDANCY` |
| `metadata` | map<string,string> | User metadata (fed into the search index) |
| `checksum_blake3` | bytes | Optional client-side BLAKE3 — verified by the server |

### `PutObjectResponse`

| Field | Type | Description |
|---|---|---|
| `version_id` | string | UUID v7 |
| `etag` | string | Hex-encoded BLAKE3 |
| `request_id` | string | Tracing id |
| `loc_token` | string | **Zero-lookup token** — skips the metadata lookup on later GETs (empty for inline objects) |

### `GetObjectRequest`

| Field | Type | Description |
|---|---|---|
| `bucket`, `key` | string | Target object |
| `version_id` | string | Empty = latest version |
| `range_start`, `range_end` | int64 | Partial read (0 = from start / to end) |
| `loc_token` | string | When set, the metadata lookup is skipped — the lowest-latency GET path |

### `ObjectInfo`

`bucket`, `key`, `version_id`, `etag`, `size`, `content_type`, `last_modified`
(Unix seconds), `storage_class`, `metadata`, `is_delete_marker`.

### `ListObjectsRequest` / `ListObjectsResponse`

Request: `bucket`, `prefix`, `delimiter`, `max_keys` (default 1000, max 1000),
`page_token`.
Response: `objects`, `common_prefixes`, `next_page_token`, `is_truncated`,
`request_id`.

### `DeleteObjectsResponse`

Partial success is explicit: `deleted` (successes) and `errors` (`key`, `code`,
`message`) come back as separate lists — one object failing does not affect the
others.

### `PutObjectResult` (batch)

`key`, `version_id`, `etag`, `loc_token`, plus `error_code` / `error_msg` on
failure. An empty `error_code` means success.

## 4. `loc_token` — zero-lookup GET

`PutObject`, `PutObjectStream` and batch writes return a `loc_token`. Store it
and pass it back in later `GetObject` / `GetObjects` calls and the server skips
the metadata lookup entirely, going straight to the data.

For applications that keep their own object references: store `loc_token`
alongside `version_id` in your own record.

## 5. Error Model

Standard gRPC status codes are used:

| gRPC code | Meaning |
|---|---|
| `NOT_FOUND` | Bucket or object does not exist |
| `ALREADY_EXISTS` | Bucket already exists |
| `FAILED_PRECONDITION` | Bucket not empty (delete without `force_empty`) |
| `INVALID_ARGUMENT` | Invalid request field, checksum mismatch |
| `RESOURCE_EXHAUSTED` | Quota or concurrency limit |
| `INTERNAL` | Server-side error |

In batch operations individual failures do not fail the RPC; they come back in
the response's `errors` / `error_code` fields.

## 6. Proto File

The full contract is in this folder: [mstore.proto](mstore.proto)

```bash
# Python
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. mstore.proto

# Go
protoc -I. --go_out=. --go-grpc_out=. mstore.proto

# Java
protoc -I. --java_out=. --grpc-java_out=. mstore.proto

# C#
protoc -I. --csharp_out=. --grpc_out=. --plugin=protoc-gen-grpc=grpc_csharp_plugin mstore.proto
```

For ready-made clients see [sdk-reference.md](sdk-reference.md).

## Related Documents

- [s3-api.md](s3-api.md) — S3 HTTP API (authenticated)
- [sdk-reference.md](sdk-reference.md) — ready-made clients in 6 languages
- [installation.md](installation.md) — installation
