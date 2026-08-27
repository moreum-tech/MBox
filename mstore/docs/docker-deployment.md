# MStore Docker Deployment / MStore Docker Kurulumu

---

## Icindekiler / Table of Contents

- [Turkce](#turkce)
- [English](#english)

---

<a id="turkce"></a>

# Turkce

## Hizli Baslangic

Imaj bir OCI arsivi olarak dagitilir; once yukleyin:

```bash
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-docker.tar.gz
docker load < mstore-docker.tar.gz      # Podman: podman load < mstore-docker.tar.gz
```

Ardindan tek komutla baslatin:

```bash
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

Bu komut:
- S3 HTTP API'yi `9010` portunda yayinlar
- gRPC API'yi (SDK, CLI, replikasyon) `9011` portunda yayinlar
- Veriyi `mstore-data` adli Docker volume'a kaydeder

Calistigini dogrulayin:

```bash
curl http://localhost:9010/minio/health/live
```

---

## Docker Compose -- Tek Node

`docker-compose.yml` dosyasi olusturun:

```yaml
services:
  mstore:
    image: mstore:v0.4.0
    container_name: mstore
    restart: unless-stopped
    ports:
      - "9010:9010"
      - "9011:9011"
    volumes:
      - mstore-data:/data
    environment:
      RUST_LOG: info
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    healthcheck:
      test: ["CMD", "mstore", "ls", "mstore://"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  mstore-data:
```

Baslatmak icin:

```bash
docker compose up -d
```

Loglar:

```bash
docker compose logs -f mstore
```

---

## Docker Compose -- Coklu Disk (Erasure Coding)

Erasure coding ile veri korumasi icin birden fazla disk baglayabilirsiniz. Asagidaki ornekte 4 disk ile 2+2 (2 data shard + 2 parity shard) yapisi gosterilmektedir.

`docker-compose.yml`:

```yaml
services:
  mstore:
    image: mstore:v0.4.0
    container_name: mstore
    restart: unless-stopped
    ports:
      - "9010:9010"
      - "9011:9011"
    volumes:
      - ./config.toml:/etc/mstore/config.toml:ro
      - drive0:/data/d0
      - drive1:/data/d1
      - drive2:/data/d2
      - drive3:/data/d3
    environment:
      RUST_LOG: info
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

volumes:
  drive0:
  drive1:
  drive2:
  drive3:
```

`config.toml`:

```toml
config_version = 1

[node]
name    = "mstore-erasure"
address = "0.0.0.0:9011"
peers   = []

[storage]
drives = [
    { path = "/data/d0", metadata_only = false },
    { path = "/data/d1", metadata_only = false },
    { path = "/data/d2", metadata_only = false },
    { path = "/data/d3", metadata_only = false },
]
direct_io        = true
sync_on_write    = true
verify_on_read   = "always"
inline_threshold = 65536

[erasure]
data_shards   = 2
parity_shards = 2
block_size    = 1048576   # 1 MB

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "mstoreadmin"
```

Bu yapilandirma ile 4 diskten herhangi 2'si arizalansa bile verileriniz korunur.

---

## Docker Compose -- 3-Node Cluster

Yuksek erisilebilirlik icin 3 node'lu bir cluster kurabilirsiniz. Nginx load balancer ile istemci istekleri dagilir.

`docker-compose.yml`:

```yaml
services:
  node0:
    image: mstore:v0.4.0
    hostname: node0
    restart: unless-stopped
    ports:
      - "9010:9010"
    volumes:
      - ./node0.toml:/etc/mstore/config.toml:ro
      - node0-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  node1:
    image: mstore:v0.4.0
    hostname: node1
    restart: unless-stopped
    ports:
      - "9020:9010"
    volumes:
      - ./node1.toml:/etc/mstore/config.toml:ro
      - node1-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  node2:
    image: mstore:v0.4.0
    hostname: node2
    restart: unless-stopped
    ports:
      - "9030:9010"
    volumes:
      - ./node2.toml:/etc/mstore/config.toml:ro
      - node2-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - node0
      - node1
      - node2

volumes:
  node0-data:
  node1-data:
  node2-data:
```

`node0.toml`:

```toml
config_version = 1

[node]
name    = "node0"
address = "node0:9011"
peers   = ["node1:9011", "node2:9011"]

[storage]
drives = [
    { path = "/data/drive0", metadata_only = false },
]
direct_io      = true
sync_on_write  = true
verify_on_read = "always"

[erasure]
data_shards   = 1
parity_shards = 0
block_size    = 1048576

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "mstoreadmin"
```

`node1.toml` ve `node2.toml` icin sadece `[node]` bolumunu degistirin:

```toml
# node1.toml
[node]
name    = "node1"
address = "node1:9011"
peers   = ["node0:9011", "node2:9011"]

# node2.toml
[node]
name    = "node2"
address = "node2:9011"
peers   = ["node0:9011", "node1:9011"]
```

`nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream mstore {
        least_conn;
        server node0:9010;
        server node1:9010;
        server node2:9010;
    }

    server {
        listen 80;

        client_max_body_size 5g;

        location / {
            proxy_pass http://mstore;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_request_buffering off;
        }
    }
}
```

> **Not:** Docker Compose ortaminda node'lar birbirini hostname ile bulur (`node0`, `node1`, `node2`). Ek DNS ayari gerekmez.

---

## Ortam Degiskenleri

| Degisken | Varsayilan | Aciklama |
|----------|------------|----------|
| `MSTORE_ROOT_USER` | `mstoreadmin` | Root erisim anahtari (access key) |
| `MSTORE_ROOT_PASSWORD` | `mstoreadmin` | Root gizli anahtar (secret key) |
| `MSTORE_MASTER_KEY` | *(yok)* | Sifreleme ana anahtari (base64) |
| `MSTORE_CONFIG` | *(yok)* | Yapilandirma dosyasi yolu |
| `RUST_LOG` | `info` | Log seviyesi (`debug`, `info`, `warn`, `error`) |

---

## Portlar

| Port | Protokol | Amac |
|------|----------|------|
| 9010 | HTTP | S3 uyumlu API (AWS SDK, curl, web konsolu) |
| 9011 | gRPC | MStore SDK, CLI, replikasyon |

---

## Sifreleme

Sifrelemeyi etkinlestirmek icin bir master key olusturun ve ortam degiskeni olarak verin:

```bash
# Master key olustur:
openssl rand -base64 32

# Docker ile kullan:
docker run -d \
  -e MSTORE_MASTER_KEY="$(openssl rand -base64 32)" \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

Docker Compose ile:

```yaml
environment:
  MSTORE_MASTER_KEY: "your-base64-encoded-32-byte-key-here"
```

> **Uyari:** Master key kaybolursa sifreli veriler kurtarilamaz. Anahtari guvenli bir yerde yedekleyin.

---

## Volume Tavsiyeleri

- Veri kaliciligi icin her zaman named volume veya bind mount kullanin.
- **Uretim ortami (coklu disk):** Gercek disk bolumlerini bind mount edin:
  ```yaml
  volumes:
    - /mnt/disk0:/data/d0
    - /mnt/disk1:/data/d1
    - /mnt/disk2:/data/d2
    - /mnt/disk3:/data/d3
  ```
- **Gelistirme/test:** Named volume yeterlidir.
- Volume olmadan container durdurulursa tum veri kaybolur.

---

## Guvenlik

Uretim ortaminda asagidaki onlemleri uygulayin:

1. **Varsayilan kimlik bilgilerini degistirin:**
   ```bash
   export MSTORE_ROOT_USER="guclu-erisim-anahtari"
   export MSTORE_ROOT_PASSWORD="guclu-gizli-anahtar"
   ```

2. **Salt okunur container modu:**
   ```yaml
   read_only: true
   tmpfs:
     - /tmp
   ```

3. **Yetki yukseltmeyi engelle:**
   ```yaml
   security_opt:
     - no-new-privileges:true
   ```

4. **Dosya tanimlayici limitlerini ayarla:**
   ```yaml
   ulimits:
     nofile:
       soft: 65536
       hard: 65536
   ```

5. **Sifrelemeyi etkinlestirin** (yukaridaki Sifreleme bolumune bakin).

---

## Saglik Kontrolu

```bash
# HTTP saglik kontrolu:
curl http://localhost:9010/minio/health/live

# CLI ile:
docker exec mstore mstore ls mstore://
```

---
---

<a id="english"></a>

# English

## Quick Start

The image ships as an OCI archive; load it first:

```bash
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-docker.tar.gz
docker load < mstore-docker.tar.gz      # Podman: podman load < mstore-docker.tar.gz
```

Then start it with a single command:

```bash
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

This command:
- Exposes the S3 HTTP API on port `9010`
- Exposes the gRPC API (SDK, CLI, replication) on port `9011`
- Persists data to a Docker named volume `mstore-data`

Verify it is running:

```bash
curl http://localhost:9010/minio/health/live
```

---

## Docker Compose -- Single Node

Create a `docker-compose.yml` file:

```yaml
services:
  mstore:
    image: mstore:v0.4.0
    container_name: mstore
    restart: unless-stopped
    ports:
      - "9010:9010"
      - "9011:9011"
    volumes:
      - mstore-data:/data
    environment:
      RUST_LOG: info
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    healthcheck:
      test: ["CMD", "mstore", "ls", "mstore://"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  mstore-data:
```

Start:

```bash
docker compose up -d
```

View logs:

```bash
docker compose logs -f mstore
```

---

## Docker Compose -- Multi-Drive (Erasure Coding)

Mount multiple volumes for data protection with erasure coding. The example below uses 4 drives with a 2+2 configuration (2 data shards + 2 parity shards).

`docker-compose.yml`:

```yaml
services:
  mstore:
    image: mstore:v0.4.0
    container_name: mstore
    restart: unless-stopped
    ports:
      - "9010:9010"
      - "9011:9011"
    volumes:
      - ./config.toml:/etc/mstore/config.toml:ro
      - drive0:/data/d0
      - drive1:/data/d1
      - drive2:/data/d2
      - drive3:/data/d3
    environment:
      RUST_LOG: info
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

volumes:
  drive0:
  drive1:
  drive2:
  drive3:
```

`config.toml`:

```toml
config_version = 1

[node]
name    = "mstore-erasure"
address = "0.0.0.0:9011"
peers   = []

[storage]
drives = [
    { path = "/data/d0", metadata_only = false },
    { path = "/data/d1", metadata_only = false },
    { path = "/data/d2", metadata_only = false },
    { path = "/data/d3", metadata_only = false },
]
direct_io        = true
sync_on_write    = true
verify_on_read   = "always"
inline_threshold = 65536

[erasure]
data_shards   = 2
parity_shards = 2
block_size    = 1048576   # 1 MB

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "mstoreadmin"
```

With this configuration, your data survives the loss of any 2 out of 4 drives.

---

## Docker Compose -- 3-Node Cluster

For high availability, deploy a 3-node cluster with an Nginx load balancer.

`docker-compose.yml`:

```yaml
services:
  node0:
    image: mstore:v0.4.0
    hostname: node0
    restart: unless-stopped
    ports:
      - "9010:9010"
    volumes:
      - ./node0.toml:/etc/mstore/config.toml:ro
      - node0-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  node1:
    image: mstore:v0.4.0
    hostname: node1
    restart: unless-stopped
    ports:
      - "9020:9010"
    volumes:
      - ./node1.toml:/etc/mstore/config.toml:ro
      - node1-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  node2:
    image: mstore:v0.4.0
    hostname: node2
    restart: unless-stopped
    ports:
      - "9030:9010"
    volumes:
      - ./node2.toml:/etc/mstore/config.toml:ro
      - node2-data:/data
    environment:
      MSTORE_ROOT_USER: "${MSTORE_ROOT_USER:-mstoreadmin}"
      MSTORE_ROOT_PASSWORD: "${MSTORE_ROOT_PASSWORD:-mstoreadmin}"
    command: ["--config", "/etc/mstore/config.toml"]

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - node0
      - node1
      - node2

volumes:
  node0-data:
  node1-data:
  node2-data:
```

`node0.toml`:

```toml
config_version = 1

[node]
name    = "node0"
address = "node0:9011"
peers   = ["node1:9011", "node2:9011"]

[storage]
drives = [
    { path = "/data/drive0", metadata_only = false },
]
direct_io      = true
sync_on_write  = true
verify_on_read = "always"

[erasure]
data_shards   = 1
parity_shards = 0
block_size    = 1048576

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "mstoreadmin"
root_secret_key = "mstoreadmin"
```

For `node1.toml` and `node2.toml`, only change the `[node]` section:

```toml
# node1.toml
[node]
name    = "node1"
address = "node1:9011"
peers   = ["node0:9011", "node2:9011"]

# node2.toml
[node]
name    = "node2"
address = "node2:9011"
peers   = ["node0:9011", "node1:9011"]
```

`nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream mstore {
        least_conn;
        server node0:9010;
        server node1:9010;
        server node2:9010;
    }

    server {
        listen 80;

        client_max_body_size 5g;

        location / {
            proxy_pass http://mstore;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_request_buffering off;
        }
    }
}
```

> **Note:** In Docker Compose, nodes discover each other via hostname (`node0`, `node1`, `node2`). No additional DNS configuration is needed.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MSTORE_ROOT_USER` | `mstoreadmin` | Root access key |
| `MSTORE_ROOT_PASSWORD` | `mstoreadmin` | Root secret key |
| `MSTORE_MASTER_KEY` | *(none)* | Encryption master key (base64-encoded 32 bytes) |
| `MSTORE_CONFIG` | *(none)* | Config file path |
| `RUST_LOG` | `info` | Log level (`debug`, `info`, `warn`, `error`) |

---

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9010 | HTTP | S3-compatible API (AWS SDK, curl, web console) |
| 9011 | gRPC | MStore SDK, CLI, replication |

---

## Encryption

Enable encryption by generating a master key and passing it as an environment variable:

```bash
# Generate a master key:
openssl rand -base64 32

# Use with Docker:
docker run -d \
  -e MSTORE_MASTER_KEY="$(openssl rand -base64 32)" \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

With Docker Compose:

```yaml
environment:
  MSTORE_MASTER_KEY: "your-base64-encoded-32-byte-key-here"
```

> **Warning:** If the master key is lost, encrypted data cannot be recovered. Back up the key in a secure location.

---

## Volume Best Practices

- Always use named volumes or bind mounts for data persistence.
- **Production (multi-drive):** Bind mount real disk partitions:
  ```yaml
  volumes:
    - /mnt/disk0:/data/d0
    - /mnt/disk1:/data/d1
    - /mnt/disk2:/data/d2
    - /mnt/disk3:/data/d3
  ```
- **Development/testing:** Named volumes are sufficient.
- Without a volume, all data is lost when the container stops.

---

## Security

Apply the following hardening measures in production:

1. **Change default credentials:**
   ```bash
   export MSTORE_ROOT_USER="strong-access-key"
   export MSTORE_ROOT_PASSWORD="strong-secret-key"
   ```

2. **Read-only container filesystem:**
   ```yaml
   read_only: true
   tmpfs:
     - /tmp
   ```

3. **Prevent privilege escalation:**
   ```yaml
   security_opt:
     - no-new-privileges:true
   ```

4. **Set file descriptor limits:**
   ```yaml
   ulimits:
     nofile:
       soft: 65536
       hard: 65536
   ```

5. **Enable encryption** (see the Encryption section above).

---

## Health Check

```bash
# HTTP health check:
curl http://localhost:9010/minio/health/live

# Via CLI:
docker exec mstore mstore ls mstore://
```
