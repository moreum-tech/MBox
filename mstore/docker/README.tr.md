[English](README.md)

# MStore Docker

Resmi Docker image ve compose yapılandırmaları.

---

## Etiketler

| Etiket | Açıklama |
|--------|----------|
| `latest` | En son kararlı sürüm |
| `v0.1.0` | Belirli sürüm |

## Hızlı Başlangıç

```bash
docker pull ghcr.io/moreum-tech/mstore:latest

docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  ghcr.io/moreum-tech/mstore:latest
```

## Docker Compose

```yaml
version: "3.8"
services:
  mstore:
    image: ghcr.io/moreum-tech/mstore:latest
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

## Güncelleme

```bash
docker pull ghcr.io/moreum-tech/mstore:latest
docker stop mstore && docker rm mstore
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  ghcr.io/moreum-tech/mstore:latest
```

Veriler volume'larda saklanır ve konteyner güncellemelerinde korunur.
