[English](README.md)

# DigiMR Token Agent

Donanim token ve akilli kart erisimi icin yerel agent. Port 5555 uzerinde calisir; DigiMR API sunucusunu kullanicinin makinesindeki PKCS#11 cihazlariyla koprular.

Hibrit modda imzalama PIN'i asla kullanicinin bilgisayarindan cikmaz.

Token Agent binary'si [API sunucu paketine](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz) dahildir.

---

## Kurulum

```bash
tar xzf digimr-linux-amd64.tar.gz
chmod +x DigiMR.Agent
./DigiMR.Agent
# http://localhost:5555 uzerinde calisir
```

## Yapilandirma

```json
{
  "Kestrel": {
    "Endpoints": {
      "Local": { "Url": "http://127.0.0.1:5555" }
    }
  }
}
```

## Kullanim

Agent, DigiMR API'de `RemoteToken` saglayicisiyla imzalama yapildiginda otomatik olarak devreye girer. Kullanicinin tarayicisi SignalR uzerinden `http://localhost:5555` adresine baglanir; API sunucusu PIN'i hicbir zaman gormez.

## Sistem Gereksinimleri

- CPU: 1 vCPU
- RAM: 512 MB
- OS: Windows 10+, Ubuntu 20.04+
- Token markaniza ait PKCS#11 surucusu yuklu olmali (AKiS, SafeNet, Gemalto vb.)
