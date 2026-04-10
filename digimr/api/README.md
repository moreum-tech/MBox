[Turkce](README.tr.md)

# DigiMR API Server

REST API server for electronic signature, verification, timestamp, EYP, and KEP operations.

**Download:** [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz) (v1.1.0)

Self-contained binary — no .NET runtime required.

---

## Installation

```bash
tar xzf digimr-linux-amd64.tar.gz
chmod +x DigiMR
sudo mv DigiMR /usr/local/bin/digimr
digimr --no-window
```

## Configuration

Place `appsettings.json` next to the binary:

```json
{
  "Kestrel": {
    "Endpoints": {
      "Local":    { "Url": "http://127.0.0.1:7701" },
      "External": { "Url": "http://0.0.0.0:7701" }
    }
  },
  "DefaultTsaUrl": "http://tzd.kamusm.gov.tr",
  "TsaMode": "free"
}
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `ASPNETCORE_ENVIRONMENT` | `Production` or `Development` |
| `TSA_MODE` | `free` (KamuSM free tier) or `paid` |
| `ASPNETCORE_URLS` | Override bind address |

## API Endpoints

Swagger UI: `http://localhost:7701/swagger`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/sign` | Sign document |
| POST | `/api/v1/sign/batch` | Batch sign |
| POST | `/api/v1/verify` | Verify signature |
| POST | `/api/v1/timestamp` | Add timestamp |
| POST | `/api/v1/eyp` | Create EYP package |
| POST | `/api/v1/kep` | Create KEP package |
| GET | `/api/v1/health` | Health check |
| GET | `/api/v1/sign/formats` | Supported formats |

## System Requirements

- CPU: 2 vCPU minimum
- RAM: 4 GB minimum
- OS: Ubuntu 20.04+, Windows Server 2019+
- For hardware token: PKCS#11 driver installed

## Update

1. Stop the service
2. Download and replace the binary
3. Restart the service
