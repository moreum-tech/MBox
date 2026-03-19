# digimr Desktop Agent - Kurulum ve Kullanım Kılavuzu
# digimr Desktop Agent - Installation and Usage Guide

**Versiyon**: 1.0
**Tarih**: Şubat 2026

---

## İçindekiler / Table of Contents

1. [Giriş / Introduction](#giriş--introduction)
2. [Kurulum / Installation](#kurulum--installation)
3. [Yapılandırma / Configuration](#yapılandırma--configuration)
4. [Kullanım / Usage](#kullanım--usage)
5. [Web Entegrasyonu / Web Integration](#web-entegrasyonu--web-integration)
6. [Güvenlik / Security](#güvenlik--security)
7. [Sorun Giderme / Troubleshooting](#sorun-giderme--troubleshooting)

---

## Giriş / Introduction

### Türkçe

**digimr Desktop Agent**, kullanıcının bilgisayarında yerel olarak çalışan bir imza servisidir. USB token veya akıllı kart gibi donanımsal güvenlik cihazlarına (HSM) erişim sağlayarak, web tarayıcısında çalışan uygulamaların bu cihazları kullanarak belgelere imza atmasını mümkün kılar.

**Temel Özellikler:**
- 🔌 USB Token/Akıllı Kart erişimi (PKCS#11)
- 🌐 Web uygulamaları için localhost API (port 5555)
- 🔒 PIN güvenliği ve oturum yönetimi
- 📝 CAdES, PAdES, XAdES imza formatları
- 📦 EYP 1.3, 2.0, 2.1 paket desteği
- 🚀 Minimal Dependency - Hafif ve hızlı

### English

**digimr Desktop Agent** is a local signature service running on the user's computer. It provides access to hardware security devices (HSM) such as USB tokens or smart cards, enabling web browser applications to sign documents using these devices.

**Key Features:**
- 🔌 USB Token/Smart Card access (PKCS#11)
- 🌐 Localhost API for web applications (port 5555)
- 🔒 PIN security and session management
- 📝 CAdES, PAdES, XAdES signature formats
- 📦 EYP 1.3, 2.0, 2.1 package support
- 🚀 Minimal Dependency - Lightweight and fast

---

## Kurulum / Installation

### Ön Gereksinimler / Prerequisites

#### Windows
```bash
# .NET 8.0 Runtime yükleyin / Install .NET 8.0 Runtime
winget install Microsoft.DotNet.Runtime.8

# Token sürücüsünü yükleyin / Install token driver
# Örnek: AKiS, SafeNet, Gemalto vb. / Example: AKiS, SafeNet, Gemalto, etc.
```

#### Linux
```bash
# .NET 8.0 Runtime
sudo apt-get install -y dotnet-runtime-8.0

# OpenSC (PKCS#11 kütüphanesi)
sudo apt-get install -y opensc pcscd
sudo systemctl start pcscd
```

#### macOS
```bash
# .NET 8.0 Runtime
brew install dotnet@8

# OpenSC
brew install opensc
```

### Agent Kurulumu / Agent Installation

#### Seçenek 1: Binary Release (Önerilen / Recommended)

```bash
# Windows
curl -L https://github.com/moreum-tech/digimr/releases/latest/download/digimr-agent-win-x64.zip -o agent.zip
unzip agent.zip -d C:\Program Files\digimr
cd "C:\Program Files\digimr"

# Linux
curl -L https://github.com/moreum-tech/digimr/releases/latest/download/digimr-agent-linux-x64.tar.gz -o agent.tar.gz
tar -xzf agent.tar.gz -C /opt/digimr
cd /opt/digimr

# macOS
curl -L https://github.com/moreum-tech/digimr/releases/latest/download/digimr-agent-osx-x64.tar.gz -o agent.tar.gz
tar -xzf agent.tar.gz -C /Applications/digimr
cd /Applications/digimr
```

#### Seçenek 2: Kaynak Koddan / From Source

```bash
# Repository klonlayın / Clone repository
git clone https://github.com/moreum-tech/digimr.git
cd digimr

# Build yapın / Build
dotnet build src/DigitalSignature.Agent/DigitalSignature.Agent.csproj -c Release

# Çalıştırın / Run
cd src/DigitalSignature.Agent/bin/Release/net8.0
dotnet DigitalSignature.Agent.dll
```

---

## Yapılandırma / Configuration

### appsettings.json

Agent'ı yapılandırmak için `appsettings.json` dosyasını düzenleyin:

```json
{
  "Agent": {
    "Mode": "Local",
    "Port": 5555,
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://myapp.example.com"
    ],
    "RequireApiKey": false,
    "ApiKey": null,
    "EnableSwagger": true,
    "SessionTimeout": 300
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### Yapılandırma Parametreleri / Configuration Parameters

| Parametre | Açıklama / Description | Varsayılan / Default |
|-----------|------------------------|---------------------|
| `Mode` | Agent modu: "Local" veya "Remote" / Agent mode: "Local" or "Remote" | `"Local"` |
| `Port` | Dinleme portu / Listening port | `5555` |
| `AllowedOrigins` | İzin verilen web uygulaması URL'leri (CORS) / Allowed web app URLs (CORS) | `["*"]` |
| `RequireApiKey` | API key zorunluluğu / Require API key | `false` |
| `ApiKey` | API erişim anahtarı / API access key | `null` |
| `EnableSwagger` | Swagger UI etkinleştirme / Enable Swagger UI | `true` |
| `SessionTimeout` | Oturum zaman aşımı (saniye) / Session timeout (seconds) | `300` |

### HTTPS Yapılandırması / HTTPS Configuration (Üretim / Production)

Güvenli bağlantı için HTTPS kullanın:

```bash
# Self-signed sertifika oluşturun / Create self-signed certificate
dotnet dev-certs https --trust

# appsettings.json'da HTTPS portunu ayarlayın / Set HTTPS port in appsettings.json
```

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://localhost:5555",
        "Certificate": {
          "Path": "certificate.pfx",
          "Password": "your-password"
        }
      }
    }
  }
}
```

---

## Kullanım / Usage

### Agent'ı Başlatma / Starting the Agent

#### Windows

**Manuel Başlatma:**
```cmd
cd "C:\Program Files\digimr"
DigitalSignature.Agent.exe
```

**Windows Servisi Olarak (Önerilen):**
```cmd
# Servis kurulumu
sc create DigiMRAgent binPath= "C:\Program Files\digimr\DigitalSignature.Agent.exe"
sc config DigiMRAgent start= auto
sc start DigiMRAgent

# Servis durumu kontrolü
sc query DigiMRAgent
```

#### Linux (systemd)

```bash
# Servis dosyası oluşturun: /etc/systemd/system/digimr-agent.service
sudo nano /etc/systemd/system/digimr-agent.service
```

```ini
[Unit]
Description=digimr Desktop Agent
After=network.target pcscd.service

[Service]
Type=simple
User=digimr
WorkingDirectory=/opt/digimr
ExecStart=/opt/digimr/DigitalSignature.Agent
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```

```bash
# Servisi etkinleştirin ve başlatın
sudo systemctl daemon-reload
sudo systemctl enable digimr-agent
sudo systemctl start digimr-agent

# Durum kontrolü
sudo systemctl status digimr-agent
```

#### macOS (launchd)

```bash
# Launch agent dosyası: ~/Library/LaunchAgents/com.digimr.agent.plist
nano ~/Library/LaunchAgents/com.digimr.agent.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.digimr.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Applications/digimr/DigitalSignature.Agent</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
# Launch agent'ı yükleyin
launchctl load ~/Library/LaunchAgents/com.digimr.agent.plist
launchctl start com.digimr.agent
```

### Agent Durumu Kontrolü / Check Agent Status

```bash
# Web tarayıcıda açın / Open in web browser
http://localhost:5555/health

# cURL ile test / Test with cURL
curl http://localhost:5555/health
```

**Beklenen yanıt / Expected response:**
```json
{
  "status": "Healthy",
  "timestamp": "2026-02-11T10:30:00Z"
}
```

---

## Web Entegrasyonu / Web Integration

### JavaScript Örneği / JavaScript Example

#### 1. Mevcut Tokenları Listeleme / List Available Tokens

```javascript
// Token listesini al / Get token list
async function listTokens() {
    const response = await fetch('http://localhost:5555/api/tokens', {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json'
        }
    });

    if (!response.ok) {
        throw new Error('Token listesi alınamadı / Failed to get token list');
    }

    const tokens = await response.json();
    console.log('Bulunan tokenlar / Found tokens:', tokens);
    return tokens;
}

// Örnek çıktı / Example output:
// [
//   {
//     "slotId": 0,
//     "label": "AKiS Token",
//     "manufacturer": "Akis",
//     "serialNumber": "12345678",
//     "model": "AKiS v1.0"
//   }
// ]
```

#### 2. Token Sertifikalarını Listeleme / List Token Certificates

```javascript
async function listCertificates(slotId, pin) {
    const response = await fetch('http://localhost:5555/api/certificates', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            slotId: slotId,
            pin: pin
        })
    });

    if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error || 'Sertifika listesi alınamadı / Failed to get certificates');
    }

    const certificates = await response.json();
    console.log('Sertifikalar / Certificates:', certificates);
    return certificates;
}

// Örnek çıktı / Example output:
// [
//   {
//     "index": 0,
//     "subject": "CN=Ahmet Yılmaz, O=Example Corp, C=TR",
//     "issuer": "CN=E-Tugra CA, O=E-Tugra, C=TR",
//     "serialNumber": "1234567890ABCDEF",
//     "notBefore": "2024-01-01T00:00:00Z",
//     "notAfter": "2027-01-01T00:00:00Z"
//   }
// ]
```

#### 3. PDF Belgesine İmza Atma / Sign PDF Document

```javascript
async function signPdf(pdfFile, slotId, pin, certificateIndex = 0) {
    // PDF'i Base64'e çevir / Convert PDF to Base64
    const pdfBase64 = await fileToBase64(pdfFile);

    const response = await fetch('http://localhost:5555/api/sign', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            documentBase64: pdfBase64,
            format: 'PAdES',  // CAdES, PAdES, XAdES
            level: 'B-LT',    // BES, B-T, B-LT, B-LTA
            slotId: slotId,
            pin: pin,
            certificateIndex: certificateIndex,
            tsaUrl: 'http://tsa-test.kamusm.gov.tr',
            signatureParameters: {
                reason: 'Belge onayı / Document approval',
                location: 'İstanbul, Türkiye',
                contactInfo: 'info@example.com',
                visible: true,
                visualSignature: {
                    page: 1,
                    x: 100,
                    y: 100,
                    width: 200,
                    height: 80,
                    stampText: 'Onaylandı / Approved'
                }
            }
        })
    });

    if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error || 'İmza başarısız / Signature failed');
    }

    const result = await response.json();

    // İmzalı PDF'i indir / Download signed PDF
    const signedPdfBlob = base64ToBlob(result.signedDocumentBase64, 'application/pdf');
    downloadFile(signedPdfBlob, 'signed-document.pdf');

    return result;
}

// Yardımcı fonksiyonlar / Helper functions
function fileToBase64(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => {
            const base64 = reader.result.split(',')[1];
            resolve(base64);
        };
        reader.onerror = reject;
        reader.readAsDataURL(file);
    });
}

function base64ToBlob(base64, mimeType) {
    const byteCharacters = atob(base64);
    const byteNumbers = new Array(byteCharacters.length);
    for (let i = 0; i < byteCharacters.length; i++) {
        byteNumbers[i] = byteCharacters.charCodeAt(i);
    }
    const byteArray = new Uint8Array(byteNumbers);
    return new Blob([byteArray], { type: mimeType });
}

function downloadFile(blob, filename) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}
```

#### 4. EYP Paketi Oluşturma / Create EYP Package

```javascript
async function createEyp(ustYaziPdf, ekler, slotId, pin) {
    const response = await fetch('http://localhost:5555/api/eyp/create', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            ustYaziBase64: await fileToBase64(ustYaziPdf),
            ekler: await Promise.all(ekler.map(async (file) => ({
                fileName: file.name,
                contentBase64: await fileToBase64(file),
                mimeType: file.type
            }))),
            ustveri: {
                belgeTarihi: new Date().toISOString(),
                belgeId: 'DOC-2026-0001',
                konusu: 'Test Belgesi',
                ilgiler: [],
                ekler: ekler.map(f => ({ dosyaAdi: f.name, aciklama: f.name })),
                guvenlikKodu: 'Tasnif Dışı',
                guvenlikKoduGecerlilikTarihi: null,
                ozId: crypto.randomUUID()
            },
            slotId: slotId,
            pin: pin,
            certificateIndex: 0,
            tsaUrl: 'http://tsa-test.kamusm.gov.tr',
            signatureLevel: 'B-LT'
        })
    });

    if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error || 'EYP oluşturma başarısız / EYP creation failed');
    }

    const result = await response.json();

    // EYP paketini indir / Download EYP package
    const eypBlob = base64ToBlob(result.packageBase64, 'application/zip');
    downloadFile(eypBlob, 'package.eyp');

    return result;
}
```

### React Örneği / React Example

```jsx
import React, { useState, useEffect } from 'react';

function TokenSigner() {
    const [tokens, setTokens] = useState([]);
    const [selectedToken, setSelectedToken] = useState(null);
    const [pin, setPin] = useState('');
    const [file, setFile] = useState(null);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        loadTokens();
    }, []);

    async function loadTokens() {
        try {
            const response = await fetch('http://localhost:5555/api/tokens');
            const data = await response.json();
            setTokens(data);
        } catch (error) {
            console.error('Token listesi alınamadı:', error);
            alert('Agent çalışmıyor olabilir. Lütfen Agent\'ın çalıştığından emin olun.');
        }
    }

    async function handleSign() {
        if (!selectedToken || !pin || !file) {
            alert('Lütfen tüm alanları doldurun');
            return;
        }

        setLoading(true);
        try {
            const pdfBase64 = await fileToBase64(file);

            const response = await fetch('http://localhost:5555/api/sign', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    documentBase64: pdfBase64,
                    format: 'PAdES',
                    level: 'B-LT',
                    slotId: selectedToken.slotId,
                    pin: pin,
                    tsaUrl: 'http://tsa-test.kamusm.gov.tr'
                })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(error.error);
            }

            const result = await response.json();
            const blob = base64ToBlob(result.signedDocumentBase64, 'application/pdf');
            downloadFile(blob, 'signed-' + file.name);

            alert('İmza başarılı!');
            setPin(''); // Güvenlik için PIN'i temizle
        } catch (error) {
            console.error('İmza hatası:', error);
            alert('İmza başarısız: ' + error.message);
        } finally {
            setLoading(false);
        }
    }

    return (
        <div>
            <h2>Token ile Belge İmzalama</h2>

            <div>
                <label>Token Seçin:</label>
                <select
                    onChange={(e) => setSelectedToken(tokens[e.target.value])}
                    disabled={tokens.length === 0}
                >
                    <option value="">-- Seçiniz --</option>
                    {tokens.map((token, index) => (
                        <option key={index} value={index}>
                            {token.label} ({token.serialNumber})
                        </option>
                    ))}
                </select>
            </div>

            <div>
                <label>PIN:</label>
                <input
                    type="password"
                    value={pin}
                    onChange={(e) => setPin(e.target.value)}
                    placeholder="Token PIN kodu"
                />
            </div>

            <div>
                <label>PDF Dosyası:</label>
                <input
                    type="file"
                    accept=".pdf"
                    onChange={(e) => setFile(e.target.files[0])}
                />
            </div>

            <button
                onClick={handleSign}
                disabled={loading || !selectedToken || !pin || !file}
            >
                {loading ? 'İmzalanıyor...' : 'İmzala'}
            </button>
        </div>
    );
}

// Yardımcı fonksiyonlar (yukarıdaki ile aynı)
function fileToBase64(file) { /* ... */ }
function base64ToBlob(base64, mimeType) { /* ... */ }
function downloadFile(blob, filename) { /* ... */ }

export default TokenSigner;
```

---

## Güvenlik / Security

### Lokal Mod Güvenliği / Local Mode Security

Lokal modda Agent sadece `localhost` (127.0.0.1) üzerinden erişilebilir:

```json
{
  "Agent": {
    "Mode": "Local",
    "AllowedOrigins": [
      "http://localhost:3000",
      "http://127.0.0.1:3000"
    ]
  }
}
```

**Güvenlik Özellikleri:**
- ✅ PIN kodu asla loglanmaz veya saklanmaz
- ✅ Oturum süresi dolduğunda otomatik kapanır
- ✅ CORS koruması ile sadece izin verilen origin'lerden erişim
- ✅ HTTPS desteği (üretim için önerilen)
- ✅ Rate limiting (DDoS koruması)

### Uzak Mod Güvenliği / Remote Mode Security

Uzak modda (sunucu erişimi) ek güvenlik önlemleri:

```json
{
  "Agent": {
    "Mode": "Remote",
    "RequireApiKey": true,
    "ApiKey": "your-secure-api-key-here",
    "AllowedOrigins": [
      "https://myapp.example.com"
    ]
  },
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:5555"
      }
    }
  }
}
```

**API Key Kullanımı:**
```javascript
const response = await fetch('https://agent.example.com:5555/api/sign', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-Api-Key': 'your-secure-api-key-here'
    },
    body: JSON.stringify({ /* ... */ })
});
```

### Firewall Kuralları / Firewall Rules

#### Windows
```cmd
# Port 5555'i localhost için aç
netsh advfirewall firewall add rule name="DigiMR Agent" dir=in action=allow protocol=TCP localport=5555 profile=private
```

#### Linux (ufw)
```bash
# Sadece localhost erişimi
sudo ufw allow from 127.0.0.1 to any port 5555

# Tüm ağdan erişim (uzak mod için)
sudo ufw allow 5555/tcp
```

---

## Sorun Giderme / Troubleshooting

### 1. Agent Başlamıyor / Agent Won't Start

**Sorun:** Port zaten kullanımda
**Çözüm:**
```bash
# Windows - Port 5555'i kullanan process'i bul
netstat -ano | findstr :5555
taskkill /PID <process-id> /F

# Linux/macOS
sudo lsof -ti:5555 | xargs kill -9
```

**Sorun:** .NET Runtime bulunamadı
**Çözüm:**
```bash
# Runtime versiyonunu kontrol edin
dotnet --list-runtimes

# .NET 8.0 Runtime yükleyin
# Windows: winget install Microsoft.DotNet.Runtime.8
# Linux: sudo apt-get install dotnet-runtime-8.0
# macOS: brew install dotnet@8
```

### 2. Token Bulunamıyor / Token Not Found

**Sorun:** Token listesi boş geliyor
**Çözüm:**
```bash
# Token sürücüsünün yüklü olduğunu kontrol edin
# Windows: Device Manager > Smart Card Readers
# Linux: pcsc_scan
# macOS: system_profiler SPUSBDataType

# PKCS#11 kütüphane yolunu kontrol edin
# Windows: C:\Windows\System32\akisp11.dll (AKiS örneği)
# Linux: /usr/lib/opensc-pkcs11.so
# macOS: /Library/Frameworks/eToken.framework/eToken
```

### 3. PIN Hatası / PIN Error

**Sorun:** "Incorrect PIN" hatası
**Çözüm:**
- PIN kodunun doğru olduğundan emin olun
- Token kilitlenmiş olabilir (PUK ile açmanız gerekebilir)
- Büyük/küçük harf duyarlılığını kontrol edin

### 4. İmza Doğrulama Hatası / Signature Verification Error

**Sorun:** İmza geçersiz görünüyor
**Çözüm:**
```bash
# Sertifika zincirini kontrol edin
# TSA URL'sinin doğru olduğundan emin olun
# Saat senkronizasyonunu kontrol edin

# Windows
w32tm /resync

# Linux
sudo ntpdate -s time.nist.gov

# macOS
sudo sntp -sS time.apple.com
```

### 5. CORS Hatası / CORS Error

**Sorun:** Web uygulaması "CORS policy" hatası veriyor
**Çözüm:**

`appsettings.json` dosyasına web uygulamanızın URL'sini ekleyin:
```json
{
  "Agent": {
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://myapp.example.com"
    ]
  }
}
```

### 6. Log Dosyalarını İnceleme / Check Log Files

```bash
# Windows
type "C:\Program Files\digimr\logs\agent-20260211.log"

# Linux
tail -f /opt/digimr/logs/agent-20260211.log

# macOS
tail -f /Applications/digimr/logs/agent-20260211.log
```

### 7. Debug Modu / Debug Mode

Daha detaylı log için `appsettings.json`:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "DigitalSignature": "Trace"
    }
  }
}
```

---

## Destek / Support

### Dokümantasyon / Documentation
- GitHub: https://github.com/moreum-tech/digimr
- API Referansı: http://localhost:5555/swagger (Agent çalışırken)

### Sorun Bildirme / Report Issues
GitHub Issues: https://github.com/moreum-tech/digimr/issues

### Topluluk / Community
- Discussions: https://github.com/moreum-tech/digimr/discussions
- Email: support@moreum.tech

---

## Versiyon Geçmişi / Version History

### v1.0.0 (Şubat 2026)
- ✅ İlk stabil sürüm / Initial stable release
- ✅ USB Token/Akıllı Kart desteği / USB Token/Smart Card support
- ✅ CAdES, PAdES, XAdES imza formatları / CAdES, PAdES, XAdES signature formats
- ✅ EYP 1.3, 2.0, 2.1 paket desteği / EYP 1.3, 2.0, 2.1 package support
- ✅ Web entegrasyonu / Web integration
- ✅ Güvenlik güncellemeleri / Security updates (582+ tests)

---

## Lisans / License

Bu yazılım MIT lisansı altında dağıtılmaktadır.
This software is distributed under the MIT license.

Copyright © 2026 Moreum Tech
