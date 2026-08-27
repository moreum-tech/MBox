# MStore Multidrive Kurulumu (Erasure Coding)

Bu belge, MStore'un erasure coding ile coklu disk yapilandirmasini aciklar.
Tek disk gelistirme ortamindan, uretim duzeyinde cok diskli kurulumlara kadar
tum senaryolari kapsar.

---

## Erasure Coding Nedir?

MStore, veri dayanikliligi icin **Reed-Solomon erasure coding** kullanir.
Her nesne iki tur parcaya bolunur:

- **data_shards (K)**: Orijinal veriyi iceren parcalar
- **parity_shards (M)**: Kayip parcalari yeniden olusturmak icin hesaplanan yedek parcalar

Bu yaklasim, **M kadar disk arizasini** tolere ederek veri kaybini onler.
Toplam erasure set boyutu **K + M** olup, maksimum **32** olabilir.

**Ornek:** 4+4 yapilandirmada 8 diskin 4'u ayni anda arizalansa bile
tum veriler okunakliligi korur.

### Reed-Solomon Blok Boyutu

Her nesne 1 MiB (1.048.576 bayt) bloklara bolunur ve her blok bagimsiz olarak
Reed-Solomon kodlamasindan gecer. Bu boyut `erasure.block_size` ile
ayarlanabilir ancak varsayilan deger cogu senaryo icin uygundur.

---

## Drive Hazirligi

### Dosya Sistemi Secimi

| Dosya Sistemi | Oneri |
|---------------|-------|
| **XFS** | Onerilen (yuksek performans, O_DIRECT destegi) |
| **ext4** | Onerilen (genis uyumluluk) |
| **ZFS / Btrfs** | Kullanilabilir, `verify_on_read = "native_only"` ile |

### Mount Noktalari

Her disk icin ayri bir mount noktasi olusturun:

```
/data/d0
/data/d1
/data/d2
/data/d3
...
```

> **UYARI: RAID KULLANMAYIN.**
> MStore kendi erasure coding'ini yapar. Donanim veya yazilim RAID kullanmak
> gereksiz yuk ekler ve MStore'un arizali diskleri dogru sekilde tespitini engeller.
> Her diski bagimsiz olarak mount edin.

---

## Onerilen Yapilandirma Tablosu

| Drive Sayisi | data | parity | Set Boyutu | Tolerans | Depolama Verimliligi |
|:------------:|:----:|:------:|:----------:|:--------:|:--------------------:|
| 1 | 1 | 0 | 1 | 0 disk | %100 (yedeksiz) |
| 2 | 2 | 0 | 2 | 0 disk | %100 (yedeksiz) |
| 4 | 2 | 2 | 4 | 2 disk | %50 |
| 6 | 3 | 3 | 6 | 3 disk | %50 |
| 8 | 4 | 4 | 8 | 4 disk | %50 |
| 12 | 6 | 6 | 12 | 6 disk | %50 |
| 16 | 8 | 8 | 16 | 8 disk | %50 |
| 17+ | 8 | 8 | 16 | 8 disk | %50 |

### Otomatik Yapilandirma Kurallari (`default_for_paths`)

Konfigurasyon dosyasi olmadan sunucu baslatildiginda, MStore drive sayisina
gore erasure duzeni otomatik secer:

| Drive Sayisi | data_shards | parity_shards | Aciklama |
|:------------:|:-----------:|:-------------:|----------|
| 1 | 1 | 0 | Tek disk, yedek yok |
| 2-3 | N | 0 | Yedek yok, tum diskler veri icin |
| 4-16 | N/2 | N/2 | Maksimum dayaniklilik |
| 17+ | 8 | 8 | Set boyutu 16 ile sabitlenir |

---

## Konfigurasyon Ornekleri

### Tek Drive (Gelistirme / Test)

```toml
[storage]
drives = [{ path = "/data/d0" }]

[erasure]
data_shards = 1
parity_shards = 0
```

### 4 Drive (2+2)

```toml
[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards = 2
parity_shards = 2
```

### 8 Drive (4+4)

```toml
[storage]
drives = [
    { path = "/data/d0" }, { path = "/data/d1" },
    { path = "/data/d2" }, { path = "/data/d3" },
    { path = "/data/d4" }, { path = "/data/d5" },
    { path = "/data/d6" }, { path = "/data/d7" },
]

[erasure]
data_shards = 4
parity_shards = 4
```

---

## Hibrit Kurulum: NVMe + HDD

Farkli disk tiplerini bir arada kullanmak icin `metadata_only` ve
`storage_class` alanlari mevcuttur:

```toml
[storage]
drives = [
    { path = "/fast/nvme0", metadata_only = true },
    { path = "/data/hdd0", storage_class = "STANDARD" },
    { path = "/data/hdd1", storage_class = "STANDARD" },
    { path = "/cold/hdd2", storage_class = "STANDARD_IA" },
]
```

| Alan | Aciklama |
|------|----------|
| `metadata_only = true` | Disk yalnizca xl.meta depolar. Hizli NVMe diskleri metadata aramalari icin idealdir. |
| `storage_class` | Yasam dongusu katmanlamasi saglar: `STANDARD` -> `STANDARD_IA` -> `GLACIER` |

---

## Storage Ayarlari

`[storage]` bolumundeki onemli parametreler:

| Parametre | Varsayilan | Aciklama |
|-----------|:----------:|----------|
| `direct_io` | `true` | O_DIRECT ile buyuk I/O islemleri. NFS/SMB uzerinde `false` yapin. |
| `sync_on_write` | `true` | Her yazma sonrasi fdatasync. **Uretimde asla `false` yapmayin.** |
| `verify_on_read` | `"always"` | Her okumada BLAKE3 checksum dogrulamasi. Degerler: `"always"`, `"native_only"`, `"never"` |
| `inline_threshold` | `4096` | Bu boyutun altindaki nesneler metadata icerisinde saklanir (bayt). |
| `block_size` | `1048576` | Reed-Solomon blok boyutu (1 MiB). |
| `path_hash_depth` | `0` | Dizin hash derinligi. 10M+ nesne icin `2` onerilir. |
| `group_commit_batch` | `64` | Tek fdatasync'e kadar biriktirilecek yazma sayisi. |
| `bloom_filter_capacity` | `10000000` | Bloom filter kapasitesi (beklenen benzersiz anahtar sayisi). |

---

## Quorum (Yeter Sayisi)

MStore, yazma ve okuma islemleri icin farkli quorum degerleri kullanir:

| Islem | Formul | Aciklama |
|-------|--------|----------|
| **Yazma quorum** | parity >= data ise `data + 1`, degilse `data` | Split-brain onleme |
| **Okuma quorum** | `data` | Yeniden olusturma icin minimum disk sayisi |

**Ornekler:**

| Yapilandirma | Yazma Quorum | Okuma Quorum | Toplam |
|:------------:|:------------:|:------------:|:------:|
| 1+0 | 1/1 | 1/1 | 1 disk |
| 2+2 | 3/4 | 2/4 | 4 disk |
| 4+4 | 5/8 | 4/8 | 8 disk |
| 8+8 | 9/16 | 8/16 | 16 disk |

---

## Dogrulama

Kurulumu dogrulamak icin asagidaki komutlari calistirin:

```bash
# Sunucuyu baslat
mstore-server --config /etc/mstore/config.toml

# Test bucket olustur
mstore mb mstore://test-bucket

# Nesne yaz ve oku
echo "hello" | mstore pipe mstore://test-bucket/test.txt
mstore cat mstore://test-bucket/test.txt
```

---

## Zero-Config Modu

Konfigurasyon dosyasi olmadan, drive yollarini dogrudan arguman olarak verin:

```bash
mstore-server /data/d0 /data/d1 /data/d2 /data/d3
# Otomatik algilar: 4 drive -> 2+2 erasure
```

Kimlik bilgileri cevre degiskenlerinden alinir:

| Degisken | Varsayilan |
|----------|:----------:|
| `MSTORE_ROOT_USER` | `mstoreadmin` |
| `MSTORE_ROOT_PASSWORD` | `mstoreadmin` |

> **Not:** Varsayilan kimlik bilgilerini uretim ortaminda mutlaka degistirin.

---

## Sorun Giderme

| Hata Mesaji | Neden | Cozum |
|-------------|-------|-------|
| `not enough drives to form even one complete erasure set` | Drive sayisi < data + parity | Drive sayisini artirin veya parity degerini azaltin |
| `drive path must be absolute` | Gorecel yol kullanilmis | `/data/d0` gibi mutlak yol kullanin |
| `need at least N drives` | Yetersiz drive | Drive sayisini artirin veya erasure duzeni kucultun |
| `must be >= 1` | data_shards = 0 | data_shards en az 1 olmali |
| `must be <= 32` | data + parity > 32 | Toplam set boyutunu 32'nin altinda tutun |

---
---

# MStore Multidrive Setup (Erasure Coding)

This document explains how to configure MStore with multiple drives using
erasure coding. It covers all scenarios from single-drive development
environments to production-grade multi-drive setups.

---

## What is Erasure Coding?

MStore uses **Reed-Solomon erasure coding** for data durability.
Each object is split into two types of shards:

- **data_shards (K)**: Shards containing the original data
- **parity_shards (M)**: Computed redundancy shards used to reconstruct lost data

This approach tolerates up to **M simultaneous drive failures** without data loss.
The total erasure set size is **K + M**, with a maximum of **32**.

**Example:** In a 4+4 configuration, even if 4 out of 8 drives fail simultaneously,
all data remains readable.

### Reed-Solomon Block Size

Each object is split into 1 MiB (1,048,576 bytes) blocks, and each block is
independently Reed-Solomon encoded. This is configurable via `erasure.block_size`
but the default is suitable for most workloads.

---

## Drive Preparation

### Filesystem Selection

| Filesystem | Recommendation |
|------------|----------------|
| **XFS** | Recommended (high performance, O_DIRECT support) |
| **ext4** | Recommended (wide compatibility) |
| **ZFS / Btrfs** | Supported with `verify_on_read = "native_only"` |

### Mount Points

Create a separate mount point for each drive:

```
/data/d0
/data/d1
/data/d2
/data/d3
...
```

> **WARNING: DO NOT USE RAID.**
> MStore performs its own erasure coding. Using hardware or software RAID adds
> unnecessary overhead and prevents MStore from correctly detecting failed drives.
> Mount each drive independently.

---

## Recommended Configurations

| Drive Count | data | parity | Set Size | Tolerance | Storage Efficiency |
|:-----------:|:----:|:------:|:--------:|:---------:|:------------------:|
| 1 | 1 | 0 | 1 | 0 drives | 100% (no redundancy) |
| 2 | 2 | 0 | 2 | 0 drives | 100% (no redundancy) |
| 4 | 2 | 2 | 4 | 2 drives | 50% |
| 6 | 3 | 3 | 6 | 3 drives | 50% |
| 8 | 4 | 4 | 8 | 4 drives | 50% |
| 12 | 6 | 6 | 12 | 6 drives | 50% |
| 16 | 8 | 8 | 16 | 8 drives | 50% |
| 17+ | 8 | 8 | 16 | 8 drives | 50% |

### Auto-Configuration Rules (`default_for_paths`)

When the server is started without a configuration file, MStore automatically
selects an erasure layout based on the drive count:

| Drive Count | data_shards | parity_shards | Description |
|:-----------:|:-----------:|:-------------:|-------------|
| 1 | 1 | 0 | Single drive, no redundancy |
| 2-3 | N | 0 | No redundancy, all drives used for data |
| 4-16 | N/2 | N/2 | Maximum durability |
| 17+ | 8 | 8 | Fixed set size of 16 |

---

## Configuration Examples

### Single Drive (Development / Test)

```toml
[storage]
drives = [{ path = "/data/d0" }]

[erasure]
data_shards = 1
parity_shards = 0
```

### 4 Drives (2+2)

```toml
[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards = 2
parity_shards = 2
```

### 8 Drives (4+4)

```toml
[storage]
drives = [
    { path = "/data/d0" }, { path = "/data/d1" },
    { path = "/data/d2" }, { path = "/data/d3" },
    { path = "/data/d4" }, { path = "/data/d5" },
    { path = "/data/d6" }, { path = "/data/d7" },
]

[erasure]
data_shards = 4
parity_shards = 4
```

---

## Hybrid Setup: NVMe + HDD

Use the `metadata_only` and `storage_class` fields to combine different
drive types:

```toml
[storage]
drives = [
    { path = "/fast/nvme0", metadata_only = true },
    { path = "/data/hdd0", storage_class = "STANDARD" },
    { path = "/data/hdd1", storage_class = "STANDARD" },
    { path = "/cold/hdd2", storage_class = "STANDARD_IA" },
]
```

| Field | Description |
|-------|-------------|
| `metadata_only = true` | Drive only stores xl.meta. Ideal for fast NVMe drives handling metadata lookups. |
| `storage_class` | Enables lifecycle tiering: `STANDARD` -> `STANDARD_IA` -> `GLACIER` |

---

## Storage Settings

Key parameters in the `[storage]` section:

| Parameter | Default | Description |
|-----------|:-------:|-------------|
| `direct_io` | `true` | O_DIRECT for large I/O operations. Set `false` on NFS/SMB. |
| `sync_on_write` | `true` | fdatasync after every write. **NEVER set `false` in production.** |
| `verify_on_read` | `"always"` | BLAKE3 checksum verification on reads. Values: `"always"`, `"native_only"`, `"never"` |
| `inline_threshold` | `4096` | Objects below this size are stored inline in metadata (bytes). |
| `block_size` | `1048576` | Reed-Solomon block size (1 MiB). |
| `path_hash_depth` | `0` | Directory hash depth. Set to `2` for buckets exceeding 10M objects. |
| `group_commit_batch` | `64` | Max writes to batch before a single fdatasync. |
| `bloom_filter_capacity` | `10000000` | Bloom filter capacity (expected distinct key count). |

---

## Quorum

MStore uses different quorum values for write and read operations:

| Operation | Formula | Description |
|-----------|---------|-------------|
| **Write quorum** | `data + 1` if parity >= data, else `data` | Prevents split-brain |
| **Read quorum** | `data` | Minimum drives needed for reconstruction |

**Examples:**

| Configuration | Write Quorum | Read Quorum | Total Drives |
|:-------------:|:------------:|:-----------:|:------------:|
| 1+0 | 1/1 | 1/1 | 1 |
| 2+2 | 3/4 | 2/4 | 4 |
| 4+4 | 5/8 | 4/8 | 8 |
| 8+8 | 9/16 | 8/16 | 16 |

---

## Verification

Run the following commands to verify the setup:

```bash
# Start the server
mstore-server --config /etc/mstore/config.toml

# Create a test bucket
mstore mb mstore://test-bucket

# Write and read an object
echo "hello" | mstore pipe mstore://test-bucket/test.txt
mstore cat mstore://test-bucket/test.txt
```

---

## Zero-Config Mode

Without a configuration file, pass drive paths directly as arguments:

```bash
mstore-server /data/d0 /data/d1 /data/d2 /data/d3
# Auto-detects: 4 drives -> 2+2 erasure
```

Credentials are read from environment variables:

| Variable | Default |
|----------|:-------:|
| `MSTORE_ROOT_USER` | `mstoreadmin` |
| `MSTORE_ROOT_PASSWORD` | `mstoreadmin` |

> **Note:** Always change default credentials in production environments.

---

## Troubleshooting

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `not enough drives to form even one complete erasure set` | Drive count < data + parity | Increase drive count or reduce parity |
| `drive path must be absolute` | Relative path used | Use absolute paths like `/data/d0` |
| `need at least N drives` | Insufficient drives | Increase drive count or reduce erasure set size |
| `must be >= 1` | data_shards = 0 | data_shards must be at least 1 |
| `must be <= 32` | data + parity > 32 | Keep total set size under 32 |
