[English](README.md)

# DigiMR Token Agent

Donanim token ve akilli kart erisimi icin yerel agent. Port 5555 uzerinde calisir; DigiMR API sunucusunu kullanicinin makinesindeki PKCS#11 cihazlariyla koprular.

Hibrit modda imzalama PIN'i asla kullanicinin bilgisayarindan cikmaz.

Token Agent binary'si [API sunucu paketine](https://github.com/moreum-tech/MBox/releases/download/digimr-v2.1.0/digimr-linux-amd64.tar.gz) dahildir.

---

## Tarayici / Web Imza Agent'i (Windows)

Dogrudan web sayfasindan imzalamak icin (JavaScript Web SDK), hafif standalone agent indirilir —
tek dosya, self-contained `digimr-agent-win-x64.exe` (~3 MB, .NET gerektirmez). Cift tikla calistir;
kurulum/unzip yok.

- **Indirme:** En guncel [DigiMR surumunden](https://github.com/moreum-tech/MBox/releases?q=digimr) `digimr-agent-win-x64.exe`.
- **Su an yalniz Windows** — native PIN penceresi Windows'a ozel; macOS/Linux yakinda.

`http://localhost:5555` uzerinde calisir ve asagidaki Token Agent ile ayni token API'sini sunar.

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
