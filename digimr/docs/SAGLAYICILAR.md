# Imza Saglayicilari (Signing Providers)

## Genel Bakis (Overview)

8 farkli provider destegi. Hepsi `ISigningProvider` interface'ini uygular.

| Provider | Tip | Kullanim Alani |
|----------|-----|----------------|
| SoftwareCertificateProvider | PKCS#12 | Test, gelistirme, yazilim sertifikasi |
| TokenSigningProvider | PKCS#11 | Akilli kart, USB token |
| HsmSigningProvider | PKCS#11 HSM | Kurumsal yuksek hacimli imzalama |
| CloudHsmSigningProvider | Cloud KMS | Azure Key Vault, AWS KMS, Google Cloud KMS |
| MobileSigningProvider | Mobil MSSP | Turkcell, Vodafone, Turk Telekom (gercek operator) |
| RemoteTokenSigningProvider | HTTP Agent | Uzaktan token erisimi |
| ESealProvider | PKCS#12 | Elektronik muhur (sunucu tarafli) |
| BiometricSigningProvider | PKCS#12 + Biyometri | Biyometrik cihaz ile imza |

---

## 1. Software Certificate Provider

### Aciklama

PKCS#12 (.pfx/.p12) dosyasi ile imzalama. Test ve gelistirme icin ideal.

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Software,
    new AuthenticationContext
    {
        CertificatePath = "certificate.pfx",
        CertificatePassword = "password"
        // veya
        // CertificateData = File.ReadAllBytes("certificate.pfx")
    });
```

### AuthenticationContext Alanlari

| Alan | Tip | Zorunlu | Aciklama |
|------|-----|---------|----------|
| CertificatePath | string | * | PFX dosya yolu |
| CertificateData | byte[] | * | PFX binary veri |
| CertificatePassword | string | Hayir | PFX sifresi |

\* CertificatePath veya CertificateData'dan biri gerekli

### Guvenlik

- Sifre bellekten sifirlanir (Dispose)
- Ozel anahtar bellekten sifirlanir
- RSA ve ECDSA destegi

---

## 2. Token/SmartCard Provider

### Aciklama

PKCS#11 uyumlu akilli kart ve USB token ile imzalama.

### Desteklenen Cihazlar

- SafeNet eToken
- AKiS (Turkiye milli kart)
- Gemalto IDPrime
- Oberthur
- Bit4id

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Token,
    new AuthenticationContext
    {
        Pin = "123456",
        // Opsiyonel - otomatik kesif yapilir
        Pkcs11LibraryPath = @"C:\Windows\System32\akisp11.dll",
        SlotId = 0,
        TokenFilter = "AKiS"  // label/seri no/uretici ile filtreleme
    });
```

### Token Cache

- Otomatik token kesfi (TokenDiscoveryService)
- Cache dosyasi: `%LOCALAPPDATA%\DigiMR\token-cache.json`
- 7 gun gecerli, DLL+token erisilebilirlik kontrolu

---

## 3. HSM Provider

### Aciklama

Hardware Security Module ile kurumsal imzalama. Session pooling ve 3 PIN modu.

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.HSM,
    new AuthenticationContext
    {
        Pin = "hsmpin",
        Pkcs11LibraryPath = @"C:\hsm\pkcs11.dll",
        SlotId = 0,
        Metadata = new Dictionary<string, string>
        {
            ["PinMode"] = "SessionStart",  // SessionStart | PerSignature | Cached
            ["HsmPoolSize"] = "5",
            ["PinCacheTtlSeconds"] = "300"
        }
    });
```

### PIN Modlari

| Mod | Aciklama |
|-----|----------|
| SessionStart | Oturum basinda PIN girilir (varsayilan) |
| PerSignature | Her imza icin login/sign/logout |
| Cached | PIN TTL ile onbelleklenir |

### Session Pool

- MinSessions: 1
- MaxSessions: konfigüre edilebilir (HsmPoolSize)
- IdleTimeout: 5 dakika
- MaxLifetime: 1 saat

---

## 3b. Cloud HSM Provider

### Aciklama

Bulut tabanli HSM servisleri ile imzalama: Azure Key Vault, AWS KMS, Google Cloud KMS.

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.CloudHSM,
    new AuthenticationContext
    {
        Metadata = new Dictionary<string, string>
        {
            ["CloudProvider"] = "AzureKeyVault",  // AzureKeyVault | AwsKms | GoogleCloudKms
            ["KeyVaultUrl"] = "https://myvault.vault.azure.net",
            ["KeyName"] = "signing-key",
            ["TenantId"] = "...",
            ["ClientId"] = "...",
            ["ClientSecret"] = "..."
        }
    });
```

### Desteklenen Bulut Servisleri

| Servis | Anahtar Turleri |
|--------|----------------|
| Azure Key Vault | RSA 2048/4096, EC P-256/P-384 |
| AWS CloudHSM / KMS | RSA, ECDSA |
| Google Cloud KMS | RSA, ECDSA |

---

## 4. Mobile Signature Provider

### Aciklama

Gercek operatör MSSP (ETSI TS 102 204) uzerinden mobil imzalama. Kullanici telefonunda SIM PIN ile onaylar.

### Desteklenen Operatorler

| Operator | Durum |
|----------|-------|
| Turkcell | Test edildi |
| Vodafone | Kod hazir, operator testi bekliyor |
| Turk Telekom | Kod hazir, operator testi bekliyor |

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905551234567",
        Metadata = new Dictionary<string, string>
        {
            ["MobileOperator"] = "Turkcell"  // Turkcell | Vodafone | TurkTelekom
        }
    });
```

Operator AP kimlikleri `appsettings.json` icindeki `MobileSignature` bolumunde yapilandirilir.
Detay icin [MOBILE_SIGNATURE.md](MOBILE_SIGNATURE.md) bakiniz.

### Akis

1. `AuthenticateAsync` -- Operator gateway'e baglanti
2. `SignHashAsync` -- Hash operator'e gonderilir, kullanici telefonunda onaylar
3. Operator imzali sonucu doner (~20-60 saniye)

---

## 5. Remote Token Provider

### Aciklama

Token Agent uzerinden uzaktan PKCS#11 token erisimi. Hash-only mod: belge asla agent'a gitmez.

### Gereksinimler

- Token Agent (port 5555) calisir durumda olmali
- Kurulum: [AGENT_SETUP.md](AGENT_SETUP.md)

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.RemoteToken,
    new AuthenticationContext
    {
        AgentUrl = "http://localhost:5555",
        Pin = "123456",
        SlotId = 0,
        CertificateIndex = 0,
        AgentApiKey = "optional-api-key"
    });
```

### Guvenlik

- Yalnizca hash gonderilir (belge icerigi asla gitmez)
- Opsiyonel API Key (X-Api-Key header)
- 30 saniye istek zaman asimi
- PIN bellekten sifirlanir

---

## 6. E-Seal Provider

### Aciklama

Sunucu tarafli otomatik muhur. Kullanici etkilesimi gerektirmez. Tuzel kisilik icin.

### Konfigürasyon

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.ESeal,
    new AuthenticationContext
    {
        CertificatePath = "seal-certificate.pfx",
        CertificatePassword = "sealpassword"
    });
```

### Ozellikler

- ETSI EN 319 411-1 QCP-l-qscd politikasi
- Otomatik muhur (batch islemler icin ideal)
- E-Seal QcStatement OID: 0.4.0.19122.1.2

---

## 7. Biometric Provider

### Aciklama

Biyometrik cihaz (imza pad, parmak izi) ile imza. Biyometrik veri imza kaniti olarak saklanir.

### Konfigürasyon

```csharp
// Cihaz ile
var device = new MyBiometricDevice(); // IBiometricDevice uygulayan sinif
var provider = new BiometricSigningProvider(device);

await provider.AuthenticateAsync(new AuthenticationContext
{
    CertificatePath = "biometric-cert.pfx",
    CertificatePassword = "password"
});

// veya cihaz olmadan (onceden alinan veri ile)
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Biometric,
    new AuthenticationContext
    {
        CertificatePath = "biometric-cert.pfx",
        CertificatePassword = "password",
        BiometricData = capturedData
    });
```

---

## Provider Karsilastirma Tablosu

| Ozellik | Software | Token | HSM | CloudHSM | Mobile | Remote | ESeal | Biometric |
|---------|----------|-------|-----|----------|--------|--------|-------|-----------|
| Anahtar Yeri | Dosya | Cihaz | HSM | Bulut | SIM | Cihaz | Dosya | Dosya |
| Kullanici Etkilesimi | Hayir | PIN | PIN | Hayir | Telefon | PIN | Hayir | Cihaz |
| Kurumsal | Hayir | Evet | Evet | Evet | Hayir | Evet | Evet | Hayir |
| Test/Gelistirme | Evet | - | - | - | - | - | - | - |
| SHA256/384/512 | Evet | Evet | Evet | Evet | Evet | Evet | Evet | Evet |
| SHA3 | Evet | Evet | Evet | Evet | Evet | Evet | Evet | Evet |
| RSA + ECDSA | Evet | RSA | RSA | Evet | RSA | RSA | Evet | Evet |
