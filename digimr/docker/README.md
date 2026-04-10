[Turkce](README.tr.md)

# DigiMR Docker

**Download:** [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-docker.tar.gz) (v1.1.0)

---

## Quick Start

```bash
# Load the image from release archive
docker load < digimr-docker.tar.gz

# Run
docker run -d --name digimr \
  -p 7701:7701 -p 7702:7702 \
  -v digimr-data:/app/data \
  -e Kestrel__Endpoints__Local__Url=http://0.0.0.0:7701 \
  -e Kestrel__Endpoints__Local__Protocols=Http1AndHttp2 \
  -e Kestrel__Endpoints__Grpc__Url=http://0.0.0.0:7702 \
  -e Kestrel__Endpoints__Grpc__Protocols=Http2 \
  digimr:v1.1.0
```

```bash
# Health check (wait ~10 seconds for startup)
curl http://localhost:7701/api/v1/health
# Expected: {"status":"healthy",...}
```

**Ports:**
- 7701: REST API + Swagger UI (`http://localhost:7701/swagger`)
- 7702: gRPC API (for Java/.NET SDK)

The app starts with default settings. To configure TSA, licensing, or other options, visit `http://localhost:7701/setup/`.

## Docker Compose

Download [docker-compose.yml](docker-compose.yml) and run:

```bash
docker compose up -d
```

The compose file runs the API server on port 7701.

## Volumes

| Volume | Path in container | Purpose |
|--------|------------------|---------|
| `digimr-data` | `/app/data` | License file, SQLite DB, config |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ASPNETCORE_ENVIRONMENT` | `Production` | Runtime environment |
| `TSA_MODE` | `free` | KamuSM TSA tier (`free` or `paid`) |
| `Kestrel__Endpoints__Local__Url` | `http://127.0.0.1:7701` | REST API bind address (use `0.0.0.0` for container) |
| `Kestrel__Endpoints__Local__Protocols` | `Http1` | Set to `Http1AndHttp2` for container |
| `Kestrel__Endpoints__Grpc__Url` | — | gRPC bind address (e.g. `http://0.0.0.0:7702`) |
| `Kestrel__Endpoints__Grpc__Protocols` | — | Set to `Http2` for gRPC |

## Hardware Token (USB)

To pass a USB smart card reader to the container on Linux:

```yaml
services:
  digimr:
    devices:
      - /dev/bus/usb:/dev/bus/usb
    volumes:
      - /usr/lib/libaetpkss.so:/usr/lib/libaetpkss.so:ro
```

Replace the `.so` path with your token vendor's PKCS#11 library.

## Update

```bash
docker load < digimr-docker-new.tar.gz

docker stop digimr && docker rm digimr
docker run -d --name digimr \
  -p 7701:7701 -p 7702:7702 \
  -v digimr-data:/app/data \
  -e Kestrel__Endpoints__Local__Url=http://0.0.0.0:7701 \
  -e Kestrel__Endpoints__Local__Protocols=Http1AndHttp2 \
  -e Kestrel__Endpoints__Grpc__Url=http://0.0.0.0:7702 \
  -e Kestrel__Endpoints__Grpc__Protocols=Http2 \
  digimr:v1.1.0
```

Data is stored in volumes and persists across container updates.
