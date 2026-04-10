[English](README.md)

# DigiMR Docker

**Indir:** [digimr-docker.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-docker.tar.gz) (v1.1.0)

---

## Hizli Baslangic

```bash
# Release arsivinden imaji yukle
docker load < digimr-docker.tar.gz

# Calistir
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
# Saglik kontrolu (baslangic icin ~10 saniye bekleyin)
curl http://localhost:7701/api/v1/health
# Beklenen: {"status":"healthy",...}
```

**Portlar:**
- 7701: REST API + Swagger UI (`http://localhost:7701/swagger`)
- 7702: gRPC API (Java/.NET SDK icin)

Uygulama varsayilan ayarlarla baslar. TSA, lisans veya diger ayarlar icin: `http://localhost:7701/setup/`

## Docker Compose

[docker-compose.yml](docker-compose.yml) dosyasini indirip calistirin:

```bash
docker compose up -d
```

Compose dosyasi API sunucusunu port 7701 uzerinde calistirir.

## Volume'lar

| Volume | Konteyner yolu | Amac |
|--------|---------------|------|
| `digimr-data` | `/app/data` | Lisans dosyasi, SQLite DB, config |

## Ortam Degiskenleri

| Degisken | Varsayilan | Aciklama |
|----------|-----------|----------|
| `ASPNETCORE_ENVIRONMENT` | `Production` | Calisma ortami |
| `TSA_MODE` | `free` | KamuSM TSA katmani (`free` veya `paid`) |
| `Kestrel__Endpoints__Local__Url` | `http://127.0.0.1:7701` | REST API bind adresi (container icin `0.0.0.0` kullan) |
| `Kestrel__Endpoints__Local__Protocols` | `Http1` | Container icin `Http1AndHttp2` kullan |
| `Kestrel__Endpoints__Grpc__Url` | — | gRPC bind adresi (orn. `http://0.0.0.0:7702`) |
| `Kestrel__Endpoints__Grpc__Protocols` | — | gRPC icin `Http2` kullan |

## Donanim Token (USB)

Linux uzerinde USB akilli kart okuyucuyu konteynere baglamak icin:

```yaml
services:
  digimr:
    devices:
      - /dev/bus/usb:/dev/bus/usb
    volumes:
      - /usr/lib/libaetpkss.so:/usr/lib/libaetpkss.so:ro
```

`.so` yolunu token ureticinizin PKCS#11 kutuphanesiyle degistirin.

## Guncelleme

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

Veriler volume'da saklanir ve konteyner guncellemelerinde korunur.
