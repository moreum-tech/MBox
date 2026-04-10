[English](README.md)

# MStore Docker

**Indir:** [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz) (v0.2.0)

---

## Hizli Baslangic

```bash
# Release arsivinden imaji yukle
docker load < mstore-docker.tar.gz

# Calistir
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.2.0
```

## Docker Compose

```yaml
services:
  mstore:
    image: mstore:v0.2.0
    ports:
      - "9010:9010"
      - "9011:9011"
    volumes:
      - mstore-data:/data
      - ./config.toml:/etc/mstore/config.toml
    environment:
      - MSTORE_CONFIG=/etc/mstore/config.toml
    restart: unless-stopped

volumes:
  mstore-data:
```

## Portlar

| Port | Protokol | Amac |
|------|----------|------|
| 9010 | HTTP | S3 uyumlu API |
| 9011 | gRPC | SDK, CLI, replikasyon |

## Guncelleme

```bash
# Yeni surumu yukle
docker load < mstore-docker-new.tar.gz

docker stop mstore && docker rm mstore
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.2.0
```

Veriler volume'da saklanir ve konteyner guncellemelerinde korunur.
