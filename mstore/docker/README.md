[Turkce](README.tr.md)

# MStore Docker

**Download:** [mstore-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-docker.tar.gz) (v0.2.0)

---

## Quick Start

```bash
# Load the image from release archive
docker load < mstore-docker.tar.gz

# Run
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

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9010 | HTTP | S3-compatible API |
| 9011 | gRPC | SDK, CLI, replication |

## Update

```bash
# Load new version
docker load < mstore-docker-new.tar.gz

docker stop mstore && docker rm mstore
docker run -d --name mstore \
  -p 9010:9010 -p 9011:9011 \
  -v mstore-data:/data \
  mstore:v0.2.0
```

Data is stored in volumes and persists across container updates.
