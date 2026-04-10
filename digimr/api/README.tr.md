[English](README.md)

# DigiMR API Sunucusu

Elektronik imza, dogrulama, zaman damgasi, EYP ve KEP islemleri icin REST API sunucusu.

**Indir:** [digimr-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz) (v1.1.0)

Bagimsiz binary — .NET runtime gerekmez.

---

## Kurulum

```bash
tar xzf digimr-linux-amd64.tar.gz
chmod +x DigiMR
sudo mv DigiMR /usr/local/bin/digimr
digimr --no-window
```

## Yapilandirma

Binary yanina `appsettings.json` dosyasi:

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

### Ortam Degiskenleri

| Degisken | Aciklama |
|----------|----------|
| `ASPNETCORE_ENVIRONMENT` | `Production` veya `Development` |
| `TSA_MODE` | `free` (KamuSM ucretsiz) veya `paid` |
| `ASPNETCORE_URLS` | Baglanti adresini gecersiz kil |

## API Endpoint'leri

Swagger UI: `http://localhost:7701/swagger`

| Yontem | Yol | Aciklama |
|--------|-----|----------|
| POST | `/api/v1/sign` | Belge imzala |
| POST | `/api/v1/sign/batch` | Toplu imzala |
| POST | `/api/v1/verify` | Imza dogrula |
| POST | `/api/v1/timestamp` | Zaman damgasi ekle |
| POST | `/api/v1/eyp` | EYP paketi olustur |
| POST | `/api/v1/kep` | KEP paketi olustur |
| GET | `/api/v1/health` | Saglik kontrolu |
| GET | `/api/v1/sign/formats` | Desteklenen formatlar |

## Sistem Gereksinimleri

- CPU: En az 2 vCPU
- RAM: En az 4 GB
- OS: Ubuntu 20.04+, Windows Server 2019+
- Donanim token icin: PKCS#11 surucusu yuklu olmali

## Guncelleme

1. Servisi durdurun
2. Binary'yi indirip degistirin
3. Servisi yeniden baslatin
