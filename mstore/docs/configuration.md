# MStore Yapılandırma Referansı / Configuration Reference

> Sürüm / Version: **v0.3.1**

## İçindekiler / Table of Contents

- [Türkçe](#türkçe)
- [English](#english)

---

# Türkçe

## 1. İki Çalışma Modu

MStore iki şekilde başlatılabilir:

**a) Sıfır yapılandırma (zero-config)** — sürücü yollarını doğrudan verin,
yapılandırma dosyası gerekmez:

```bash
mstore-server /data                        # tek sürücü
mstore-server /data1 /data2 /data3 /data4  # 4 sürücü, erasure coding
```

**b) Yapılandırma dosyası** — üretim için önerilen yol:

```bash
mstore-server --config /etc/mstore/config.toml
```

> Konumsal sürücü yolları verildiğinde `--config` **yok sayılır**.

## 2. Komut Satırı Seçenekleri

| Seçenek | Ortam değişkeni | Varsayılan | Açıklama |
|---|---|---|---|
| `[PATH]...` | — | — | Sıfır yapılandırma modunda sürücü yolları |
| `--config <PATH>` | `MSTORE_CONFIG` | — | TOML yapılandırma dosyası |
| `--api-bind <ADDR>` | `MSTORE_API_BIND` | `api.bind` | S3 HTTP API adresini geçersiz kılar |
| `--grpc-bind <ADDR>` | `MSTORE_GRPC_BIND` | `0.0.0.0:9011` | gRPC adresini geçersiz kılar |
| `--console-path <PATH>` | `MSTORE_CONSOLE_PATH` | — | Konsol statik dosya dizini |

## 3. Ortam Değişkenleri

| Değişken | Amaç |
|---|---|
| `MSTORE_CONFIG` | Yapılandırma dosyası yolu |
| `MSTORE_API_BIND` | S3 API bind adresi |
| `MSTORE_GRPC_BIND` | gRPC bind adresi |
| `MSTORE_CONSOLE_PATH` | Konsol statik dosya dizini |
| `MSTORE_ROOT_USER` | Root erişim anahtarı (config'i geçersiz kılar) |
| `MSTORE_ROOT_PASSWORD` | Root gizli anahtarı (config'i geçersiz kılar) |
| `MSTORE_MASTER_KEY` | SSE-S3 sunucu tarafı şifreleme ana anahtarı |
| `RUST_LOG` | Günlük seviyesi — örn. `info`, `mstore_server=debug` |

> Kimlik bilgilerini üretimde TOML dosyasına düz metin yazmak yerine
> `MSTORE_ROOT_USER` / `MSTORE_ROOT_PASSWORD` ile verin.

## 4. Minimum Yapılandırma

```toml
[node]
name    = "mstore-1"
address = "127.0.0.1:9010"

[storage]
drives = [{ path = "/data/drive1" }]

[erasure]
data_shards   = 1
parity_shards = 0

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "degistirin"
```

## 5. Bölümler

### `[node]` — Düğüm kimliği

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `name` | string | *(zorunlu)* | Günlüklerde ve admin API'de görünen ad |
| `address` | string | *(zorunlu)* | Eşler arası iletişim için ilan edilen adres |
| `peers` | string[] | `[]` | Dağıtık kurulumda diğer düğümler |
| `pg_count` | u32 | `0` | Yerleşim grubu sayısı; `0` = tek düğüm modu |
| `weight` | u32 | `1` | Rendezvous hashing ağırlığı (kapasiteyle orantılı) |

Çok düğümlü kurulum: [multinode-setup.md](multinode-setup.md)

### `[storage]` — Sürücüler ve metadata motoru

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `drives` | tablo[] | *(zorunlu)* | Sürücü listesi (aşağıya bakın) |
| `direct_io` | bool | `true` | Büyük nesne G/Ç'sinde işletim sistemi önbelleğini atla. NFS/SMB üzerinde `false` yapın |
| `sync_on_write` | bool | `true` | Her yazmadan sonra `fdatasync`. **Üretimde asla `false` yapmayın** |
| `verify_on_read` | enum | `always` | Her GET'te BLAKE3 doğrula: `always` \| `native_only` (ZFS/Btrfs'e güven) \| `never` |
| `inline_threshold` | bayt | `4096` | Bu boyutun altındaki nesneler ayrı shard dosyası yerine `xl.meta` içine gömülür |
| `path_hash_depth` | u8 | `0` | Sıcak dizinleri önlemek için yol hash derinliği. 10M+ nesne beklenen bucket'larda `2` |
| `group_commit_batch` | sayı | `64` | Tek `fdatasync` öncesi biriktirilecek `xl.meta` yazması |
| `group_commit_interval_ms` | ms | `5` | Batch dolmasa da zorlanan commit aralığı |
| `meta_cache_bytes` | bayt | `0` (kapalı) | `xl.meta` LRU önbelleği; RocksDB blok önbelleği genelde yeterli |
| `bloom_filter_capacity` | sayı | `10000000` | Beklenen farklı nesne anahtarı sayısı |
| `meta_shard_count` | sayı | `0` (oto) | Metadata shard sayısı; `0` = sürücü × 4 |
| `rocksdb_shard_threshold` | sayı | `50000000` | Bir shard bu anahtar sayısını aşınca yeni shard açılır; `0` = kapalı |
| `rocksdb_initial_shards` | sayı | `0` | İlk açılışta ön-bölme; yük altında yazma duraklamalarını önler |
| `rocksdb_compression` | string | `zstd` | L2+ seviyeleri için: `none` \| `snappy` \| `zstd` |
| `blob_min_size` | bayt | `256` | Bu boyutun üstündeki RocksDB değerleri blob dosyalarına taşınır; `0` = BlobDB kapalı |
| `bloom_checkpoint_interval_secs` | sn | `300` | Hızlı açılış için bloom filtresi kontrol noktası aralığı |
| `multipart_expiry_hours` | saat | `168` | Tamamlanmamış multipart yüklemelerin temizlenme yaşı (7 gün) |

**`drives` girdileri:**

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `path` | yol | *(zorunlu)* | Bağlama noktası |
| `metadata_only` | bool | `false` | Yalnızca `xl.meta` sakla — NAS önünde hızlı NVMe metadata sürücüsü için |
| `storage_class` | string | `STANDARD` | Yaşam döngüsü katmanlaması için sınıf; örn. NVMe=`STANDARD`, HDD=`STANDARD_IA` |

Çok sürücülü kurulum: [multidrive-setup.md](multidrive-setup.md)

### `[erasure]` — Silme kodlaması

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `data_shards` | u8 | *(zorunlu)* | Veri shard'ı (K). Toplam sürücü = K + M |
| `parity_shards` | u8 | *(zorunlu)* | Parite shard'ı (M) = tolere edilen sürücü arızası |
| `block_size` | bayt | `1048576` | Reed-Solomon blok boyutu (1 MiB) |

Kurallar:

- `data_shards >= 1`
- `data_shards + parity_shards <= 32`
- **Okuma çekirdeği** = `data_shards`
- **Yazma çekirdeği** = `data_shards`, ancak `parity_shards >= data_shards` ise
  split-brain'i önlemek için `data_shards + 1`

### `[api]` — S3 HTTP API

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `bind` | string | `0.0.0.0:9010` | Dinleme adresi |
| `max_object_size` | bayt | 5 TiB | Tek PUT ile kabul edilen en büyük nesne |
| `request_timeout_secs` | sn | `300` | İstek zaman aşımı |
| `write_concurrency_per_cpu` | sayı | `30` | Çekirdek başına eşzamanlı yazma (PUT/DELETE/POST) |
| `read_concurrency_per_cpu` | sayı | `150` | Çekirdek başına eşzamanlı okuma (GET/HEAD) |
| `max_write_concurrency` | sayı | `0` (sınırsız) | Yazma için mutlak tavan |
| `max_read_concurrency` | sayı | `0` (sınırsız) | Okuma için mutlak tavan |
| `max_total_requests` | sayı | `10000` | Uçuştaki toplam istek tavanı; aşılırsa `503` |

Etkin limit = `min(çekirdek_başına × CPU_sayısı, max_*)`.
`max_concurrent_requests` eski bir alandır; okunur ama yok sayılır.

### `[auth]` — Kimlik doğrulama

| Alan | Tip | Açıklama |
|---|---|---|
| `root_access_key` | string | Root erişim anahtarı |
| `root_secret_key` | string | Root gizli anahtarı — üretimde ortam değişkeni/vault kullanın |
| `[auth.ldap]` | tablo | Opsiyonel LDAP entegrasyonu |
| `[auth.oidc]` | tablo | Opsiyonel OIDC sağlayıcı |

**`[auth.ldap]`:** `server_url`, `bind_dn`, `bind_password`,
`user_search_base`, `user_search_filter`, `group_search_base` *(ops.)*

**`[auth.oidc]`:** `issuer_url`, `client_id`, `client_secret` *(ops.)*,
`role_claim` *(varsayılan `role`)*

### `[compliance]` — Uyumluluk ve denetim

| Alan | Tip | Varsayılan | Açıklama |
|---|---|---|---|
| `worm_compliance` | bool | `false` | SEC 17a-4 / FINRA 4511 / CFTC 1.31 WORM modu — COMPLIANCE kilidi kimse tarafından kaldırılamaz |
| `audit_retention_days` | u32 | `0` (süresiz) | Denetim WAL saklama süresi |
| `audit_bucket` | string | `__audit__` | Denetim kayıtları ve imha sertifikaları bucket'ı |
| `audit_retention_years` | u32 | `10` | Denetim kaydı saklama süresi (yıl) |
| `destruction_certificate` | bool | `false` | Silme işlemlerinde imzalı imha sertifikası üret |
| `digimr_endpoint` | string | — | DigiMR e-imza API'si; ayarlıysa imha sertifikaları dijital imzalanır |
| `signature_verification` | string | `off` | Yüklenen nesnelerde imza doğrulama: `off` \| `warn` \| `enforce` |
| `kvkk_secure_delete` | bool | `false` | `x-amz-meta-kvkk-secure-delete: true` etiketli nesnelerde 3 geçişli güvenli silme |
| `[compliance.audit_kafka]` | tablo | — | Denetim olaylarını yerel WAL'a ek olarak Kafka'ya yaz |

### `[replication]` — Siteler arası çoğaltma *(opsiyonel)*

| Alan | Açıklama |
|---|---|
| `site_id` | Bu site'ın benzersiz kimliği — döngü önleme ve çakışma çözümü |
| `source_endpoint` | Çoğaltma GET istekleri için yerel S3 uç noktası |
| `targets` | Hedef site listesi |
| `conflict_resolution` | Çakışma çözüm stratejisi |
| `[replication.retry]` | `initial_backoff_ms` (1000), `max_backoff_ms` (60000), `max_retries` (`0` = sonsuz) |

### `[lambda]` — Olay bildirimleri *(opsiyonel)*

| Alan | Açıklama |
|---|---|
| `targets` | Webhook / Kafka / NATS vb. hedefler |
| `sqs_endpoint` | SQS uyumlu uç nokta (ARN yönlendirmesi) |
| `sns_endpoint` | SNS uyumlu uç nokta |
| `eventbridge_endpoint` | EventBridge uyumlu uç nokta |

### `[tls]` — HTTPS *(opsiyonel)*

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `cert_path` | *(zorunlu)* | Sertifika dosyası (PEM) |
| `key_path` | *(zorunlu)* | Özel anahtar (PEM) |
| `min_version` | `1.2` | Asgari TLS sürümü |

### `[kms]` — SSE-KMS *(opsiyonel)*

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `endpoint` | *(zorunlu)* | KMS uç noktası |
| `access_key` / `secret_key` | *(zorunlu)* | KMS kimlik bilgileri |
| `default_key_id` | *(zorunlu)* | SSE-KMS için varsayılan ana anahtar |
| `dek_cache_ttl_secs` | `300` | Veri şifreleme anahtarı önbellek ömrü |

SSE-S3 için KMS gerekmez; `MSTORE_MASTER_KEY` yeterlidir.

### `[sftp]` — SFTP arayüzü *(opsiyonel)*

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `bind` | `0.0.0.0:2022` | SFTP dinleme adresi |
| `host_key_path` | *(zorunlu)* | SSH host anahtarı |

### `[lifecycle]` — Yaşam döngüsü kuralları *(opsiyonel)*

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `scan_interval_secs` | `3600` | Kural değerlendirme tarama aralığı |
| `max_objects_per_sec` | `10000` | Throttle; `0` = sınırsız (önerilmez) |
| `off_peak_window` | — | Yalnızca bu aralıkta tara — `HH:MM-HH:MM` (yerel saat) |

### `[reclaim]` — Öksüz veri geri kazanımı *(opsiyonel)*

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `tick_secs` | `30` | Worker uyanma aralığı (olay veri yolu da uyandırır) |
| `max_files_per_sec` | `10000` | Saniyede en fazla silinen dosya; `0` = sınırsız |
| `batch_size` | `1000` | Bir turda işlenen pending satır sayısı |
| `deep_scan_min_age_secs` | `86400` | Bu yaştan genç metadatasız dosya öksüz sayılmaz |

Ayrıntı: [orphan-reclamation.md](orphan-reclamation.md)

### `[index]` — Tam metin arama

Metadata indeksleme (anahtar, boyut, içerik tipi, tarih) **her zaman açıktır**.
Bu bölüm yalnızca içerik çıkarımını kontrol eder.

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `content_extraction` | `false` | PDF, DOCX, XLSX ve metin dosyalarından metin çıkar |
| `extract_binary_types` | PDF, DOCX, XLSX | Özel ayrıştırıcısı olan MIME tipleri; metin dosyaları otomatik algılanır |
| `max_extract_size` | `0` (sınırsız) | Bu boyuttan büyük dosyaları atla |
| `max_concurrent_extractions` | `4` | Paralel çıkarım işçisi |
| `extraction_timeout_secs` | `120` | Dosya başına çıkarım zaman aşımı |

### `config_version`

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `config_version` | `1` | Şema sürümü; kırıcı değişikliklerde artar. Eski dosyalar yeni alanlar için varsayılan alır |

## 6. Üretim İçin Kontrol Listesi

- [ ] `sync_on_write = true` (varsayılan) — değiştirmeyin
- [ ] `verify_on_read = "always"`, ZFS/Btrfs üzerindeyseniz `"native_only"`
- [ ] `root_secret_key` TOML'de değil, `MSTORE_ROOT_PASSWORD` ortam değişkeninde
- [ ] `parity_shards >= 2` (üretim; tek sürücü hariç)
- [ ] `[tls]` yapılandırılmış veya önünde TLS sonlandıran ters vekil var
- [ ] gRPC portu `9011` internete **kapalı** ([grpc-api.md](grpc-api.md))
- [ ] 10M+ nesne bekleniyorsa `path_hash_depth = 2`
- [ ] SSE-S3 kullanılacaksa `MSTORE_MASTER_KEY` ayarlı ve yedeklenmiş

## İlgili Belgeler

- [installation.md](installation.md) — Kurulum
- [docker-deployment.md](docker-deployment.md) — Docker / Podman
- [multidrive-setup.md](multidrive-setup.md) — Çok sürücülü kurulum
- [multinode-setup.md](multinode-setup.md) — Çok düğümlü küme
- [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — Dağıtım senaryoları

---

# English

## 1. Two Run Modes

MStore can be started in two ways:

**a) Zero-config** — pass drive paths directly, no configuration file needed:

```bash
mstore-server /data                        # single drive
mstore-server /data1 /data2 /data3 /data4  # 4 drives with erasure coding
```

**b) Configuration file** — the recommended route for production:

```bash
mstore-server --config /etc/mstore/config.toml
```

> When positional drive paths are given, `--config` is **ignored**.

## 2. Command-Line Options

| Option | Environment variable | Default | Description |
|---|---|---|---|
| `[PATH]...` | — | — | Drive paths for zero-config mode |
| `--config <PATH>` | `MSTORE_CONFIG` | — | TOML configuration file |
| `--api-bind <ADDR>` | `MSTORE_API_BIND` | `api.bind` | Override the S3 HTTP API address |
| `--grpc-bind <ADDR>` | `MSTORE_GRPC_BIND` | `0.0.0.0:9011` | Override the gRPC address |
| `--console-path <PATH>` | `MSTORE_CONSOLE_PATH` | — | Console static-files directory |

## 3. Environment Variables

| Variable | Purpose |
|---|---|
| `MSTORE_CONFIG` | Configuration file path |
| `MSTORE_API_BIND` | S3 API bind address |
| `MSTORE_GRPC_BIND` | gRPC bind address |
| `MSTORE_CONSOLE_PATH` | Console static-files directory |
| `MSTORE_ROOT_USER` | Root access key (overrides the config file) |
| `MSTORE_ROOT_PASSWORD` | Root secret key (overrides the config file) |
| `MSTORE_MASTER_KEY` | Master key for SSE-S3 server-side encryption |
| `RUST_LOG` | Log level — e.g. `info`, `mstore_server=debug` |

> In production supply credentials through `MSTORE_ROOT_USER` /
> `MSTORE_ROOT_PASSWORD` rather than writing them into the TOML file.

## 4. Minimum Configuration

```toml
[node]
name    = "mstore-1"
address = "127.0.0.1:9010"

[storage]
drives = [{ path = "/data/drive1" }]

[erasure]
data_shards   = 1
parity_shards = 0

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "change-me"
```

## 5. Sections

### `[node]` — Node identity

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | *(required)* | Name shown in logs and the admin API |
| `address` | string | *(required)* | Advertised address for peer-to-peer communication |
| `peers` | string[] | `[]` | Other nodes in a distributed deployment |
| `pg_count` | u32 | `0` | Placement-group count; `0` = single-node mode |
| `weight` | u32 | `1` | Rendezvous-hashing weight (proportional to capacity) |

Multi-node setup: [multinode-setup.md](multinode-setup.md)

### `[storage]` — Drives and metadata engine

| Field | Type | Default | Description |
|---|---|---|---|
| `drives` | table[] | *(required)* | Drive list (see below) |
| `direct_io` | bool | `true` | Bypass the OS page cache for large-object I/O. Set `false` on NFS/SMB |
| `sync_on_write` | bool | `true` | `fdatasync` after every write. **Never set `false` in production** |
| `verify_on_read` | enum | `always` | Re-verify BLAKE3 on every GET: `always` \| `native_only` (trust ZFS/Btrfs) \| `never` |
| `inline_threshold` | bytes | `4096` | Objects below this are stored inline in `xl.meta` instead of separate shard files |
| `path_hash_depth` | u8 | `0` | Path-hash depth to avoid hot directories. Use `2` for buckets expected to exceed 10M objects |
| `group_commit_batch` | count | `64` | `xl.meta` writes batched before a single `fdatasync` |
| `group_commit_interval_ms` | ms | `5` | Forced commit interval even when the batch is not full |
| `meta_cache_bytes` | bytes | `0` (disabled) | LRU cache for `xl.meta`; the RocksDB block cache is normally sufficient |
| `bloom_filter_capacity` | count | `10000000` | Expected number of distinct object keys |
| `meta_shard_count` | count | `0` (auto) | Metadata shard count; `0` = drives × 4 |
| `rocksdb_shard_threshold` | count | `50000000` | New shard is created when one shard exceeds this key count; `0` = disabled |
| `rocksdb_initial_shards` | count | `0` | Pre-split on first start; avoids write pauses under load |
| `rocksdb_compression` | string | `zstd` | For L2+ levels: `none` \| `snappy` \| `zstd` |
| `blob_min_size` | bytes | `256` | RocksDB values above this move to blob files; `0` disables BlobDB |
| `bloom_checkpoint_interval_secs` | s | `300` | Bloom-filter checkpoint interval for fast startup |
| `multipart_expiry_hours` | h | `168` | Age at which incomplete multipart uploads are cleaned up (7 days) |

**`drives` entries:**

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | path | *(required)* | Mount point |
| `metadata_only` | bool | `false` | Store only `xl.meta` — for a fast NVMe metadata drive in front of a NAS |
| `storage_class` | string | `STANDARD` | Class used for lifecycle tiering; e.g. NVMe=`STANDARD`, HDD=`STANDARD_IA` |

Multi-drive setup: [multidrive-setup.md](multidrive-setup.md)

### `[erasure]` — Erasure coding

| Field | Type | Default | Description |
|---|---|---|---|
| `data_shards` | u8 | *(required)* | Data shards (K). Total drives = K + M |
| `parity_shards` | u8 | *(required)* | Parity shards (M) = tolerated drive failures |
| `block_size` | bytes | `1048576` | Reed-Solomon block size (1 MiB) |

Rules:

- `data_shards >= 1`
- `data_shards + parity_shards <= 32`
- **Read quorum** = `data_shards`
- **Write quorum** = `data_shards`, but `data_shards + 1` when
  `parity_shards >= data_shards`, to avoid split-brain

### `[api]` — S3 HTTP API

| Field | Type | Default | Description |
|---|---|---|---|
| `bind` | string | `0.0.0.0:9010` | Listen address |
| `max_object_size` | bytes | 5 TiB | Largest object accepted in a single PUT |
| `request_timeout_secs` | s | `300` | Request timeout |
| `write_concurrency_per_cpu` | count | `30` | Concurrent writes (PUT/DELETE/POST) per core |
| `read_concurrency_per_cpu` | count | `150` | Concurrent reads (GET/HEAD) per core |
| `max_write_concurrency` | count | `0` (uncapped) | Absolute ceiling on writes |
| `max_read_concurrency` | count | `0` (uncapped) | Absolute ceiling on reads |
| `max_total_requests` | count | `10000` | In-flight request ceiling; exceeding it returns `503` |

Effective limit = `min(per_cpu × cpu_count, max_*)`.
`max_concurrent_requests` is a legacy field: it parses but is ignored.

### `[auth]` — Authentication

| Field | Type | Description |
|---|---|---|
| `root_access_key` | string | Root access key |
| `root_secret_key` | string | Root secret key — use an env var/vault in production |
| `[auth.ldap]` | table | Optional LDAP integration |
| `[auth.oidc]` | table | Optional OIDC provider |

**`[auth.ldap]`:** `server_url`, `bind_dn`, `bind_password`,
`user_search_base`, `user_search_filter`, `group_search_base` *(opt.)*

**`[auth.oidc]`:** `issuer_url`, `client_id`, `client_secret` *(opt.)*,
`role_claim` *(default `role`)*

### `[compliance]` — Compliance and audit

| Field | Type | Default | Description |
|---|---|---|---|
| `worm_compliance` | bool | `false` | SEC 17a-4 / FINRA 4511 / CFTC 1.31 WORM mode — COMPLIANCE locks cannot be overridden by anyone |
| `audit_retention_days` | u32 | `0` (forever) | Audit WAL retention |
| `audit_bucket` | string | `__audit__` | Bucket for audit records and destruction certificates |
| `audit_retention_years` | u32 | `10` | Audit record retention in years |
| `destruction_certificate` | bool | `false` | Generate signed destruction certificates on delete |
| `digimr_endpoint` | string | — | DigiMR e-signature API; when set, destruction certificates are digitally signed |
| `signature_verification` | string | `off` | Signature verification for uploads: `off` \| `warn` \| `enforce` |
| `kvkk_secure_delete` | bool | `false` | 3-pass secure delete for objects tagged `x-amz-meta-kvkk-secure-delete: true` |
| `[compliance.audit_kafka]` | table | — | Append audit events to Kafka in addition to the local WAL |

### `[replication]` — Cross-site replication *(optional)*

| Field | Description |
|---|---|
| `site_id` | Unique site identifier — cycle prevention and conflict resolution |
| `source_endpoint` | Local S3 endpoint used as the source for replication GETs |
| `targets` | Target site list |
| `conflict_resolution` | Conflict resolution strategy |
| `[replication.retry]` | `initial_backoff_ms` (1000), `max_backoff_ms` (60000), `max_retries` (`0` = infinite) |

### `[lambda]` — Event notifications *(optional)*

| Field | Description |
|---|---|
| `targets` | Webhook / Kafka / NATS and similar targets |
| `sqs_endpoint` | SQS-compatible endpoint (ARN routing) |
| `sns_endpoint` | SNS-compatible endpoint |
| `eventbridge_endpoint` | EventBridge-compatible endpoint |

### `[tls]` — HTTPS *(optional)*

| Field | Default | Description |
|---|---|---|
| `cert_path` | *(required)* | Certificate file (PEM) |
| `key_path` | *(required)* | Private key (PEM) |
| `min_version` | `1.2` | Minimum TLS version |

### `[kms]` — SSE-KMS *(optional)*

| Field | Default | Description |
|---|---|---|
| `endpoint` | *(required)* | KMS endpoint |
| `access_key` / `secret_key` | *(required)* | KMS credentials |
| `default_key_id` | *(required)* | Default master key for SSE-KMS |
| `dek_cache_ttl_secs` | `300` | Data-encryption-key cache TTL |

SSE-S3 needs no KMS; `MSTORE_MASTER_KEY` is enough.

### `[sftp]` — SFTP interface *(optional)*

| Field | Default | Description |
|---|---|---|
| `bind` | `0.0.0.0:2022` | SFTP listen address |
| `host_key_path` | *(required)* | SSH host key |

### `[lifecycle]` — Lifecycle rules *(optional)*

| Field | Default | Description |
|---|---|---|
| `scan_interval_secs` | `3600` | Rule-evaluation scan interval |
| `max_objects_per_sec` | `10000` | Throttle; `0` = unlimited (not recommended) |
| `off_peak_window` | — | Only scan inside this window — `HH:MM-HH:MM` (local time) |

### `[reclaim]` — Orphan reclamation *(optional)*

| Field | Default | Description |
|---|---|---|
| `tick_secs` | `30` | Worker wake interval (the event bus also wakes it) |
| `max_files_per_sec` | `10000` | Max files deleted per second; `0` = unlimited |
| `batch_size` | `1000` | Pending rows processed per pass |
| `deep_scan_min_age_secs` | `86400` | Metadata-less files younger than this are not treated as orphans |

Details: [orphan-reclamation.md](orphan-reclamation.md)

### `[index]` — Full-text search

Metadata indexing (key, size, content type, date) is **always on**. This section
only controls content extraction.

| Field | Default | Description |
|---|---|---|
| `content_extraction` | `false` | Extract text from PDF, DOCX, XLSX and text files |
| `extract_binary_types` | PDF, DOCX, XLSX | MIME types with dedicated parsers; text files are auto-detected |
| `max_extract_size` | `0` (unlimited) | Skip files larger than this |
| `max_concurrent_extractions` | `4` | Parallel extraction workers |
| `extraction_timeout_secs` | `120` | Per-file extraction timeout |

### `config_version`

| Field | Default | Description |
|---|---|---|
| `config_version` | `1` | Schema version; incremented on breaking changes. Older files get defaults for new fields |

## 6. Production Checklist

- [ ] `sync_on_write = true` (default) — leave it alone
- [ ] `verify_on_read = "always"`, or `"native_only"` on ZFS/Btrfs
- [ ] `root_secret_key` in `MSTORE_ROOT_PASSWORD`, not in the TOML file
- [ ] `parity_shards >= 2` (production; single-drive setups excepted)
- [ ] `[tls]` configured, or a reverse proxy terminating TLS in front
- [ ] gRPC port `9011` **closed** to the internet ([grpc-api.md](grpc-api.md))
- [ ] `path_hash_depth = 2` if 10M+ objects are expected
- [ ] `MSTORE_MASTER_KEY` set and backed up if SSE-S3 will be used

## Related Documents

- [installation.md](installation.md) — installation
- [docker-deployment.md](docker-deployment.md) — Docker / Podman
- [multidrive-setup.md](multidrive-setup.md) — multi-drive setup
- [multinode-setup.md](multinode-setup.md) — multi-node cluster
- [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — deployment scenarios
