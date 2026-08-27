# MStore Deployment Senaryoları

## Kimler Kullanacak?

| Segment | Örnek | İhtiyaç | MStore Çözümü |
|---------|-------|---------|---------------|
| **SaaS şirketleri** | CRM, ERP, proje yönetimi | Müşteri dosyaları, ekler, export'lar | Multi-tenant, tenant başı kota |
| **Medya / içerik** | Video platformu, CDN origin | Büyük dosyalar, yüksek okuma | Erasure coding, streaming GET |
| **Finans / sigorta** | Banka, sigorta şirketi | Düzenleyici arşiv, WORM | Object Lock, compliance mode |
| **Sağlık** | Hastane, PACS sistemi | DICOM görüntüler, uzun saklama | Tiering (NVMe → HDD), encryption |
| **Oyun / mobil** | Oyun stüdyosu, mobil uygulama | Asset dağıtımı, save dosyaları | Presigned URL, gRPC SDK |
| **AI/ML** | Model eğitimi, veri pipeline | Eğitim verileri, checkpoint'ler | Batch ops, yüksek throughput |
| **Backup / DR** | IT departmanı | Yedekleme hedefi, replikasyon | Site replication, lifecycle |
| **IoT / Edge** | Fabrika, sensör ağı | Edge'den merkeze veri toplama | Hafif node, async replication |

---

## Senaryo 1: Tek Node (Startup / Geliştirici)

```
Kullanıcı: Küçük SaaS, geliştirici, PoC
Veri:      < 1 TB
Bütçe:     Minimum
```

```
┌─────────────────────┐
│    MStore Node       │
│    S3: :9010         │
│    gRPC: :9011       │
│    4 disk (RAID yok) │
│    Erasure: 2+2      │
└─────────────────────┘
       ↑
    Uygulama (AWS SDK)
```

**Kurulum:** Tek binary, tek config.toml, `mstore-server /data` ile başlat.

**Ne zaman yetmez:** Sunucu kapanırsa servis durur. Disk arızasında parity koruyor ama node seviyesinde SPOF var.

---

## Senaryo 2: Aktif-Pasif (KOBİ)

```
Kullanıcı: Orta ölçekli şirket, 7/24 uptime gerekli
Veri:      1-50 TB
Bütçe:     2 sunucu
```

```
┌─────────────────┐    site replication    ┌─────────────────┐
│  Primary Node   │ ───────────────────→   │  Standby Node   │
│  S3: :9010      │    (async)             │  S3: :9010      │
│  8 disk, 4+4    │                        │  8 disk, 4+4    │
│  İstanbul DC    │                        │  Ankara DC      │
└─────────────────┘                        └─────────────────┘
       ↑                                          ↑
    Uygulama                              (failover durumunda)
```

**Kurulum:**
- 2 sunucu, her birinde MStore
- Primary'de `[replication]` config ile async replikasyon
- DNS failover veya load balancer ile yönlendirme

**Gerekli:**
- Site replication config
- Monitoring (Prometheus + Grafana)
- Failover scripti veya DNS health check

---

## Senaryo 3: Multi-Node Cluster (Kurumsal)

```
Kullanıcı: Büyük şirket, yüksek kapasite, yüksek erişilebilirlik
Veri:      50 TB - 1 PB
Bütçe:     3-5 sunucu
```

```
                    Load Balancer (:443)
                    ┌──────┴──────┐
                    ↓             ↓             ↓
           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
           │   node-0     │ │   node-1     │ │   node-2     │
           │   S3: :9010  │ │   S3: :9010  │ │   S3: :9010  │
           │   gRPC: :9011│ │   gRPC: :9011│ │   gRPC: :9011│
           │   12 disk    │ │   12 disk    │ │   12 disk    │
           │   pg=8192    │ │   pg=8192    │ │   pg=8192    │
           └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                  │     gRPC peer mesh      │
                  └────────────────────────────────┘
```

**Nasıl çalışır:**
1. Client herhangi bir node'a request gönderir (load balancer arkasında)
2. Node, rendezvous hashing ile objenin sahibini belirler
3. Obje local'deyse → doğrudan servis eder
4. Obje başka node'daysa → gRPC ile proxy eder, client fark etmez

**Kurulum:**
```toml
# node-0 config
[node]
name     = "node-0"
address  = "10.0.1.10:9011"
peers    = ["http://10.0.1.11:9011", "http://10.0.1.12:9011"]
pg_count = 8192
weight   = 1      # Eşit kapasite

[storage]
drives = [
    { path = "/data/d0" }, { path = "/data/d1" },
    { path = "/data/d2" }, { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2
```

**Gerekli:**
- 3+ sunucu (quorum: N/2+1)
- Her sunucuda aynı MStore binary
- Load balancer (HAProxy, Nginx, veya cloud LB)
- Shared nothing — her node'un kendi diskleri

---

## Senaryo 4: Multi-Site (Coğrafi Dağıtık)

```
Kullanıcı: Çok lokasyonlu şirket, düşük latency her bölgede
Veri:      100 TB+
Bütçe:     Site başı 3+ sunucu
```

```
       İstanbul Cluster              Frankfurt Cluster
   ┌────────────────────┐        ┌────────────────────┐
   │ node-0  node-1     │  site  │ node-0  node-1     │
   │ node-2  (3 node)   │ ←───→ │ node-2  (3 node)   │
   │ LB: ist.store.com  │ repli  │ LB: fra.store.com  │
   └────────────────────┘ cation └────────────────────┘
           ↑                              ↑
     Türkiye kullanıcıları          Avrupa kullanıcıları
```

**Nasıl çalışır:**
- Her site kendi cluster'ı ile bağımsız çalışır
- Site replication (async/sync/quorum) ile veriler senkron
- GeoDNS ile kullanıcılar en yakın site'a yönlendirilir
- Bir site kapansa diğeri tam hizmet verir

**Gerekli:**
- Site başı 3+ node cluster
- Site replication config (async önerilir — WAN latency)
- GeoDNS veya global load balancer
- Conflict resolution: Last-Write-Wins (UUID v7 timestamp)

---

## Senaryo 5: Edge + Merkez (IoT / Fabrika)

```
Kullanıcı: Üretim tesisi, sensör verileri
Veri:      Edge: 100 GB/gün, Merkez: PB seviyesi
```

```
   Fabrika 1          Fabrika 2          Fabrika 3
   ┌──────────┐       ┌──────────┐       ┌──────────┐
   │ MStore   │       │ MStore   │       │ MStore   │
   │ 1 node   │       │ 1 node   │       │ 1 node   │
   │ 2 disk   │       │ 2 disk   │       │ 2 disk   │
   └────┬─────┘       └────┬─────┘       └────┬─────┘
        │    async          │    async         │
        └──────────────┬────┘──────────────────┘
                       ↓
              ┌─────────────────┐
              │  Merkez Cluster │
              │  3 node, 36 disk│
              │  Uzun saklama   │
              │  Tiering: HDD   │
              └─────────────────┘
```

**Nasıl çalışır:**
- Edge node'lar sensör verilerini local'de saklar (düşük latency)
- Async replication ile merkeze kopyalar
- Merkez cluster lifecycle ile: 30 gün NVMe → HDD → 1 yıl sonra sil
- Edge'de disk dolduğunda eski veriler lifecycle ile silinir

---

## Senaryo 6: Multi-Tenant SaaS Platform

```
Kullanıcı: SaaS platform sağlayıcı, müşterilerine storage sunar
Veri:      Tenant başı 10 GB - 10 TB
```

```
                     API Gateway
                         ↓
              ┌──────────────────────┐
              │   MStore Cluster     │
              │   3 node             │
              │                      │
              │   Tenant: acme-corp  │  → Kota: 500 GB
              │   Tenant: startup-x  │  → Kota: 50 GB
              │   Tenant: enterprise │  → Kota: 10 TB
              └──────────────────────┘
```

**Nasıl çalışır:**
- Her tenant'ın kendi bucket namespace'i: `acme-corp/bucket1`, `acme-corp/bucket2`
- Tenant bazlı kota (storage + object count + request rate)
- Tenant bazlı IAM (her tenant kendi access key'leri)
- Tek cluster üzerinde izole çalışır

**Gerekli:**
- Multi-tenancy config
- Tenant yönetim API'si
- Rate limiting per tenant
- Billing entegrasyonu (kullanım raporları)

---

## Topoloji Karar Ağacı

```
Veri < 1 TB?
  └→ Evet → Senaryo 1 (tek node)

7/24 uptime gerekli?
  └→ Hayır → Senaryo 1
  └→ Evet ↓

Veri < 50 TB?
  └→ Evet → Senaryo 2 (aktif-pasif)
  └→ Hayır ↓

Tek lokasyon?
  └→ Evet → Senaryo 3 (multi-node cluster)
  └→ Hayır → Senaryo 4 (multi-site)

IoT / Edge?
  └→ Senaryo 5

Multi-tenant SaaS?
  └→ Senaryo 6
```

---

## Minimum Donanım Gereksinimleri

| Senaryo | CPU | RAM | Disk | Ağ |
|---------|-----|-----|------|-----|
| Tek node | 4 core | 8 GB | 4× HDD/SSD | 1 Gbps |
| Aktif-pasif | 4 core × 2 | 16 GB × 2 | 4× SSD × 2 | 1 Gbps + WAN |
| 3-node cluster | 8 core × 3 | 32 GB × 3 | 12× SSD × 3 | 10 Gbps |
| Multi-site | Cluster × site | Cluster × site | Cluster × site | 10 Gbps + WAN |
| Edge | 2 core | 4 GB | 2× SSD | WiFi/4G |

---

## Kurulacak Bileşenler

| Bileşen | Tek Node | Cluster | Multi-Site |
|---------|----------|---------|------------|
| mstore-server | ✓ | ✓ (her node) | ✓ (her node) |
| config.toml | ✓ | ✓ (node-specific) | ✓ (node-specific) |
| Load balancer | - | HAProxy/Nginx | HAProxy/Nginx |
| TLS sertifikası | Opsiyonel | Önerilir | Zorunlu |
| Monitoring | Opsiyonel | Prometheus+Grafana | Prometheus+Grafana |
| GeoDNS | - | - | Cloudflare/Route53 |
| Backup | Cron + rsync | Site replication | Site replication |
