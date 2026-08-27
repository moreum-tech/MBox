# MStore Multinode (Cluster) Kurulumu / Multinode (Cluster) Setup

---

## Turkce / Turkish

---

### On Gereksinimler

- **En az 3 sunucu** (quorum icin: N/2+1 node ayakta olmali)
- **Ag erisimi:** Tum node'lar birbirinin gRPC portuna (`9011`) ve S3 HTTP portuna (`9010`) erisebilmeli
- **Ayni binary:** Her node'da ayni surumde `mstore-server` binary'si yuklu olmali
- **Disk yapisi:** Her node'da en az `data_shards + parity_shards` kadar disk/dizin tanimlanmali
- **Saat senkronizasyonu:** NTP ile tum node'larin saatleri senkron olmali
- **Load balancer (opsiyonel ama onerilir):** HAProxy, Nginx veya cloud LB ile client'larin tek bir endpoint uzerinden erisimi saglanabilir

### Multinode Nasil Calisir

MStore cluster'i, objeleri node'lar arasinda **Rendezvous (HRW) hashing** ile dagitir:

1. Her obje, bucket ve key'ine gore bir **placement group (PG)** 'a atanir.
2. Placement group'lar, HRW hashing ile node'lara eslestirilir. Varsayilan PG sayisi: `pg_count = 8192`.
3. Client **herhangi bir node'a** baglanabilir. Node, istegi alir:
   - Obje **local** disklerindeyse dogrudan servis eder.
   - Obje **baska bir node'da** ise, istegi gRPC uzerinden ilgili node'a **seffaf sekilde proxy** eder.
4. Erasure coding ile veri `data_shards + parity_shards` parcaya bolunur ve farkli disklere yazilir.

Bu mimari sayesinde client'in hangi node'a baglandigi farketmez; tum cluster tek bir depolama alani gibi gorunur.

### Adim Adim Kurulum

#### 1. Dizin Yapisi

Her node'da veri dizinlerini olusturun:

```bash
sudo mkdir -p /data/d0 /data/d1 /data/d2 /data/d3
sudo chown mstore:mstore /data/d0 /data/d1 /data/d2 /data/d3
```

#### 2. Konfigurasyon Dosyalari

**Node 0 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-0"
address  = "10.0.1.10:9011"
peers    = ["http://10.0.1.11:9011", "http://10.0.1.12:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

**Node 1 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-1"
address  = "10.0.1.11:9011"
peers    = ["http://10.0.1.10:9011", "http://10.0.1.12:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

**Node 2 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-2"
address  = "10.0.1.12:9011"
peers    = ["http://10.0.1.10:9011", "http://10.0.1.11:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

> **Onemli:** `root_access_key` ve `root_secret_key` degerlerini kendi guclu kimlik bilgilerinizle degistirin. Tum node'larda **ayni** auth bilgileri kullanilmalidir.

#### 3. Node'lari Baslatma

Her node'da:

```bash
mstore-server --config /etc/mstore/config.toml
```

Systemd ile calistirmak icin ornek unit dosyasi:

```ini
[Unit]
Description=MStore Object Storage
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mstore
Group=mstore
ExecStart=/usr/local/bin/mstore-server --config /etc/mstore/config.toml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mstore
```

#### 4. Dogrulama

```bash
# Herhangi bir node uzerinden cluster'a erisimi dogrulayin:
mstore-ctl --endpoint http://10.0.1.10:9010 ls mstore://

# Bucket olusturun:
mstore-ctl --endpoint http://10.0.1.10:9010 mb mstore://test-bucket

# Dosya yukleyin ve indirin:
mstore-ctl --endpoint http://10.0.1.10:9010 cp /tmp/test.txt mstore://test-bucket/test.txt
mstore-ctl --endpoint http://10.0.1.11:9010 cp mstore://test-bucket/test.txt /tmp/test-download.txt
```

Farkli node'lar uzerinden hem yazma hem okuma test ederek seffaf proxy'nin calistigini dogrulayin.

### Load Balancer Konfigurasyonu

Client'larin tek bir endpoint uzerinden cluster'a erisebilmesi icin bir load balancer onerilir.

**HAProxy ornegi (`/etc/haproxy/haproxy.cfg`):**

```
frontend mstore_s3
    bind *:9010
    default_backend mstore_nodes

backend mstore_nodes
    balance roundrobin
    option httpchk GET /minio/health/live
    http-check expect status 200
    server node0 10.0.1.10:9010 check inter 5s fall 3 rise 2
    server node1 10.0.1.11:9010 check inter 5s fall 3 rise 2
    server node2 10.0.1.12:9010 check inter 5s fall 3 rise 2
```

**Nginx ornegi (`/etc/nginx/conf.d/mstore.conf`):**

```nginx
upstream mstore_cluster {
    least_conn;
    server 10.0.1.10:9010 max_fails=3 fail_timeout=10s;
    server 10.0.1.11:9010 max_fails=3 fail_timeout=10s;
    server 10.0.1.12:9010 max_fails=3 fail_timeout=10s;
}

server {
    listen 9010;

    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://mstore_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

> **Not:** `client_max_body_size 0` ve `proxy_buffering off` buyuk dosya yuklemeleri icin gereklidir.

### Node Agirliklari (Weight)

`weight` alani, bir node'un cluster icindeki kapasite oranini belirler. Daha fazla diske veya daha buyuk disklere sahip node'lara daha yuksek weight deger atanir.

| Node | Disk Sayisi | Weight |
|------|-------------|--------|
| node-0 | 4 disk | 1 |
| node-1 | 8 disk | 2 |
| node-2 | 12 disk | 3 |

HRW hashing, weight degerlerini dikkate alarak objeleri oransal sekilde dagitir. Bu sayede buyuk kapasiteli node'lar daha fazla obje barindirabilir.

### Quorum ve Tolerans

| Parametre | Deger | Aciklama |
|-----------|-------|----------|
| Cluster quorum | N/2 + 1 | 3 node'lu cluster'da en az 2 node ayakta olmali |
| Yazma quorum | data_shards + 1 | 2+1 = 3 shard basarili yazilmali |
| Okuma quorum | data_shards | 2 shard okunabilirse obje okunabilir |

**Ornekler:**
- **3 node:** 1 node kaybedilebilir, cluster calismaya devam eder.
- **5 node:** 2 node kaybedilebilir.
- **7 node:** 3 node kaybedilebilir.

> **Uyari:** Quorum saglanamadigi durumda cluster yazma islemlerini reddeder. Okuma islemleri, mevcut shard'lar yeterliyse devam edebilir.

### Monitoring

Prometheus metrikleri her node'dan toplanabilir:

```
http://<node-ip>:9010/minio/v2/metrics/cluster
```

Her node ayrica kendi sagligi icin bir health endpoint sunar:

```
http://<node-ip>:9010/minio/health/live
```

Ornek Prometheus scrape konfigurasyonu (`prometheus.yml`):

```yaml
scrape_configs:
  - job_name: 'mstore'
    scrape_interval: 15s
    static_configs:
      - targets:
          - '10.0.1.10:9010'
          - '10.0.1.11:9010'
          - '10.0.1.12:9010'
    metrics_path: /minio/v2/metrics/cluster
```

### Sorun Giderme

| Sorun | Olasi Neden | Cozum |
|-------|-------------|-------|
| Node peer'lara baglanamyor | Firewall veya ag sorunu | `9011` (gRPC) ve `9010` (HTTP) portlarinin acik oldugunu dogrulayin |
| Cluster quorum hatasi | Yetersiz sayida node ayakta | En az N/2+1 node'un calistigini kontrol edin |
| Obje okunamyor | Yeterli shard mevcut degil | Disk sagligini kontrol edin; heal islemi baslatin |
| Yavas performans | Ag gecikme suresi yuksek | Node'lar arasi ping suresini kontrol edin; ayni veri merkezinde konumlanmayi tercih edin |
| Auth hatasi | Farkli credential'lar | Tum node'larda `root_access_key` ve `root_secret_key` degerlerinin ayni oldugunu dogrulayin |
| Disk dolu hatasi | Yetersiz disk alani | `df -h` ile bos alani kontrol edin; gerekirse disk ekleyip weight'i guncelleyin |

**Log seviyesini artirma:**

```bash
RUST_LOG=debug mstore-server --config /etc/mstore/config.toml
```

---

## English

---

### Prerequisites

- **Minimum 3 servers** (quorum requires N/2+1 nodes to be available)
- **Network access:** All nodes must be able to reach each other on gRPC port (`9011`) and S3 HTTP port (`9010`)
- **Same binary:** The same version of `mstore-server` must be installed on every node
- **Disk layout:** Each node must have at least `data_shards + parity_shards` drives/directories configured
- **Clock synchronization:** All nodes must be time-synced via NTP
- **Load balancer (optional but recommended):** HAProxy, Nginx, or a cloud LB to give clients a single endpoint

### How Multinode Works

MStore distributes objects across nodes using **Rendezvous (HRW) hashing**:

1. Each object is assigned to a **placement group (PG)** based on its bucket and key.
2. Placement groups are mapped to nodes via HRW hashing. Default PG count: `pg_count = 8192`.
3. A client can connect to **any node**. The node that receives the request:
   - Serves the object directly if it resides on **local** disks.
   - **Transparently proxies** the request via gRPC to the appropriate node if the object is elsewhere.
4. Erasure coding splits data into `data_shards + parity_shards` fragments distributed across different disks.

This architecture means it does not matter which node a client connects to; the entire cluster appears as a single storage system.

### Step-by-Step Setup

#### 1. Directory Structure

Create data directories on each node:

```bash
sudo mkdir -p /data/d0 /data/d1 /data/d2 /data/d3
sudo chown mstore:mstore /data/d0 /data/d1 /data/d2 /data/d3
```

#### 2. Configuration Files

**Node 0 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-0"
address  = "10.0.1.10:9011"
peers    = ["http://10.0.1.11:9011", "http://10.0.1.12:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

**Node 1 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-1"
address  = "10.0.1.11:9011"
peers    = ["http://10.0.1.10:9011", "http://10.0.1.12:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

**Node 2 (`/etc/mstore/config.toml`):**

```toml
config_version = 1

[node]
name     = "node-2"
address  = "10.0.1.12:9011"
peers    = ["http://10.0.1.10:9011", "http://10.0.1.11:9011"]
pg_count = 8192
weight   = 1

[storage]
drives = [
    { path = "/data/d0" },
    { path = "/data/d1" },
    { path = "/data/d2" },
    { path = "/data/d3" },
]

[erasure]
data_shards   = 2
parity_shards = 2

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "YOUR_ACCESS_KEY"
root_secret_key = "YOUR_SECRET_KEY"

[compliance]
```

> **Important:** Replace `root_access_key` and `root_secret_key` with your own strong credentials. All nodes **must** use the same auth values.

#### 3. Starting Nodes

On each node:

```bash
mstore-server --config /etc/mstore/config.toml
```

Example systemd unit file for production deployments:

```ini
[Unit]
Description=MStore Object Storage
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mstore
Group=mstore
ExecStart=/usr/local/bin/mstore-server --config /etc/mstore/config.toml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mstore
```

#### 4. Verification

```bash
# Verify cluster access from any node:
mstore-ctl --endpoint http://10.0.1.10:9010 ls mstore://

# Create a bucket:
mstore-ctl --endpoint http://10.0.1.10:9010 mb mstore://test-bucket

# Upload and download a file:
mstore-ctl --endpoint http://10.0.1.10:9010 cp /tmp/test.txt mstore://test-bucket/test.txt
mstore-ctl --endpoint http://10.0.1.11:9010 cp mstore://test-bucket/test.txt /tmp/test-download.txt
```

Test both reads and writes through different nodes to confirm transparent proxying works correctly.

### Load Balancer Configuration

A load balancer is recommended so clients can reach the cluster through a single endpoint.

**HAProxy example (`/etc/haproxy/haproxy.cfg`):**

```
frontend mstore_s3
    bind *:9010
    default_backend mstore_nodes

backend mstore_nodes
    balance roundrobin
    option httpchk GET /minio/health/live
    http-check expect status 200
    server node0 10.0.1.10:9010 check inter 5s fall 3 rise 2
    server node1 10.0.1.11:9010 check inter 5s fall 3 rise 2
    server node2 10.0.1.12:9010 check inter 5s fall 3 rise 2
```

**Nginx example (`/etc/nginx/conf.d/mstore.conf`):**

```nginx
upstream mstore_cluster {
    least_conn;
    server 10.0.1.10:9010 max_fails=3 fail_timeout=10s;
    server 10.0.1.11:9010 max_fails=3 fail_timeout=10s;
    server 10.0.1.12:9010 max_fails=3 fail_timeout=10s;
}

server {
    listen 9010;

    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://mstore_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

> **Note:** `client_max_body_size 0` and `proxy_buffering off` are required for large file uploads.

### Node Weights

The `weight` field determines a node's relative capacity within the cluster. Assign higher weights to nodes with more disks or larger disks.

| Node | Disk Count | Weight |
|------|------------|--------|
| node-0 | 4 disks | 1 |
| node-1 | 8 disks | 2 |
| node-2 | 12 disks | 3 |

HRW hashing takes weight values into account and distributes objects proportionally. This ensures nodes with greater capacity store more objects.

### Quorum and Tolerance

| Parameter | Value | Description |
|-----------|-------|-------------|
| Cluster quorum | N/2 + 1 | In a 3-node cluster, at least 2 nodes must be up |
| Write quorum | data_shards + 1 | 2+1 = 3 shards must be written successfully |
| Read quorum | data_shards | Object is readable if 2 shards are available |

**Examples:**
- **3 nodes:** Can tolerate 1 node failure.
- **5 nodes:** Can tolerate 2 node failures.
- **7 nodes:** Can tolerate 3 node failures.

> **Warning:** When quorum cannot be established, the cluster rejects write operations. Read operations may continue if sufficient shards are available.

### Monitoring

Prometheus metrics can be scraped from every node:

```
http://<node-ip>:9010/minio/v2/metrics/cluster
```

Each node also exposes a health endpoint:

```
http://<node-ip>:9010/minio/health/live
```

Example Prometheus scrape configuration (`prometheus.yml`):

```yaml
scrape_configs:
  - job_name: 'mstore'
    scrape_interval: 15s
    static_configs:
      - targets:
          - '10.0.1.10:9010'
          - '10.0.1.11:9010'
          - '10.0.1.12:9010'
    metrics_path: /minio/v2/metrics/cluster
```

### Troubleshooting

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Node cannot reach peers | Firewall or network issue | Verify ports `9011` (gRPC) and `9010` (HTTP) are open |
| Cluster quorum error | Too few nodes available | Ensure at least N/2+1 nodes are running |
| Object unreadable | Insufficient shards available | Check disk health; initiate a heal operation |
| Slow performance | High network latency | Check inter-node ping times; prefer co-location in the same datacenter |
| Auth error | Mismatched credentials | Verify `root_access_key` and `root_secret_key` are identical on all nodes |
| Disk full error | Insufficient disk space | Check free space with `df -h`; add disks and update weight if needed |

**Increasing log verbosity:**

```bash
RUST_LOG=debug mstore-server --config /etc/mstore/config.toml
```
