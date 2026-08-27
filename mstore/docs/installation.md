# MStore Kurulum Kılavuzu / Installation Guide

> Sürüm / Version: **v0.4.0**

## İçindekiler / Table of Contents

- [Türkçe](#türkçe)
- [English](#english)

---

# Türkçe

## Dağıtım Paketleri

MStore önceden derlenmiş ikili dosyalar olarak dağıtılır — kaynak koda ve Rust
araç zincirine ihtiyacınız yoktur.

| Dosya | Platform | İçerik |
|---|---|---|
| `mstore-linux-amd64.tar.gz` | Linux x86_64 | `mstore-server`, `mstore` (CLI) |
| `mstore-windows-amd64.zip` | Windows x86_64 | `mstore-server.exe`, `mstore.exe` |
| `mstore-docker.tar.gz` | Docker / Podman | OCI imajı (`docker load`) |

İndirme bağlantıları: [MStore ürün sayfası](../README.tr.md)

## Sistem Gereksinimleri

| | Minimum | Önerilen |
|---|---|---|
| CPU | 2 çekirdek | 8+ çekirdek (AES-NI destekli) |
| RAM | 2 GB | 16 GB+ (metadata cache için) |
| Disk | 1 disk | Erasure coding için 4+ disk |
| İşletim sistemi | Linux (glibc 2.28+) / Windows 10+ | Linux |

Erasure coding kullanacaksanız disk sayısı `data_shards + parity_shards`
toplamına eşit olmalıdır. Ayrıntı: [multidrive-setup.md](multidrive-setup.md)

## Linux Kurulumu

```bash
# 1. İndir ve aç
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-linux-amd64.tar.gz
tar xzf mstore-linux-amd64.tar.gz
sudo install -m 0755 mstore-server mstore /usr/local/bin/

# 2. Veri dizini ve yapılandırma
sudo mkdir -p /var/lib/mstore/disk1 /etc/mstore
```

`/etc/mstore/config.toml`:

```toml
[node]
name    = "mstore-1"
address = "127.0.0.1:9010"

[[storage.drives]]
path = "/var/lib/mstore/disk1"

[erasure]
data_shards   = 1
parity_shards = 0

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "DEGISTIRIN"
root_secret_key = "BUNU-MUTLAKA-DEGISTIRIN"
```

```bash
# 3. Çalıştır
mstore-server --config /etc/mstore/config.toml
```

> **Uyarı:** `root_access_key` / `root_secret_key` boş bırakılırsa kimlik doğrulama
> tamamen devre dışı kalır. Üretimde mutlaka doldurun.

### systemd Servisi

```ini
# /etc/systemd/system/mstore.service
[Unit]
Description=MStore Object Storage
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mstore
Group=mstore
Environment=MSTORE_CONFIG=/etc/mstore/config.toml
Environment=RUST_LOG=info
ExecStart=/usr/local/bin/mstore-server --config /etc/mstore/config.toml
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --system --home /var/lib/mstore --shell /usr/sbin/nologin mstore
sudo chown -R mstore:mstore /var/lib/mstore
sudo systemctl daemon-reload
sudo systemctl enable --now mstore
sudo systemctl status mstore
```

## Windows Kurulumu

```powershell
# 1. İndir ve aç
Expand-Archive mstore-windows-amd64.zip -DestinationPath C:\mstore

# 2. Veri dizini
New-Item -ItemType Directory C:\mstore\data\disk1 -Force

# 3. config.toml dosyasını C:\mstore\config.toml olarak oluşturun (yukarıdaki örnek)

# 4. Çalıştır
C:\mstore\mstore-server.exe --config C:\mstore\config.toml
```

Windows servisi olarak çalıştırmak için [NSSM](https://nssm.cc) veya
`sc.exe create` kullanabilirsiniz.

## Docker / Podman Kurulumu

```bash
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-docker.tar.gz
docker load < mstore-docker.tar.gz      # Podman: podman load < ...

docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

Ayrıntılı Compose örnekleri (tek node, çoklu disk, 3-node cluster):
[docker-deployment.md](docker-deployment.md)

## Portlar

| Port | Protokol | Amaç | Dışarı açılmalı mı? |
|---|---|---|---|
| 9010 | HTTP | S3 uyumlu API + Web Konsol | Evet |
| 9011 | gRPC | SDK, CLI, replikasyon | İstemciler gRPC kullanacaksa |
| 2022 | SSH | SFTP ağ geçidi (opsiyonel) | Yalnızca etkinleştirilirse |

## Kurulumu Doğrulama

```bash
# Sağlık kontrolü (kimlik doğrulama gerektirmez)
curl http://127.0.0.1:9010/minio/health/live
curl http://127.0.0.1:9010/minio/health/ready

# Sürüm bilgisi — yönetim API'si olduğu için root ya da `s3:AdminAll` yetkili
# bir kimlikle SigV4 imzalı istek ister; imzasız çağrı 403 döner.
# Ayrıntı: s3-api.md → Yönetim API'si

# AWS CLI ile uçtan uca test
aws configure set aws_access_key_id     DEGISTIRIN
aws configure set aws_secret_access_key BUNU-MUTLAKA-DEGISTIRIN
aws --endpoint-url http://127.0.0.1:9010 s3 mb s3://test
echo "merhaba" > /tmp/a.txt
aws --endpoint-url http://127.0.0.1:9010 s3 cp /tmp/a.txt s3://test/
aws --endpoint-url http://127.0.0.1:9010 s3 ls s3://test/
```

## Web Konsol

Sunucu çalışırken tarayıcıdan açın:

```
http://127.0.0.1:9010/mstore/console
```

Kök erişim anahtarınızla giriş yapın. Sayfalar: Dashboard, Buckets, Objects,
IAM, Settings.

## CLI

Paketten çıkan `mstore` ikili dosyası bir S3/gRPC istemcisidir:

```bash
mstore mb     mstore://my-bucket
mstore cp     dosya.txt mstore://my-bucket/dosya.txt
mstore ls     mstore://my-bucket/
mstore head   mstore://my-bucket/dosya.txt
mstore rm     mstore://my-bucket/dosya.txt
mstore rb     mstore://my-bucket
mstore search mstore://my-bucket "sorgu"
```

## Lisans ve Deneme Süresi

İlk çalıştırmada **90 günlük Enterprise denemesi** başlar — kayıt gerekmez.
Deneme bittiğinde Free katman süresiz devam eder (4 TB, temel özellikler).

Denemeyi atlayıp doğrudan Free katmanda başlamak için:

```bash
touch /var/lib/mstore/.skip-trial
# veya
MSTORE_SKIP_TRIAL=1 mstore-server --config /etc/mstore/config.toml
```

Standard / Enterprise lisansları için: **info@moreum.com**

## Sonraki Adımlar

- [configuration.md](configuration.md) — Tüm yapılandırma anahtarları
- [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — Topoloji seçimi
- [multidrive-setup.md](multidrive-setup.md) — Erasure coding ile çoklu disk
- [multinode-setup.md](multinode-setup.md) — Cluster kurulumu
- [s3-api.md](s3-api.md) · [grpc-api.md](grpc-api.md) · [sdk-reference.md](sdk-reference.md)

---

# English

## Distribution Packages

MStore ships as pre-built binaries — you do not need the source tree or a Rust
toolchain.

| File | Platform | Contents |
|---|---|---|
| `mstore-linux-amd64.tar.gz` | Linux x86_64 | `mstore-server`, `mstore` (CLI) |
| `mstore-windows-amd64.zip` | Windows x86_64 | `mstore-server.exe`, `mstore.exe` |
| `mstore-docker.tar.gz` | Docker / Podman | OCI image (`docker load`) |

Download links: [MStore product page](../README.md)

## System Requirements

| | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores | 8+ cores (with AES-NI) |
| RAM | 2 GB | 16 GB+ (metadata cache) |
| Disk | 1 drive | 4+ drives for erasure coding |
| OS | Linux (glibc 2.28+) / Windows 10+ | Linux |

With erasure coding the drive count must equal `data_shards + parity_shards`.
See [multidrive-setup.md](multidrive-setup.md).

## Linux Installation

```bash
# 1. Download and extract
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-linux-amd64.tar.gz
tar xzf mstore-linux-amd64.tar.gz
sudo install -m 0755 mstore-server mstore /usr/local/bin/

# 2. Data directory and configuration
sudo mkdir -p /var/lib/mstore/disk1 /etc/mstore
```

`/etc/mstore/config.toml`:

```toml
[node]
name    = "mstore-1"
address = "127.0.0.1:9010"

[[storage.drives]]
path = "/var/lib/mstore/disk1"

[erasure]
data_shards   = 1
parity_shards = 0

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "CHANGE-ME"
root_secret_key = "CHANGE-ME-TOO"
```

```bash
# 3. Run
mstore-server --config /etc/mstore/config.toml
```

> **Warning:** leaving `root_access_key` / `root_secret_key` empty disables
> authentication entirely. Always set them in production.

### systemd Service

```ini
# /etc/systemd/system/mstore.service
[Unit]
Description=MStore Object Storage
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mstore
Group=mstore
Environment=MSTORE_CONFIG=/etc/mstore/config.toml
Environment=RUST_LOG=info
ExecStart=/usr/local/bin/mstore-server --config /etc/mstore/config.toml
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --system --home /var/lib/mstore --shell /usr/sbin/nologin mstore
sudo chown -R mstore:mstore /var/lib/mstore
sudo systemctl daemon-reload
sudo systemctl enable --now mstore
sudo systemctl status mstore
```

## Windows Installation

```powershell
# 1. Download and extract
Expand-Archive mstore-windows-amd64.zip -DestinationPath C:\mstore

# 2. Data directory
New-Item -ItemType Directory C:\mstore\data\disk1 -Force

# 3. Create C:\mstore\config.toml (see the sample above)

# 4. Run
C:\mstore\mstore-server.exe --config C:\mstore\config.toml
```

To run as a Windows service use [NSSM](https://nssm.cc) or `sc.exe create`.

## Docker / Podman Installation

```bash
wget https://github.com/moreum-tech/MBox/releases/download/mstore-v0.4.0/mstore-docker.tar.gz
docker load < mstore-docker.tar.gz      # Podman: podman load < ...

docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.4.0
```

Detailed Compose examples (single node, multi-drive, 3-node cluster):
[docker-deployment.md](docker-deployment.md)

## Ports

| Port | Protocol | Purpose | Expose publicly? |
|---|---|---|---|
| 9010 | HTTP | S3-compatible API + web console | Yes |
| 9011 | gRPC | SDK, CLI, replication | Only if clients use gRPC |
| 2022 | SSH | SFTP gateway (optional) | Only when enabled |

## Verifying the Installation

```bash
# Health checks (no authentication required)
curl http://127.0.0.1:9010/minio/health/live
curl http://127.0.0.1:9010/minio/health/ready

# Version info — this is admin API surface, so it needs a SigV4-signed request
# from root or an identity granted `s3:AdminAll`; unsigned calls get a 403.
# See s3-api.md → Admin API

# End-to-end test with the AWS CLI
aws configure set aws_access_key_id     CHANGE-ME
aws configure set aws_secret_access_key CHANGE-ME-TOO
aws --endpoint-url http://127.0.0.1:9010 s3 mb s3://test
echo "hello" > /tmp/a.txt
aws --endpoint-url http://127.0.0.1:9010 s3 cp /tmp/a.txt s3://test/
aws --endpoint-url http://127.0.0.1:9010 s3 ls s3://test/
```

## Web Console

With the server running, open:

```
http://127.0.0.1:9010/mstore/console
```

Sign in with your root access key. Pages: Dashboard, Buckets, Objects, IAM,
Settings.

## CLI

The `mstore` binary in the archive is an S3/gRPC client:

```bash
mstore mb     mstore://my-bucket
mstore cp     file.txt mstore://my-bucket/file.txt
mstore ls     mstore://my-bucket/
mstore head   mstore://my-bucket/file.txt
mstore rm     mstore://my-bucket/file.txt
mstore rb     mstore://my-bucket
mstore search mstore://my-bucket "query"
```

## Licensing and Trial

The first launch starts a **90-day Enterprise trial** — no registration needed.
When it ends, the Free tier continues indefinitely (4 TB, basic features).

To skip the trial and start directly in the Free tier:

```bash
touch /var/lib/mstore/.skip-trial
# or
MSTORE_SKIP_TRIAL=1 mstore-server --config /etc/mstore/config.toml
```

For Standard / Enterprise licenses: **info@moreum.com**

## Next Steps

- [configuration.md](configuration.md) — every configuration key
- [deployment-scenarios.tr.md](deployment-scenarios.tr.md) — choosing a topology
- [multidrive-setup.md](multidrive-setup.md) — multi-drive with erasure coding
- [multinode-setup.md](multinode-setup.md) — cluster setup
- [s3-api.md](s3-api.md) · [grpc-api.md](grpc-api.md) · [sdk-reference.md](sdk-reference.md)
