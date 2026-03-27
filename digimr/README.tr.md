[English](README.md)

# DigiMR

Turk elektronik imza platformu. CAdES, PAdES, XAdES, JAdES, ASiC-E imzalama ve dogrulama icin bagımsız REST API.

---

## Indirmeler

| Bilesen | Tur | Acıklama | Son Surum |
|---------|-----|----------|-----------|
| [API Sunucu](api/) | Servis | REST API sunucu dosyası | v0.1.0 |
| [Token Agent](agent/) | Arac | Donanım token erisimi icin yerel PKCS#11 koprusu | v0.1.0 |
| [Docker](docker/) | Container | Docker image ve compose | v0.1.0 |
| [C# SDK](sdk/csharp/) | SDK | .NET istemci kutuphanesi | v0.1.0 |

---

## Ozellikler

- Imza formatları — CAdES, PAdES, XAdES, JAdES, ASiC-E/S
- Imza seviyeleri — BES, T, LT, LTA (AdES baseline)
- EYP — 1.3 ve 2.0 arsiv paketleri
- KEP — Kayıtlı elektronik posta paketleri
- TSA — KamuSM ucretsiz/ucretli katman, yapilandırılabilir
- Mobil imza — Turkcell, Vodafone, Turk Telekom
- Token destegi — Yerel agent uzerinden PKCS#11 (AKiS, SafeNet, Gemalto)
- Toplu imzalama — tek cagrıda birden fazla belge imzalama
- Dogrulama — CRL/OCSP kontrollu imza dogrulaması
- Swagger UI — `/swagger` adresinde interaktif API dokumantasyonu

## Port

| Port | Protokol | Amac |
|------|----------|------|
| 7701 | HTTP | REST API ve Swagger UI |

---

info@moreum.com | [www.moreum.com](https://www.moreum.com)
