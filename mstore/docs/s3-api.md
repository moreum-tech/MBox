# MStore S3 API Referansı / S3 API Reference

> Sürüm / Version: **v0.4.0** · Varsayılan uç nokta / Default endpoint: `http://<host>:9010`

## İçindekiler / Table of Contents

- [Türkçe](#türkçe)
- [English](#english)

---

# Türkçe

MStore, S3 HTTP API'sini `9010` portunda yayınlar. AWS SDK'ları, AWS CLI,
`s3cmd`, `rclone`, Cyberduck ve diğer S3 uyumlu araçlar **hiçbir değişiklik
gerektirmeden** çalışır — tek yapmanız gereken uç noktayı MStore'a yönlendirmektir.

```bash
aws --endpoint-url http://127.0.0.1:9010 s3 ls
```

## 1. Kimlik Doğrulama

| Yöntem | Açıklama |
|---|---|
| **SigV4 (Authorization başlığı)** | Standart AWS imzalama. Tüm AWS SDK'ları varsayılan olarak bunu kullanır. |
| **SigV4 (presigned query-string)** | Süre sınırlı, imzalı URL. Tarayıcıdan doğrudan indirme/yükleme için. |
| **STS AssumeRole** | `POST /` — geçici kimlik bilgisi üretir. |
| **IMDS** | Instance Metadata Service uyumlu uç noktalar; AWS SDK'ları kimlik bilgisini otomatik keşfeder. |
| **LDAP / OIDC** | Kurumsal dizin veya OIDC sağlayıcı üzerinden kimlik doğrulama (yapılandırmaya bağlı). |

`[auth]` bölümünde `root_access_key` / `root_secret_key` boşsa **kimlik doğrulama
devre dışıdır** ve tüm istekler kabul edilir. Üretimde mutlaka doldurun.

### Presigned URL üretme

```bash
curl "http://127.0.0.1:9010/mstore/presign?bucket=my-bucket&key=dosya.pdf&method=GET&expires=3600"
```

| Parametre | Zorunlu | Varsayılan |
|---|---|---|
| `bucket` | Evet | — |
| `key` | Evet | — |
| `method` | Hayır | `GET` |
| `expires` | Hayır | `3600` (saniye) |

## 2. Bucket İşlemleri

| İşlem | HTTP |
|---|---|
| ListBuckets | `GET /` |
| CreateBucket | `PUT /{bucket}` |
| DeleteBucket | `DELETE /{bucket}` |
| HeadBucket | `HEAD /{bucket}` |
| ListObjects / ListObjectsV2 | `GET /{bucket}` |
| ListObjectVersions | `GET /{bucket}?versions` |
| DeleteObjects (toplu silme) | `POST /{bucket}?delete` |
| POST Object (form upload) | `POST /{bucket}` |

### Bucket alt kaynakları

Her biri `GET` (oku), `PUT` (yaz) ve çoğu `DELETE` (kaldır) destekler:

| Alt kaynak | Sorgu parametresi | GET | PUT | DELETE |
|---|---|:-:|:-:|:-:|
| Versioning | `?versioning` | ✓ | ✓ | |
| Tagging | `?tagging` | ✓ | ✓ | ✓ |
| Policy | `?policy` | ✓ | ✓ | ✓ |
| CORS | `?cors` | ✓ | ✓ | ✓ |
| Object Lock | `?object-lock` | ✓ | ✓ | |
| Encryption | `?encryption` | ✓ | ✓ | ✓ |
| Notification | `?notification` | ✓ | ✓ | |
| Replication | `?replication` | ✓ | ✓ | ✓ |
| Lifecycle | `?lifecycle` | ✓ | ✓ | ✓ |
| Website | `?website` | ✓ | ✓ | ✓ |
| ACL | `?acl` | ✓ | ✓ | |
| Logging | `?logging` | ✓ | ✓ | ✓ |
| Public Access Block | `?publicAccessBlock` | ✓ | ✓ | ✓ |
| Accelerate | `?accelerate` | ✓ | ✓ | |
| Inventory | `?inventory` | ✓ | ✓ | ✓ |

## 3. Nesne İşlemleri

| İşlem | HTTP |
|---|---|
| PutObject | `PUT /{bucket}/{key}` |
| GetObject | `GET /{bucket}/{key}` |
| HeadObject | `HEAD /{bucket}/{key}` |
| DeleteObject | `DELETE /{bucket}/{key}` |
| CopyObject | `PUT /{bucket}/{key}` + `x-amz-copy-source` |
| GetObject (byte range) | `GET` + `Range: bytes=0-1023` |
| GetObject (sürüm) | `GET /{bucket}/{key}?versionId=...` |

### Nesne alt kaynakları

| Alt kaynak | Sorgu parametresi | GET | PUT | DELETE |
|---|---|:-:|:-:|:-:|
| Tagging | `?tagging` | ✓ | ✓ | ✓ |
| Retention | `?retention` | ✓ | ✓ | |
| Legal Hold | `?legal-hold` | ✓ | ✓ | |
| ACL | `?acl` | ✓ | | |
| Restore | `?restore` (POST) | | | |
| S3 Select | `?select` (POST) | | | |

### Multipart Upload

| İşlem | HTTP |
|---|---|
| CreateMultipartUpload | `POST /{bucket}/{key}?uploads` |
| UploadPart | `PUT /{bucket}/{key}?uploadId=...&partNumber=N` |
| UploadPartCopy | `PUT` + `x-amz-copy-source` + `uploadId`/`partNumber` |
| ListParts | `GET /{bucket}/{key}?uploadId=...` |
| CompleteMultipartUpload | `POST /{bucket}/{key}?uploadId=...` |
| AbortMultipartUpload | `DELETE /{bucket}/{key}?uploadId=...` |

Tamamlanmamış yüklemeler `storage.multipart_expiry_hours` (varsayılan 168 saat)
sonunda otomatik temizlenir.

### Desteklenen başlıklar

| Başlık | Amaç |
|---|---|
| `x-amz-meta-*` | Kullanıcı tanımlı metadata (arama indeksine de girer) |
| `x-amz-storage-class` | `STANDARD`, `REDUCED_REDUNDANCY`, `GLACIER`, `DEEP_ARCHIVE`, `INTELLIGENT_TIERING` |
| `x-amz-server-side-encryption` | SSE-S3 / SSE-KMS |
| `x-amz-server-side-encryption-customer-key` | SSE-C (istek başına anahtar) |
| `x-amz-copy-source` | CopyObject / UploadPartCopy kaynağı |
| `Range` | Kısmi okuma |
| `If-Match`, `If-None-Match`, `If-Modified-Since` | Koşullu istekler |

### Yanıt başlıkları

| Başlık | Açıklama |
|---|---|
| `ETag` | BLAKE3 tabanlı içerik özeti (MD5'ten 4–5× hızlı) |
| `x-amz-version-id` | Sürümleme açıksa nesne sürüm kimliği |
| `x-amz-request-id` | İstek izleme kimliği |
| `x-mstore-timing` | **MStore'a özgü:** GET/PUT için faz bazlı mikrosaniye dökümü |

`x-mstore-timing` başlığı bir isteğin nerede zaman harcadığını (metadata okuma,
disk I/O, erasure decode, şifre çözme) mikrosaniye çözünürlüğünde gösterir —
performans sorunlarını ayıklarken tek başına kullanılabilir.

## 4. MStore'a Özgü API'ler

Bunlar S3 standardının dışındadır; aynı `9010` portundan sunulur.

### Sağlık ve metrikler

| Uç nokta | Açıklama |
|---|---|
| `GET /minio/health/live` | Süreç ayakta mı (kimlik doğrulama gerekmez) |
| `GET /minio/health/ready` | İstek almaya hazır mı |
| `GET /minio/health/cluster` | Cluster yeter sayısı sağlanıyor mu |
| `GET /minio/v2/metrics/cluster` | Prometheus formatında metrikler |
| `GET /minio/admin/v3/info` | Sunucu bilgisi (MinIO admin uyumlu) |

> MinIO yollarının korunması bilinçlidir: mevcut MinIO izleme/otomasyon
> araçlarınız MStore ile değişiklik yapmadan çalışır.

### Tam metin arama

Her bucket için Tantivy indeksi tutulur ve yazma olaylarıyla güncellenir.

```bash
# Tek bucket içinde
curl -X POST http://127.0.0.1:9010/mstore/search/my-bucket \
  -H 'Content-Type: application/json' \
  -d '{"fulltext":"sözleşme","content_type":"application/pdf","limit":50}'

# Tüm bucketlarda
curl -X POST http://127.0.0.1:9010/mstore/search \
  -H 'Content-Type: application/json' \
  -d '{"key_prefix":"2026/01/","size_min":1048576}'
```

İstek gövdesi alanları:

| Alan | Tip | Açıklama |
|---|---|---|
| `bucket` | string | Tam bucket adı (URL'de verilirse o kazanır) |
| `key` | string | Tam anahtar eşleşmesi |
| `key_prefix` | string | Anahtar ön eki |
| `content_type` | string | Tam eşleşme (ör. `application/pdf`) |
| `storage_class` | string | Depolama sınıfı filtresi |
| `size_min` / `size_max` | number | Bayt cinsinden boyut aralığı |
| `created_after` / `created_before` | string | ISO 8601 tarih |
| `metadata` | object | `x-amz-meta-*` çiftleri (AND mantığı) |
| `fulltext` | string | Metadata değerlerinde tam metin araması |
| `exclude_delete_markers` | bool | Varsayılan `true` |
| `limit` | number | Varsayılan 100, en fazla 10000 |

Yanıt:

```json
{
  "hits": [
    {
      "bucket": "my-bucket", "key": "2026/01/rapor.pdf",
      "version_id": "…", "etag": "…", "size": 284133,
      "content_type": "application/pdf", "storage_class": "STANDARD",
      "created_at": 1767225600, "metadata": {"proje": "alpha"},
      "is_delete_marker": false, "score": 3.41, "excerpt": "…"
    }
  ],
  "total_matches": 27,
  "count": 1
}
```

### Yönetim API'si (`/mstore/api/v1`)

| Uç nokta | Açıklama |
|---|---|
| `POST /login` | Konsol oturumu açar |
| `GET /info` | Sunucu ve küme bilgisi |
| `GET /config` | Etkin yapılandırma |
| `GET /buckets` | Bucket listesi (kotalar ve sayaçlarla) |
| `GET /version` | Sürüm bilgisi |
| `GET /storage/info` | Disk kullanımı, erasure düzeni |
| `GET/PUT /storage/buckets/{bucket}/quota` | Bucket kotası |
| `GET /storage/buckets/{bucket}/count` | Bucket içindeki nesne sayısı |
| `GET /storage/lens` · `/storage-lens` · `/storage-lens/{id}` | Depolama analitiği |
| `GET /heal/status` · `POST /heal/start` | Arka plan bütünlük onarımı |
| `POST /batch` · `GET /batch/jobs` · `GET /batch/jobs/{id}` | Toplu işlem işleri |
| `POST /orphans/scan` · `POST /orphans/purge` | Öksüz veri taraması/temizliği |
| `GET/POST/DELETE /iam/users`, `/iam/groups`, `/iam/policies`, `/iam/service-accounts` | IAM yönetimi |
| `POST /iam/sync` | LDAP/OIDC eşitleme |
| `GET/POST/DELETE /tenants`, `/tenants/{id}` | Çok kiracılı yönetim |
| `GET/POST/DELETE /accesspoints`, `/accesspoints/{name}` | Erişim noktaları |
| `GET /tiers` · `GET /tiers/status` | Katmanlama durumu |
| `POST /service/restart` · `POST /service/stop` | Servis kontrolü |

Öksüz veri geri kazanımının ayrıntıları: [orphan-reclamation.md](orphan-reclamation.md)

### Web Konsol

`GET /mstore/console` — tarayıcı tabanlı yönetim arayüzü (Svelte).

## 5. Hata Yanıtları

Hatalar S3'ün XML biçimindedir:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>NoSuchKey</Code>
  <Message>The specified key does not exist.</Message>
  <Resource>/my-bucket/eksik.txt</Resource>
  <RequestId>…</RequestId>
</Error>
```

Sık karşılaşılan kodlar: `NoSuchBucket`, `NoSuchKey`, `BucketAlreadyExists`,
`BucketNotEmpty`, `AccessDenied`, `SignatureDoesNotMatch`, `InvalidRequest`,
`EntityTooLarge`, `SlowDown` (hız sınırı), `InternalError`.

## 6. Sınırlar

| Sınır | Değer | Yapılandırma |
|---|---|---|
| Tek PUT gövdesi | 5 GiB | HTTP gövde sınırı |
| Maksimum nesne boyutu | 5 TiB (multipart ile) | `api.max_object_size` |
| Eşzamanlı istek | 4096 | `api.max_concurrent_requests` |
| İstek zaman aşımı | 300 sn | `api.request_timeout_secs` |
| Arama sonucu | 10000 | İstek başına `limit` |

Eşzamanlılık sınırı aşıldığında `503 SlowDown` döner.

## 7. AWS SDK Örnekleri

### Python (boto3)

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://127.0.0.1:9010",
    aws_access_key_id="ACCESS",
    aws_secret_access_key="SECRET",
    region_name="us-east-1",
)

s3.create_bucket(Bucket="belgeler")
s3.put_object(Bucket="belgeler", Key="rapor.pdf", Body=open("rapor.pdf", "rb"),
              ContentType="application/pdf", Metadata={"proje": "alpha"})
print(s3.list_objects_v2(Bucket="belgeler")["Contents"])
```

### C# (AWSSDK.S3)

```csharp
var config = new AmazonS3Config { ServiceURL = "http://127.0.0.1:9010", ForcePathStyle = true };
var s3 = new AmazonS3Client("ACCESS", "SECRET", config);

await s3.PutBucketAsync("belgeler");
await s3.PutObjectAsync(new PutObjectRequest {
    BucketName = "belgeler", Key = "rapor.pdf", FilePath = "rapor.pdf"
});
```

### Java (AWS SDK v2)

```java
S3Client s3 = S3Client.builder()
    .endpointOverride(URI.create("http://127.0.0.1:9010"))
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create("ACCESS", "SECRET")))
    .region(Region.US_EAST_1)
    .forcePathStyle(true)
    .build();

s3.createBucket(b -> b.bucket("belgeler"));
s3.putObject(b -> b.bucket("belgeler").key("rapor.pdf"), Path.of("rapor.pdf"));
```

### rclone

```ini
# ~/.config/rclone/rclone.conf
[mstore]
type = s3
provider = Other
endpoint = http://127.0.0.1:9010
access_key_id = ACCESS
secret_access_key = SECRET
force_path_style = true
```

```bash
rclone copy ./klasor mstore:belgeler/
```

## İlgili Belgeler

- [grpc-api.md](grpc-api.md) — Yüksek performanslı gRPC arayüzü
- [sdk-reference.md](sdk-reference.md) — Yerel SDK'lar (6 dil)
- [installation.md](installation.md) — Kurulum
- [configuration.md](configuration.md) — Yapılandırma

---

# English

MStore serves the S3 HTTP API on port `9010`. AWS SDKs, the AWS CLI, `s3cmd`,
`rclone`, Cyberduck and other S3-compatible tools work **without any
modification** — all you change is the endpoint.

```bash
aws --endpoint-url http://127.0.0.1:9010 s3 ls
```

## 1. Authentication

| Method | Description |
|---|---|
| **SigV4 (Authorization header)** | Standard AWS signing. Every AWS SDK uses this by default. |
| **SigV4 (presigned query string)** | Time-limited signed URL for direct browser upload/download. |
| **STS AssumeRole** | `POST /` — issues temporary credentials. |
| **IMDS** | Instance Metadata Service–compatible endpoints; AWS SDKs discover credentials automatically. |
| **LDAP / OIDC** | Enterprise directory or OIDC provider authentication (configuration dependent). |

If `root_access_key` / `root_secret_key` are empty in `[auth]`, **authentication
is disabled** and every request is accepted. Always set them in production.

### Generating a presigned URL

```bash
curl "http://127.0.0.1:9010/mstore/presign?bucket=my-bucket&key=file.pdf&method=GET&expires=3600"
```

| Parameter | Required | Default |
|---|---|---|
| `bucket` | Yes | — |
| `key` | Yes | — |
| `method` | No | `GET` |
| `expires` | No | `3600` (seconds) |

## 2. Bucket Operations

| Operation | HTTP |
|---|---|
| ListBuckets | `GET /` |
| CreateBucket | `PUT /{bucket}` |
| DeleteBucket | `DELETE /{bucket}` |
| HeadBucket | `HEAD /{bucket}` |
| ListObjects / ListObjectsV2 | `GET /{bucket}` |
| ListObjectVersions | `GET /{bucket}?versions` |
| DeleteObjects (bulk) | `POST /{bucket}?delete` |
| POST Object (form upload) | `POST /{bucket}` |

### Bucket sub-resources

Each supports `GET` (read), `PUT` (write) and most support `DELETE` (remove):

| Sub-resource | Query parameter | GET | PUT | DELETE |
|---|---|:-:|:-:|:-:|
| Versioning | `?versioning` | ✓ | ✓ | |
| Tagging | `?tagging` | ✓ | ✓ | ✓ |
| Policy | `?policy` | ✓ | ✓ | ✓ |
| CORS | `?cors` | ✓ | ✓ | ✓ |
| Object Lock | `?object-lock` | ✓ | ✓ | |
| Encryption | `?encryption` | ✓ | ✓ | ✓ |
| Notification | `?notification` | ✓ | ✓ | |
| Replication | `?replication` | ✓ | ✓ | ✓ |
| Lifecycle | `?lifecycle` | ✓ | ✓ | ✓ |
| Website | `?website` | ✓ | ✓ | ✓ |
| ACL | `?acl` | ✓ | ✓ | |
| Logging | `?logging` | ✓ | ✓ | ✓ |
| Public Access Block | `?publicAccessBlock` | ✓ | ✓ | ✓ |
| Accelerate | `?accelerate` | ✓ | ✓ | |
| Inventory | `?inventory` | ✓ | ✓ | ✓ |

## 3. Object Operations

| Operation | HTTP |
|---|---|
| PutObject | `PUT /{bucket}/{key}` |
| GetObject | `GET /{bucket}/{key}` |
| HeadObject | `HEAD /{bucket}/{key}` |
| DeleteObject | `DELETE /{bucket}/{key}` |
| CopyObject | `PUT /{bucket}/{key}` + `x-amz-copy-source` |
| GetObject (byte range) | `GET` + `Range: bytes=0-1023` |
| GetObject (version) | `GET /{bucket}/{key}?versionId=...` |

### Object sub-resources

| Sub-resource | Query parameter | GET | PUT | DELETE |
|---|---|:-:|:-:|:-:|
| Tagging | `?tagging` | ✓ | ✓ | ✓ |
| Retention | `?retention` | ✓ | ✓ | |
| Legal Hold | `?legal-hold` | ✓ | ✓ | |
| ACL | `?acl` | ✓ | | |
| Restore | `?restore` (POST) | | | |
| S3 Select | `?select` (POST) | | | |

### Multipart upload

| Operation | HTTP |
|---|---|
| CreateMultipartUpload | `POST /{bucket}/{key}?uploads` |
| UploadPart | `PUT /{bucket}/{key}?uploadId=...&partNumber=N` |
| UploadPartCopy | `PUT` + `x-amz-copy-source` + `uploadId`/`partNumber` |
| ListParts | `GET /{bucket}/{key}?uploadId=...` |
| CompleteMultipartUpload | `POST /{bucket}/{key}?uploadId=...` |
| AbortMultipartUpload | `DELETE /{bucket}/{key}?uploadId=...` |

Incomplete uploads are cleaned up automatically after
`storage.multipart_expiry_hours` (default 168 hours).

### Supported headers

| Header | Purpose |
|---|---|
| `x-amz-meta-*` | User-defined metadata (also fed into the search index) |
| `x-amz-storage-class` | `STANDARD`, `REDUCED_REDUNDANCY`, `GLACIER`, `DEEP_ARCHIVE`, `INTELLIGENT_TIERING` |
| `x-amz-server-side-encryption` | SSE-S3 / SSE-KMS |
| `x-amz-server-side-encryption-customer-key` | SSE-C (per-request key) |
| `x-amz-copy-source` | CopyObject / UploadPartCopy source |
| `Range` | Partial read |
| `If-Match`, `If-None-Match`, `If-Modified-Since` | Conditional requests |

### Response headers

| Header | Description |
|---|---|
| `ETag` | BLAKE3-based content digest (4–5× faster than MD5) |
| `x-amz-version-id` | Object version id when versioning is enabled |
| `x-amz-request-id` | Request tracing id |
| `x-mstore-timing` | **MStore-specific:** per-phase microsecond breakdown for GET/PUT |

`x-mstore-timing` shows where a request spent its time (metadata read, disk I/O,
erasure decode, decryption) at microsecond resolution — enough on its own to
diagnose most latency problems.

## 4. MStore-Specific APIs

These are outside the S3 standard and are served on the same port `9010`.

### Health and metrics

| Endpoint | Description |
|---|---|
| `GET /minio/health/live` | Process liveness (no authentication) |
| `GET /minio/health/ready` | Ready to serve requests |
| `GET /minio/health/cluster` | Cluster quorum status |
| `GET /minio/v2/metrics/cluster` | Prometheus-format metrics |
| `GET /minio/admin/v3/info` | Server info (MinIO admin compatible) |

> Keeping the MinIO paths is deliberate: your existing MinIO monitoring and
> automation works against MStore unchanged.

### Full-text search

Each bucket has a Tantivy index kept current by write events.

```bash
# Within a single bucket
curl -X POST http://127.0.0.1:9010/mstore/search/my-bucket \
  -H 'Content-Type: application/json' \
  -d '{"fulltext":"contract","content_type":"application/pdf","limit":50}'

# Across every bucket
curl -X POST http://127.0.0.1:9010/mstore/search \
  -H 'Content-Type: application/json' \
  -d '{"key_prefix":"2026/01/","size_min":1048576}'
```

Request body fields:

| Field | Type | Description |
|---|---|---|
| `bucket` | string | Exact bucket name (the URL path wins if given) |
| `key` | string | Exact key match |
| `key_prefix` | string | Key prefix |
| `content_type` | string | Exact match (e.g. `application/pdf`) |
| `storage_class` | string | Storage class filter |
| `size_min` / `size_max` | number | Size range in bytes |
| `created_after` / `created_before` | string | ISO 8601 date |
| `metadata` | object | `x-amz-meta-*` pairs (AND logic) |
| `fulltext` | string | Full-text search across metadata values |
| `exclude_delete_markers` | bool | Default `true` |
| `limit` | number | Default 100, max 10000 |

Response:

```json
{
  "hits": [
    {
      "bucket": "my-bucket", "key": "2026/01/report.pdf",
      "version_id": "…", "etag": "…", "size": 284133,
      "content_type": "application/pdf", "storage_class": "STANDARD",
      "created_at": 1767225600, "metadata": {"project": "alpha"},
      "is_delete_marker": false, "score": 3.41, "excerpt": "…"
    }
  ],
  "total_matches": 27,
  "count": 1
}
```

### Admin API (`/mstore/api/v1`)

| Endpoint | Description |
|---|---|
| `POST /login` | Open a console session |
| `GET /info` | Server and cluster info |
| `GET /config` | Effective configuration |
| `GET /buckets` | Bucket list with quotas and counts |
| `GET /version` | Version info |
| `GET /storage/info` | Disk usage, erasure layout |
| `GET/PUT /storage/buckets/{bucket}/quota` | Bucket quota |
| `GET /storage/buckets/{bucket}/count` | Object count in a bucket |
| `GET /storage/lens` · `/storage-lens` · `/storage-lens/{id}` | Storage analytics |
| `GET /heal/status` · `POST /heal/start` | Background integrity repair |
| `POST /batch` · `GET /batch/jobs` · `GET /batch/jobs/{id}` | Batch operation jobs |
| `POST /orphans/scan` · `POST /orphans/purge` | Orphan data scan/purge |
| `GET/POST/DELETE /iam/users`, `/iam/groups`, `/iam/policies`, `/iam/service-accounts` | IAM management |
| `POST /iam/sync` | LDAP/OIDC synchronization |
| `GET/POST/DELETE /tenants`, `/tenants/{id}` | Multi-tenant management |
| `GET/POST/DELETE /accesspoints`, `/accesspoints/{name}` | Access points |
| `GET /tiers` · `GET /tiers/status` | Tiering status |
| `POST /service/restart` · `POST /service/stop` | Service control |

Orphan reclamation details: [orphan-reclamation.md](orphan-reclamation.md)

### Web console

`GET /mstore/console` — browser-based management UI (Svelte).

## 5. Error Responses

Errors use the S3 XML shape:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>NoSuchKey</Code>
  <Message>The specified key does not exist.</Message>
  <Resource>/my-bucket/missing.txt</Resource>
  <RequestId>…</RequestId>
</Error>
```

Common codes: `NoSuchBucket`, `NoSuchKey`, `BucketAlreadyExists`,
`BucketNotEmpty`, `AccessDenied`, `SignatureDoesNotMatch`, `InvalidRequest`,
`EntityTooLarge`, `SlowDown` (rate limited), `InternalError`.

## 6. Limits

| Limit | Value | Configuration |
|---|---|---|
| Single PUT body | 5 GiB | HTTP body limit |
| Maximum object size | 5 TiB (via multipart) | `api.max_object_size` |
| Concurrent requests | 4096 | `api.max_concurrent_requests` |
| Request timeout | 300 s | `api.request_timeout_secs` |
| Search results | 10000 | `limit` per request |

Exceeding the concurrency limit returns `503 SlowDown`.

## 7. AWS SDK Examples

### Python (boto3)

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://127.0.0.1:9010",
    aws_access_key_id="ACCESS",
    aws_secret_access_key="SECRET",
    region_name="us-east-1",
)

s3.create_bucket(Bucket="documents")
s3.put_object(Bucket="documents", Key="report.pdf", Body=open("report.pdf", "rb"),
              ContentType="application/pdf", Metadata={"project": "alpha"})
print(s3.list_objects_v2(Bucket="documents")["Contents"])
```

### C# (AWSSDK.S3)

```csharp
var config = new AmazonS3Config { ServiceURL = "http://127.0.0.1:9010", ForcePathStyle = true };
var s3 = new AmazonS3Client("ACCESS", "SECRET", config);

await s3.PutBucketAsync("documents");
await s3.PutObjectAsync(new PutObjectRequest {
    BucketName = "documents", Key = "report.pdf", FilePath = "report.pdf"
});
```

### Java (AWS SDK v2)

```java
S3Client s3 = S3Client.builder()
    .endpointOverride(URI.create("http://127.0.0.1:9010"))
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create("ACCESS", "SECRET")))
    .region(Region.US_EAST_1)
    .forcePathStyle(true)
    .build();

s3.createBucket(b -> b.bucket("documents"));
s3.putObject(b -> b.bucket("documents").key("report.pdf"), Path.of("report.pdf"));
```

### rclone

```ini
# ~/.config/rclone/rclone.conf
[mstore]
type = s3
provider = Other
endpoint = http://127.0.0.1:9010
access_key_id = ACCESS
secret_access_key = SECRET
force_path_style = true
```

```bash
rclone copy ./folder mstore:documents/
```

## Related Documents

- [grpc-api.md](grpc-api.md) — high-performance gRPC interface
- [sdk-reference.md](sdk-reference.md) — native SDKs (6 languages)
- [installation.md](installation.md) — installation
- [configuration.md](configuration.md) — configuration
