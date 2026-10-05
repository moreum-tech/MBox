[Türkçe](README.tr.md)

# DigiMR Tablet Signing

Tablet/kiosk signing application: handwritten signature, qualified e-signature with eToken, and
mobile signature. The tablet exposes a local HTTP API (port 7777); your system sends a job, the
operator signs on the tablet, and the signed PDFs are posted back to your `callbackUrl`.

**Latest release:** [v1.1.0](https://github.com/moreum-tech/MBox/releases/tag/digimr-tablet-v1.1.0)

---

## Try it now — Test Package (Windows)

A single zip installs the tablet signing app together with a test web app. Install it on a Windows
tablet or on any Windows computer and test end to end (signing also works with a mouse).

1. Download [digimr-tablet-test-paketi-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.zip) and extract it to a folder.
2. Run `Kurulum.bat` and accept the administrator prompt. At the end both apps start and the
   addresses to open are shown.
3. Open the test page in a browser:
   - on the same computer: `http://localhost:5080`
   - from another PC on the network: `http://<tablet-ip>:5080`
4. Enter the signer name, choose a PDF and press **İmzala'ya Gönder**. The document opens on the
   tablet; once signed and submitted, the signed PDF appears in the **İmzalı belgeler** list.

For handwritten signatures an optional identity check can be selected: **MRZ** (back of the ID
card shown to the camera) or **NFC** (card chip read).

Details (uninstall, admin password reset, troubleshooting) are in the `README.md` inside the zip
(Turkish).

## Downloads

| File | Description |
|------|-------------|
| [digimr-tablet-test-paketi-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.zip) | Test package: tablet app + test web app + installer (Windows 10/11 x64) |
| [digimr-tablet-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-windows-amd64.zip) | Tablet app — Windows 10/11 x64 (requires Microsoft Edge WebView2 Runtime) |
| [digimr-tablet-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-linux-amd64.tar.gz) | Tablet app — Linux x64 (requires .NET 10 + ASP.NET Core 10 runtime and WebKitGTK) |
| [digimr-tablet-ornek-gonderici-windows-amd64.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-windows-amd64.zip) | Sample sender web app — Windows x64 |
| [digimr-tablet-ornek-gonderici-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-linux-amd64.tar.gz) | Sample sender web app — Linux x64 |
| [digimr-tablet-ornek-gonderici-kaynak.zip](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-ornek-gonderici-kaynak.zip) | Sample sender source code (.NET 10) — a starting point for your integration |
| [DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf) | Integration guide (Turkish): API endpoints, JSON fields, callback contract, security |

Installation steps for each package are in the `README.md` inside it.

## Connecting your own system

Install the tablet app on its own (`digimr-tablet-windows-amd64.zip` or the Linux package) and send
jobs to the tablet's `POST /api/v1/relay-sign` endpoint. See:

- [Integration guide (PDF)](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/DigiMR_Tablet_Imzalama_Entegrasyon_Rehberi.pdf)
- [Tablet relay-sign API](../docs/tr/TABLET_RELAY_SIGN.md)
- Sample sender source code: `digimr-tablet-ornek-gonderici-kaynak.zip`

## Ports

| Port | Application | Note |
|------|-------------|------|
| 7777 | Tablet app | `127.0.0.1` only by default; `0.0.0.0` in network mode — set an `ApiKey` |
| 5080 | Test / sample sender web app | Opened from browsers on the network |

## Notes

- The test web app has no user authentication; use it only for trials on a trusted network.
- Engine DLLs are obfuscated.
- Windows binaries (DigiMR `.exe` and `DigitalSignature.*.dll`) are code-signed and timestamped by
  **Moreum Bilişim Teknolojileri A.Ş.** Enterprise security software (EDR, AppLocker / WDAC) can
  allow them with a publisher-certificate rule, which also covers future releases.

[All DigiMR Tablet releases](https://github.com/moreum-tech/MBox/releases?q=digimr-tablet)

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
