[Türkçe](README.tr.md)

# DigiMR Tablet Signing

Tablet/kiosk signing application: handwritten signature, qualified e-signature with eToken, and
mobile signature. The tablet exposes a local HTTP API (port 7777); your system sends a job, the
operator signs on the tablet, and the signed PDFs are posted back to your `callbackUrl`.

**Latest release:** [v1.1.0](https://github.com/moreum-tech/MBox/releases/tag/digimr-tablet-v1.1.0)

---

## Try it now — Test Package (Windows)

A single installer sets up the tablet signing app together with a test web app. Install it on a Windows
tablet or on any Windows computer and test end to end (signing also works with a mouse).

1. Download and run [digimr-tablet-test-paketi-windows-amd64.msi](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.msi).
2. Choose the installation scope:
   - **Just for me** — no administrator rights needed; installs to
     `%LOCALAPPDATA%\Programs\DigiMR\Tablet`.
   - **Everyone on this computer** — requires administrator rights; installs to
     `C:\Program Files\DigiMR\Tablet` and adds a Windows Firewall rule for port 5080.

   Then set the tablet app's administrator password (at least 4 characters; asked when opening
   the Settings page and on updates). When setup finishes both apps start; the desktop shortcut
   **DigiMR Tablet** starts them later.
3. Open the test page in a browser:
   - on the same computer: `http://localhost:5080`
   - from another PC on the network: `http://<tablet-ip>:5080`
4. Enter the signer name, choose a PDF and press **İmzala'ya Gönder**. The document opens on the
   tablet; once signed and submitted, the signed PDF appears in the **İmzalı belgeler** list.

For handwritten signatures an optional identity check can be selected: **MRZ** (back of the ID
card shown to the camera) or **NFC** (card chip read).

### Updating

Run the new MSI over the existing installation; there is no need to uninstall first. The installer
finds the existing installation and installs to the same location.

- **If you know the administrator password:** enter it; all settings are kept.
- **If you don't:** setup continues like a fresh install. The previous settings are moved to
  `tablet-windows\ayar-yedegi-<date>\`, settings return to defaults and a new password is set.

In both cases the license and signed documents are kept. If the existing installation was made
for everyone, the update asks for administrator approval.

Details (uninstall, admin password reset, silent install) are in `BENIOKU.txt` in the install
folder (Turkish; also under Start menu > DigiMR Tablet > Benioku). The installer and the DigiMR
binaries are code-signed by Moreum Bilişim Teknolojileri A.Ş.

## Downloads

| File | Description |
|------|-------------|
| [digimr-tablet-test-paketi-windows-amd64.msi](https://github.com/moreum-tech/MBox/releases/download/digimr-tablet-v1.1.0/digimr-tablet-test-paketi-windows-amd64.msi) | Test package installer: tablet app + test web app (Windows 10/11 x64) |
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
