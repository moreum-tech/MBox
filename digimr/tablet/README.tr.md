[English](README.md)

# DigiMR Tablet İmza

Tablet/kiosk imza uygulaması: el imzası, eToken ile nitelikli e-imza ve mobil imza. Tablet yerel
bir HTTP API (port 7777) sunar; sisteminiz iş gönderir, operatör tablette imzalar, imzalı PDF'ler
`callbackUrl` adresinize geri gönderilir.

**Son sürüm:** [v1.1.0](https://github.com/moreum-tech/MBox/releases/tag/digimr-tablet-v1.1.0)

---

## Hemen deneyin — Test Paketi (Windows)

Tek bir kurulum dosyası tablet imza uygulamasını ve bir test web uygulamasını birlikte kurar. Bir Windows
tablete ya da herhangi bir Windows bilgisayara kurup uçtan uca deneyebilirsiniz (imza fare ile de
atılabilir).

1. [digimr-tablet-test-paketi-windows-amd64.msi](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.msi) dosyasını indirip çalıştırın.
2. Kurulum kapsamını seçin:
   - **Yalnızca benim için** — yönetici izni gerekmez; kurulum yeri
     `%LOCALAPPDATA%\Programs\DigiMR\Tablet`.
   - **Bu bilgisayarı kullanan herkes için** — yönetici izni gerekir; kurulum yeri
     `C:\Program Files\DigiMR\Tablet`, 5080 portu için Windows Güvenlik Duvarı izni de eklenir.

   Kurulum bitince iki uygulama da başlar; sonraki açılışlarda masaüstündeki **DigiMR Tablet**
   kısayolu kullanılır.
3. Tarayıcıda test sayfasını açın:
   - aynı bilgisayarda: `http://localhost:5080`
   - ağdaki başka bir PC'den: `http://<tablet-ip>:5080`
4. İmzalayan adını girin, PDF seçin, **İmzala'ya Gönder**'e basın. Belge tablet ekranında açılır;
   imza atılıp gönderilince imzalı PDF sayfadaki **İmzalı belgeler** listesinde görünür.

El imzasında isteğe bağlı kimlik doğrulaması seçilebilir: **MRZ** (kimlik kartının arka yüzü
kameraya gösterilir) ya da **NFC** (kart çipi okutulur).

Ayrıntılar (kaldırma, yönetici şifresini sıfırlama, sessiz kurulum) kurulum klasöründeki
`BENIOKU.txt` dosyasındadır (Başlat menüsü > DigiMR Tablet > Benioku). Kurulum paketi ve DigiMR
ikilileri Moreum Bilişim Teknolojileri A.Ş. adına kod imzalıdır.

## İndirmeler

| Dosya | Açıklama |
|-------|----------|
| [digimr-tablet-test-paketi-windows-amd64.msi](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.msi) | Test paketi kurulumu: tablet uygulaması + test web uygulaması (Windows 10/11 x64) |
| [digimr-tablet-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-windows-amd64.zip) | Tablet uygulaması — Windows 10/11 x64 (Microsoft Edge WebView2 Runtime gerekir) |
| [digimr-tablet-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-linux-amd64.tar.gz) | Tablet uygulaması — Linux x64 (.NET 10 + ASP.NET Core 10 runtime ve WebKitGTK gerekir) |
| [digimr-tablet-ornek-gonderici-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-windows-amd64.zip) | Örnek gönderici web uygulaması — Windows x64 |
| [digimr-tablet-ornek-gonderici-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-linux-amd64.tar.gz) | Örnek gönderici web uygulaması — Linux x64 |
| [digimr-tablet-ornek-gonderici-kaynak.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-kaynak.zip) | Örnek gönderici kaynak kodu (.NET 10) — kendi entegrasyonunuz için başlangıç noktası |
| [DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf) | Entegrasyon rehberi: API uçları, JSON alanları, callback sözleşmesi, güvenlik |

Her paketin kurulum adımları paket içindeki `README.md` dosyasındadır.

## Kendi sisteminizi bağlamak

Tablet uygulamasını tek başına kurun (`digimr-tablet-windows-amd64.zip` ya da Linux paketi) ve
işleri tabletin `POST /api/v1/relay-sign` ucuna gönderin. Ayrıntılar:

- [Entegrasyon rehberi (PDF)](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf)
- [Tablet relay-sign API](../docs/tr/TABLET_RELAY_SIGN.md)
- Örnek gönderici kaynak kodu: `digimr-tablet-ornek-gonderici-kaynak.zip`

## Portlar

| Port | Uygulama | Not |
|------|----------|-----|
| 7777 | Tablet uygulaması | Varsayılan yalnız `127.0.0.1`; ağ modunda `0.0.0.0`, `ApiKey` tanımlayın |
| 5080 | Test / örnek gönderici web uygulaması | Ağdaki tarayıcılardan açılır |

## Notlar

- Test web uygulamasında kullanıcı doğrulaması yoktur; yalnız güvenilir ağda deneme için kullanın.
- Motor DLL'leri obfuscate edilmiştir.
- Windows ikilileri (DigiMR `.exe` ve `DigitalSignature.*.dll`) **Moreum Bilişim Teknolojileri A.Ş.**
  adına kod imzalı ve zaman damgalıdır. Kurumsal güvenlik yazılımlarında (EDR, AppLocker / WDAC)
  yayıncı sertifikasına göre izin tanımlanabilir; bu izin sonraki sürümleri de kapsar.

[Tüm DigiMR Tablet sürümleri](https://github.com/moreum-tech/MBox/releases?q=digimr-tablet)

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
