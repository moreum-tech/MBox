# MStore SDK Referansı / SDK Reference

> Sürüm / Version: **v0.3.1**

## İçindekiler / Table of Contents

- [Türkçe](#türkçe)
- [English](#english)

---

# Türkçe

## 1. Hangi İstemciyi Kullanmalıyım?

| Durum | Önerilen istemci | Port |
|---|---|---|
| Mevcut bir S3 uygulamasını taşıyorsunuz | **AWS SDK** (boto3, AWSSDK.S3, aws-sdk-java, …) | 9010 |
| Kimlik doğrulama, IAM, denetim kaydı gerekiyor | **AWS SDK** | 9010 |
| Tarayıcıdan doğrudan yükleme/indirme | **Presigned URL** | 9010 |
| Maksimum verim, düşük gecikme, toplu işlem | **MStore gRPC istemcisi** | 9011 |
| Komut satırı / betikler | **`mstore` CLI** (sürüm arşivinde) | 9011 |

En kısa yol AWS SDK'sıdır: uç noktayı MStore'a çevirmeniz yeterlidir, kod
değişmez. Örnekler: [s3-api.md § 7](s3-api.md#7-aws-sdk-örnekleri)

> **Güvenlik notu:** gRPC portu (`9011`) istek başına kimlik doğrulaması
> yapmaz — ulaşan her istek root yetkisiyle çalışır. Yalnızca güvenilir ağdan
> erişin. Ayrıntı: [grpc-api.md](grpc-api.md)

## 2. gRPC İstemcisi Edinme

MStore gRPC istemcileri şu an **genel paket depolarında yayınlanmamaktadır**
(PyPI, npm, Maven Central, NuGet, crates.io). İki yol vardır:

**a) Kendi stub'larınızı üretin** — sözleşme bu klasördedir:
[mstore.proto](mstore.proto). Üretim komutları:
[grpc-api.md § 6](grpc-api.md#6-proto-dosyası)

**b) Hazır istemci kaynağı isteyin** — 6 dilde referans istemci mevcuttur;
**info@moreum.com** adresinden talep edebilirsiniz.

Aşağıdaki bölümler bu referans istemcilerin API yüzeyini belgeler: kurmadan
önce entegrasyonun nasıl görüneceğini değerlendirebilirsiniz.

## 3. Ortak Davranış

Tüm istemciler aynı desenleri paylaşır:

| Davranış | Değer |
|---|---|
| Streaming eşiği | 4 MiB — bu boyutun altındaki nesneler tek mesajda, üstündekiler otomatik olarak chunk'lanarak gönderilir |
| Chunk boyutu | 4 MiB |
| Maks. mesaj boyutu | 16 MiB (istemci varsayılanı; sunucu 256 MiB kabul eder) |
| Varsayılan uç nokta | `localhost:9011` |

Uygulama kodu `put(bucket, key, data)` çağırır; küçük/büyük nesne seçimini
istemci kendisi yapar.

## 4. Rust — `mstore-client`

En geniş yüzeye sahip istemci; multipart ve batch işlemleri de kapsar.

```rust
use mstore_client::MStoreClient;

let client = MStoreClient::new("http://127.0.0.1:9011").await?;

client.create_bucket("belgeler").await?;
let out = client.put_object("belgeler", "rapor.pdf", bytes).await?;
let obj = client.get_object("belgeler", "rapor.pdf").await?;
```

| Grup | Metotlar |
|---|---|
| Bağlantı | `builder()`, `connect(config)`, `new(endpoint)` |
| Yazma | `put_object`, `put_object_with` |
| Okuma | `get_object`, `get_object_version`, `get_object_range`, `get_object_with` |
| Bilgi | `head_object`, `head_object_with`, `object_exists` |
| Listeleme | `list_objects`, `list_objects_prefix`, `list_objects_with`, `list_all_objects` |
| Silme | `delete_object`, `delete_object_version` |
| Bucket | `create_bucket`, `delete_bucket`, `list_buckets` |
| Multipart | `create_multipart_upload`, `upload_part`, `complete_multipart_upload`, `abort_multipart_upload` |
| Batch | `delete_objects`, `delete_objects_with`, `put_objects`, `get_objects`, `get_objects_with` |

`list_all_objects` sayfalamayı sizin yerinize yürütür; `object_exists` bir
`head_object` çağrısını bool'a indirger.

## 5. Python — `mstore-sdk`

```python
from mstore import MStoreClient

with MStoreClient("localhost:9011") as client:
    client.create_bucket("belgeler")
    client.put_object("belgeler", "rapor.pdf", data,
                      content_type="application/pdf",
                      metadata={"proje": "alpha"})
    obj = client.get_object("belgeler", "rapor.pdf")
    for item in client.list_all_objects("belgeler", prefix="2026/"):
        print(item.key, item.size)
```

| Metot | Açıklama |
|---|---|
| `MStoreClient(endpoint="localhost:9011", *, secure=False, max_message_size=16MiB)` | Bağlantı; context manager destekler |
| `put_object(bucket, key, data, *, content_type, storage_class, metadata)` | 4 MiB üstünde otomatik streaming |
| `get_object(bucket, key, ...)` | Tam nesneyi belleğe alır |
| `get_object_stream(bucket, key)` | Chunk chunk okur — büyük nesneler için |
| `head_object(bucket, key)` | Metadata (indirmeden) |
| `delete_object(bucket, key)` | Siler / delete marker yazar |
| `list_objects(bucket, ...)` | Tek sayfa |
| `list_all_objects(bucket, ...)` | Tüm sayfaları dolaşan üreteç |
| `create_bucket(bucket, *, region="")` · `delete_bucket(bucket, *, force=False)` · `list_buckets()` | Bucket işlemleri |
| `upload_file(...)` · `download_file(...)` | Dosya yolu ile kısayol |
| `close()` | Kanalı kapatır |

Gereksinimler: Python ≥ 3.9, `grpcio ≥ 1.60`, `protobuf ≥ 4.25`.

## 6. Go — `github.com/mstore/mstore-go`

```go
client, err := mstore.NewClient("127.0.0.1:9011")
if err != nil { return err }
defer client.Close()

ctx := context.Background()
if err := client.CreateBucket(ctx, "belgeler"); err != nil { return err }
out, err := client.PutObject(ctx, "belgeler", "rapor.pdf", data)
obj, err := client.GetObject(ctx, "belgeler", "rapor.pdf")
```

| Metot | İmza özeti |
|---|---|
| `NewClient(endpoint, opts ...ClientOptions)` | Bağlantı |
| `Close()` | Kapatır |
| `PutObject(ctx, bucket, key, data, opts ...PutOptions)` | `*PutObjectOutput` |
| `GetObject(ctx, bucket, key, opts ...GetOptions)` | `*GetObjectOutput` |
| `GetObjectStream(ctx, bucket, key)` | `io.Reader` + `*GetStreamMeta` |
| `HeadObject(ctx, bucket, key)` | `*HeadObjectOutput` |
| `DeleteObject(ctx, bucket, key)` | `*DeleteObjectOutput` |
| `ListObjects(ctx, bucket, opts ...ListOptions)` | `*ListObjectsOutput` |
| `CreateBucket(ctx, bucket)` · `DeleteBucket(ctx, bucket, force)` · `ListBuckets(ctx)` | Bucket işlemleri |

Go ≥ 1.22, `google.golang.org/grpc v1.64`.

## 7. Java — `com.mstore:mstore-client`

```java
try (MStoreClient client = new MStoreClient("127.0.0.1", 9011)) {
    client.createBucket("belgeler");
    PutResult r = client.putObject("belgeler", "rapor.pdf", data);
    GetResult g = client.getObject("belgeler", "rapor.pdf");
    ListResult l = client.listObjects("belgeler", "2026/", "/", 1000);
}
```

| Metot | Dönüş |
|---|---|
| `putObject(bucket, key, data)` / `(…, contentType, storageClass, metadata)` | `PutResult(versionId, etag, requestId)` |
| `getObject(bucket, key)` / `(…, versionId, rangeStart, rangeEnd)` | `GetResult(body, versionId, etag, …)` |
| `headObject(bucket, key)` | `HeadResult(versionId, etag, size, …)` |
| `deleteObject(bucket, key)` | `DeleteResult(deleteMarker, deleteMarkerId, requestId)` |
| `listObjects(bucket, prefix, delimiter, maxKeys)` | `ListResult(objects, commonPrefixes, …)` |
| `createBucket` · `deleteBucket(bucket, force)` · `listBuckets()` | — / `List<BucketItemResult>` |
| `uploadFile(bucket, key, Path)` · `downloadFile(bucket, key, Path)` | Dosya kısayolları |
| `close()` | `AutoCloseable` |

Sabitler: `STREAMING_THRESHOLD = 4 MiB`, `CHUNK_SIZE = 4 MiB`.

## 8. .NET — `MStore.Client`

```csharp
using var client = new MStoreClient("http://127.0.0.1:9011");

await client.CreateBucketAsync("belgeler");
var put = await client.PutObjectAsync("belgeler", "rapor.pdf", data);
var get = await client.GetObjectAsync("belgeler", "rapor.pdf");
```

| Metot | Dönüş |
|---|---|
| `PutObjectAsync(...)` | `PutObjectResult(VersionId, ETag, RequestId)` |
| `GetObjectAsync(...)` | `GetObjectResult` |
| `HeadObjectAsync(...)` | `HeadObjectResult` |
| `DeleteObjectAsync(...)` | `DeleteObjectResult` |
| `ListObjectsAsync(...)` | `ListObjectsResult` |
| `CreateBucketAsync` · `DeleteBucketAsync(bucket, force)` · `ListBucketsAsync()` | — / `List<BucketItemResult>` |
| `UploadFileAsync` · `DownloadFileAsync` | Dosya kısayolları |
| `Dispose()` | `IDisposable` |

`net8.0`, `Grpc.Net.Client 2.63`. Sabitler: `StreamingThreshold`, `ChunkSize` = 4 MiB.

## 9. JavaScript / TypeScript — `@mstore/sdk`

```ts
import { MStoreClient } from '@mstore/sdk';

const client = new MStoreClient('127.0.0.1:9011');
await client.createBucket('belgeler');
await client.putObject('belgeler', 'rapor.pdf', buffer, { contentType: 'application/pdf' });
const obj = await client.getObject('belgeler', 'rapor.pdf');
client.close();
```

| Metot | Dönüş |
|---|---|
| `putObject(bucket, key, data, opts?)` | `Promise<PutObjectOutput>` |
| `getObject(bucket, key, opts?)` | `Promise<GetObjectOutput>` |
| `headObject(bucket, key)` | `Promise<HeadObjectOutput>` |
| `deleteObject(bucket, key)` | `Promise<DeleteObjectOutput>` |
| `listObjects(bucket, opts?)` | `Promise<ListObjectsOutput>` |
| `createBucket(bucket, region?)` · `deleteBucket(bucket, force?)` · `listBuckets()` | `Promise<void>` / `Promise<BucketItem[]>` |
| `close()` | — |

`@grpc/grpc-js ^1.10`. Node.js için — tarayıcıda gRPC yerine
[presigned URL](s3-api.md#presigned-url-üretme) kullanın.

## 10. CLI — `mstore`

Sürüm arşivinden çıkar, ayrıca kurulum gerektirmez:

```bash
mstore mb     mstore://belgeler
mstore cp     rapor.pdf mstore://belgeler/rapor.pdf
mstore ls     mstore://belgeler/
mstore head   mstore://belgeler/rapor.pdf
mstore rm     mstore://belgeler/rapor.pdf
mstore rb     mstore://belgeler
mstore search mstore://belgeler "sözleşme"
```

## İlgili Belgeler

- [grpc-api.md](grpc-api.md) — Ham gRPC sözleşmesi ve mesaj alanları
- [s3-api.md](s3-api.md) — S3 HTTP API ve AWS SDK örnekleri
- [installation.md](installation.md) — Kurulum
- [mstore.proto](mstore.proto) — Protobuf sözleşmesi

---

# English

## 1. Which Client Should I Use?

| Situation | Recommended client | Port |
|---|---|---|
| Migrating an existing S3 application | **AWS SDK** (boto3, AWSSDK.S3, aws-sdk-java, …) | 9010 |
| You need authentication, IAM, audit logging | **AWS SDK** | 9010 |
| Direct browser upload/download | **Presigned URL** | 9010 |
| Maximum throughput, low latency, batch work | **MStore gRPC client** | 9011 |
| Command line / scripts | **`mstore` CLI** (in the release archive) | 9011 |

The shortest path is an AWS SDK: point the endpoint at MStore and your code is
unchanged. Examples: [s3-api.md § 7](s3-api.md#7-aws-sdk-examples)

> **Security note:** the gRPC port (`9011`) performs no per-request
> authentication — every request that reaches it runs as root. Only reach it
> from a trusted network. See
> [grpc-api.md](grpc-api.md)

## 2. Obtaining a gRPC Client

The MStore gRPC clients are **not currently published to public package
registries** (PyPI, npm, Maven Central, NuGet, crates.io). There are two routes:

**a) Generate your own stubs** — the contract is in this folder:
[mstore.proto](mstore.proto). Generation commands:
[grpc-api.md § 6](grpc-api.md#6-proto-file)

**b) Request the ready-made client source** — reference clients exist in six
languages; ask at **info@moreum.com**.

The sections below document those reference clients' API surface, so you can
evaluate what an integration would look like before installing anything.

## 3. Shared Behaviour

Every client shares the same patterns:

| Behaviour | Value |
|---|---|
| Streaming threshold | 4 MiB — objects below it go in one message, above it are chunked automatically |
| Chunk size | 4 MiB |
| Max message size | 16 MiB (client default; the server accepts 256 MiB) |
| Default endpoint | `localhost:9011` |

Application code calls `put(bucket, key, data)`; the client picks the
small-object or streaming path itself.

## 4. Rust — `mstore-client`

The widest surface, including multipart and batch operations.

```rust
use mstore_client::MStoreClient;

let client = MStoreClient::new("http://127.0.0.1:9011").await?;

client.create_bucket("documents").await?;
let out = client.put_object("documents", "report.pdf", bytes).await?;
let obj = client.get_object("documents", "report.pdf").await?;
```

| Group | Methods |
|---|---|
| Connection | `builder()`, `connect(config)`, `new(endpoint)` |
| Write | `put_object`, `put_object_with` |
| Read | `get_object`, `get_object_version`, `get_object_range`, `get_object_with` |
| Info | `head_object`, `head_object_with`, `object_exists` |
| List | `list_objects`, `list_objects_prefix`, `list_objects_with`, `list_all_objects` |
| Delete | `delete_object`, `delete_object_version` |
| Bucket | `create_bucket`, `delete_bucket`, `list_buckets` |
| Multipart | `create_multipart_upload`, `upload_part`, `complete_multipart_upload`, `abort_multipart_upload` |
| Batch | `delete_objects`, `delete_objects_with`, `put_objects`, `get_objects`, `get_objects_with` |

`list_all_objects` walks pagination for you; `object_exists` reduces a
`head_object` call to a bool.

## 5. Python — `mstore-sdk`

```python
from mstore import MStoreClient

with MStoreClient("localhost:9011") as client:
    client.create_bucket("documents")
    client.put_object("documents", "report.pdf", data,
                      content_type="application/pdf",
                      metadata={"project": "alpha"})
    obj = client.get_object("documents", "report.pdf")
    for item in client.list_all_objects("documents", prefix="2026/"):
        print(item.key, item.size)
```

| Method | Description |
|---|---|
| `MStoreClient(endpoint="localhost:9011", *, secure=False, max_message_size=16MiB)` | Connection; supports context-manager use |
| `put_object(bucket, key, data, *, content_type, storage_class, metadata)` | Streams automatically above 4 MiB |
| `get_object(bucket, key, ...)` | Reads the whole object into memory |
| `get_object_stream(bucket, key)` | Chunk-by-chunk read — for large objects |
| `head_object(bucket, key)` | Metadata without downloading |
| `delete_object(bucket, key)` | Delete / write a delete marker |
| `list_objects(bucket, ...)` | Single page |
| `list_all_objects(bucket, ...)` | Generator that walks every page |
| `create_bucket(bucket, *, region="")` · `delete_bucket(bucket, *, force=False)` · `list_buckets()` | Bucket operations |
| `upload_file(...)` · `download_file(...)` | File-path shortcuts |
| `close()` | Closes the channel |

Requirements: Python ≥ 3.9, `grpcio ≥ 1.60`, `protobuf ≥ 4.25`.

## 6. Go — `github.com/mstore/mstore-go`

```go
client, err := mstore.NewClient("127.0.0.1:9011")
if err != nil { return err }
defer client.Close()

ctx := context.Background()
if err := client.CreateBucket(ctx, "documents"); err != nil { return err }
out, err := client.PutObject(ctx, "documents", "report.pdf", data)
obj, err := client.GetObject(ctx, "documents", "report.pdf")
```

| Method | Signature summary |
|---|---|
| `NewClient(endpoint, opts ...ClientOptions)` | Connection |
| `Close()` | Shut down |
| `PutObject(ctx, bucket, key, data, opts ...PutOptions)` | `*PutObjectOutput` |
| `GetObject(ctx, bucket, key, opts ...GetOptions)` | `*GetObjectOutput` |
| `GetObjectStream(ctx, bucket, key)` | `io.Reader` + `*GetStreamMeta` |
| `HeadObject(ctx, bucket, key)` | `*HeadObjectOutput` |
| `DeleteObject(ctx, bucket, key)` | `*DeleteObjectOutput` |
| `ListObjects(ctx, bucket, opts ...ListOptions)` | `*ListObjectsOutput` |
| `CreateBucket(ctx, bucket)` · `DeleteBucket(ctx, bucket, force)` · `ListBuckets(ctx)` | Bucket operations |

Go ≥ 1.22, `google.golang.org/grpc v1.64`.

## 7. Java — `com.mstore:mstore-client`

```java
try (MStoreClient client = new MStoreClient("127.0.0.1", 9011)) {
    client.createBucket("documents");
    PutResult r = client.putObject("documents", "report.pdf", data);
    GetResult g = client.getObject("documents", "report.pdf");
    ListResult l = client.listObjects("documents", "2026/", "/", 1000);
}
```

| Method | Returns |
|---|---|
| `putObject(bucket, key, data)` / `(…, contentType, storageClass, metadata)` | `PutResult(versionId, etag, requestId)` |
| `getObject(bucket, key)` / `(…, versionId, rangeStart, rangeEnd)` | `GetResult(body, versionId, etag, …)` |
| `headObject(bucket, key)` | `HeadResult(versionId, etag, size, …)` |
| `deleteObject(bucket, key)` | `DeleteResult(deleteMarker, deleteMarkerId, requestId)` |
| `listObjects(bucket, prefix, delimiter, maxKeys)` | `ListResult(objects, commonPrefixes, …)` |
| `createBucket` · `deleteBucket(bucket, force)` · `listBuckets()` | — / `List<BucketItemResult>` |
| `uploadFile(bucket, key, Path)` · `downloadFile(bucket, key, Path)` | File shortcuts |
| `close()` | `AutoCloseable` |

Constants: `STREAMING_THRESHOLD = 4 MiB`, `CHUNK_SIZE = 4 MiB`.

## 8. .NET — `MStore.Client`

```csharp
using var client = new MStoreClient("http://127.0.0.1:9011");

await client.CreateBucketAsync("documents");
var put = await client.PutObjectAsync("documents", "report.pdf", data);
var get = await client.GetObjectAsync("documents", "report.pdf");
```

| Method | Returns |
|---|---|
| `PutObjectAsync(...)` | `PutObjectResult(VersionId, ETag, RequestId)` |
| `GetObjectAsync(...)` | `GetObjectResult` |
| `HeadObjectAsync(...)` | `HeadObjectResult` |
| `DeleteObjectAsync(...)` | `DeleteObjectResult` |
| `ListObjectsAsync(...)` | `ListObjectsResult` |
| `CreateBucketAsync` · `DeleteBucketAsync(bucket, force)` · `ListBucketsAsync()` | — / `List<BucketItemResult>` |
| `UploadFileAsync` · `DownloadFileAsync` | File shortcuts |
| `Dispose()` | `IDisposable` |

`net8.0`, `Grpc.Net.Client 2.63`. Constants: `StreamingThreshold`, `ChunkSize` = 4 MiB.

## 9. JavaScript / TypeScript — `@mstore/sdk`

```ts
import { MStoreClient } from '@mstore/sdk';

const client = new MStoreClient('127.0.0.1:9011');
await client.createBucket('documents');
await client.putObject('documents', 'report.pdf', buffer, { contentType: 'application/pdf' });
const obj = await client.getObject('documents', 'report.pdf');
client.close();
```

| Method | Returns |
|---|---|
| `putObject(bucket, key, data, opts?)` | `Promise<PutObjectOutput>` |
| `getObject(bucket, key, opts?)` | `Promise<GetObjectOutput>` |
| `headObject(bucket, key)` | `Promise<HeadObjectOutput>` |
| `deleteObject(bucket, key)` | `Promise<DeleteObjectOutput>` |
| `listObjects(bucket, opts?)` | `Promise<ListObjectsOutput>` |
| `createBucket(bucket, region?)` · `deleteBucket(bucket, force?)` · `listBuckets()` | `Promise<void>` / `Promise<BucketItem[]>` |
| `close()` | — |

`@grpc/grpc-js ^1.10`. Node.js only — in the browser use a
[presigned URL](s3-api.md#generating-a-presigned-url) instead of gRPC.

## 10. CLI — `mstore`

Extracted from the release archive; no separate installation needed:

```bash
mstore mb     mstore://documents
mstore cp     report.pdf mstore://documents/report.pdf
mstore ls     mstore://documents/
mstore head   mstore://documents/report.pdf
mstore rm     mstore://documents/report.pdf
mstore rb     mstore://documents
mstore search mstore://documents "contract"
```

## Related Documents

- [grpc-api.md](grpc-api.md) — raw gRPC contract and message fields
- [s3-api.md](s3-api.md) — S3 HTTP API and AWS SDK examples
- [installation.md](installation.md) — installation
- [mstore.proto](mstore.proto) — protobuf contract
