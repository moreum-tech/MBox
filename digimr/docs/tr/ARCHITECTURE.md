# DigiMR Mimari

**SDK Versiyon:** 2.0 | **Son Guncelleme:** 2026-04-17

Bu dokuman DigiMR sisteminin ust seviye mimari gorunumunu, katmanlar arasi iletisimi ve tipik veri akislarini diyagramlarla anlatir.

---

## 1. 4 Katmanli Yapi

```mermaid
flowchart LR
    Client["Musteri Uygulamasi<br/>(C#, Java, Web, Mobile)"]
    DotNetSDK[".NET SDK<br/>DigitalSignature.SDK<br/>IDigitalSignatureSDK"]
    REST["REST API<br/>:7701<br/>JSON/HTTP"]
    gRPC["gRPC API<br/>:7702<br/>Protobuf/HTTP2"]
    JavaSDK["Java SDK<br/>java-native<br/>saf Java, in-process"]

    Client -->|C# kullanimi| DotNetSDK
    Client -->|HTTP/REST| REST
    Client -->|gRPC client| gRPC
    Client -->|Java kullanimi| JavaSDK

    DotNetSDK -.->|ayni process| REST
    DotNetSDK -.->|ayni process| gRPC

    REST -->|wraps| DotNetSDK
    gRPC -->|wraps| DotNetSDK

    DotNetSDK --> TSA["TSA<br/>KamuSM<br/>FreeTSA<br/>TURKTRUST"]
    DotNetSDK --> Token["PKCS#11<br/>Akilli Kart<br/>Token/HSM"]
    DotNetSDK --> TrustStore["Trust Store<br/>Guvenilir Kokler"]
    JavaSDK -->|yerel imza| TSA
    JavaSDK -->|yerel imza| Token
```

**Katmanlar:**

| Katman | Sorumluluk | Port/Kapsam |
|---|---|---|
| **Client** | Kullanici uygulamasi (EBYS, e-ticaret, mobile app) | N/A |
| **.NET SDK** | Ana kutuphane, imzalama + dogrulama mantigi | NuGet `DigitalSignature.SDK` |
| **REST API** | HTTP/JSON API — SDK'nin uzaktan erisim arayuzu | `:7701` |
| **gRPC API** | Binary protokol — yuksek performans + streaming | `:7702` |
| **Java SDK** | Saf Java in-process imzalama kutuphanesi (CAdES/PAdES/XAdES/JAdES/EYP, BouncyCastle/PDFBox) — eski gRPC/MA3 istemcisi emekliye ayrildi, release hazirlaniyor | JAR, kurumsal Java uygulamalari |

---

## 2. Tipik Imzalama Akisi (PAdES B-T)

```mermaid
sequenceDiagram
    participant C as Musteri Kodu
    participant SDK as DigitalSignatureSDK
    participant Provider as ISigningProvider<br/>(Token/HSM)
    participant TSA as TSA Sunucusu<br/>(KamuSM)

    C->>SDK: SetLicenseFile("...")
    C->>SDK: ConfigureTsa("kamusm", KamuSM(url, userId, pwd))

    C->>SDK: CreateAndAuthenticateProviderAsync(Token, ctx)
    SDK->>Provider: PKCS#11 LoadLibrary
    Provider-->>SDK: SessionHandle
    SDK->>Provider: Login(Pin)
    Provider-->>SDK: Authenticated

    C->>SDK: SignDataWithProviderAsync(pdfData, provider, params)
    SDK->>SDK: PDF hash hesapla (SHA-256)
    SDK->>Provider: Sign(hash)
    Provider-->>SDK: RawSignature

    alt Level >= B_T
        SDK->>TSA: POST /TSRequest (DER)
        TSA-->>SDK: TSResponse (TimestampToken)
    end

    SDK->>SDK: PAdES yapisini olustur<br/>(imza + sertifika zinciri + timestamp)
    SDK-->>C: SignatureResult{Success, SignedData}

    C->>Provider: Dispose()
```

**Onemli noktalar:**
- `using var provider` — provider `IDisposable`, mutlaka dispose edilmeli (token session'i kapar)
- B-B seviyesinde TSA cagrilmaz, B-T ve ustunde cagrilir
- Hash algoritmasi `SignatureParameters.HashAlgorithm` ile kontrol edilir (varsayilan SHA-256)

---

## 3. Iki Asamali Imza (Prepare/Finalize)

External signer (uzak HSM, mobil imza, merkezi imza servisi) kullanildiginda.

```mermaid
sequenceDiagram
    participant C as Musteri Kodu
    participant SDK as DigitalSignatureSDK
    participant Ext as External Signer<br/>(HSM/Mobile/Remote)
    participant TSA as TSA Sunucusu

    C->>SDK: PrepareSignatureAsync(pdfData, params, certData, chainData)
    SDK->>SDK: SignedAttributes olustur<br/>(signingCertificate, contentHash, etc)
    SDK->>SDK: SignedAttributes hash hesapla
    SDK-->>C: SignPrepareResult{Hash, HashAlgorithm, PreparedDocument}

    C->>Ext: SignHashAsync(hash, "SHA256")<br/>[HTTP/gRPC/SIM-OTP/HSM PKCS#11]
    Ext-->>C: RawSignature

    C->>SDK: FinalizeSignatureAsync(prepareResult, rawSignature)
    SDK->>SDK: Raw signature'i SignedAttributes ile kombine et
    SDK->>SDK: CMS/PAdES yapisini tamamla

    alt Level >= B_T
        SDK->>TSA: POST /TSRequest
        TSA-->>SDK: TimestampToken
    end

    SDK-->>C: SignatureResult{Success, SignedData}
```

**Kullanim alanlari:**
- **HSM:** Private key HSM'den ayrilmaz; SDK sadece hash'i imzalatir
- **Mobil imza:** Kullanici telefonunda OTP + SIM kart imzasi
- **Merkezi imza servisi:** Regulasyon geregi imza operasyonu ayri serviste

Detay: [ORNEKLER.md Senaryo 14](ORNEKLER.md#senaryo-14-iki-asamali-imza-preparefinalize--external-signer)

---

## 4. Dogrulama Akisi

```mermaid
flowchart TD
    Input["byte[] documentData"] --> Detect{"Format Tespit"}

    Detect -->|PDF| PAdES[PAdES]
    Detect -->|CMS/PKCS7| CAdES[CAdES]
    Detect -->|XML| XAdES[XAdES]
    Detect -->|JWS| JAdES[JAdES]
    Detect -->|ZIP+asic| ASiC[ASiC]
    Detect -->|ZIP+opc| OOXML[OOXML]
    Detect -->|TS 13298| EYP[EYP]

    PAdES --> Extract[Imzayi cikart]
    CAdES --> Extract
    XAdES --> Extract
    JAdES --> Extract
    ASiC --> Extract
    OOXML --> Extract
    EYP --> Extract

    Extract --> CertChain[Sertifika zincirini al]
    CertChain --> TrustCheck["Trust Store ile dogrula<br/>(AddTrustedRoot)"]
    CertChain --> Revocation["Iptal kontrolu<br/>CRL + OCSP"]
    CertChain --> TSValid["Zaman damgasi dogrula<br/>(varsa)"]

    TrustCheck --> Result
    Revocation --> Result
    TSValid --> Result["ValidationResult<br/>{IsValid, Signatures[]}"]

    Result -->|Grace period?| Mature{ValidationState}
    Mature -->|MATURE| Done([Kesin gecerli])
    Mature -->|PREMATURE| Warn([Gecici gecerli<br/>tekrar dogrula])
```

**Grace Period mantigi (`ValidationOptions.GracePeriodSeconds`):**
- 0 = grace period kullanilmaz, iptal anlik kontrol edilir
- \>0 = Imzalama zamanindan sonraki N saniye icinde iptal bildirimi olsa bile imza gecerli sayilir (ETSI EN 319 102-1 §5.6.2.3)

---

## 5. Provider Tipleri ve Yetkilendirme

```mermaid
flowchart LR
    CreateAuth["CreateAndAuthenticateProviderAsync"] --> Type{SigningProviderType}

    Type -->|Software| SW[PKCS#12 pfx/p12<br/>CertificateData + CertificatePassword]
    Type -->|Token| TK[PKCS#11 akilli kart<br/>Pin + Pkcs11LibraryPath + SlotId]
    Type -->|Mobile| MB[SIM kart imza<br/>Msisdn + Otp]
    Type -->|HSM| HS[Donanim HSM<br/>PKCS#11 + session pool]
    Type -->|CloudHSM| CH[Azure Key Vault<br/>AWS KMS / Google KMS]
    Type -->|RemoteToken| RT[Token Agent<br/>AgentUrl + AgentApiKey]
    Type -->|Biometric| BI[Imza pad / parmak izi<br/>BiometricData]
    Type -->|ESeal| ES[Elektronik muhur<br/>sunucu tarafli otomatik]

    SW --> Return[ISigningProvider]
    TK --> Return
    MB --> Return
    HS --> Return
    CH --> Return
    RT --> Return
    BI --> Return
    ES --> Return
```

Detay: [SAGLAYICILAR.md](SAGLAYICILAR.md)

---

## 6. EYP Olusturma Akisi (TS 13298 V2.0)

```mermaid
sequenceDiagram
    participant C as Musteri
    participant SDK as SDK
    participant Signer as Signer Provider
    participant Seal as Seal Provider
    participant TSA

    C->>SDK: CreateEypPackageV21Async(options, signer, seal)
    SDK->>SDK: UstYazi + Ekler + Ustveri birlestir (OPC)
    SDK->>SDK: PaketOzeti hash'le

    SDK->>Signer: Ustveri.xml'i CAdES ile imzala
    Signer-->>SDK: SignedUstveri
    SDK->>TSA: TimestampRequest
    TSA-->>SDK: TimestampToken
    SDK->>SDK: CAdES-T olustur (imza + TS)

    SDK->>SDK: NihaiUstveri olustur
    SDK->>SDK: NihaiOzet hash'le

    alt Seal provider var
        SDK->>Seal: NihaiOzet'i CAdES-A ile muhurle
        Seal-->>SDK: SealedNihaiOzet
    end

    SDK-->>C: EypCreateResult{Success, PackageData}
```

**Hukuki zemin:** TS 13298 V2.0 — Elektronik Yazisma Paketi standardi. Ustyazi ve muhur iki ayri islem; regulasyon imzacinin gercek kisi, muhurun kurum olmasini gerektirir.

---

## 7. Dağıtım Topolojileri

### 7.1 Tek Sunucu (Basit Deployment)

```mermaid
flowchart TB
    Client["Musteri Uygulamasi"]

    subgraph Server[DigiMR Sunucusu]
        API["REST + gRPC API<br/>:7701 :7702"]
        SDK[".NET SDK"]
    end

    Token["USB Token<br/>(sunucuya bagli)"]
    TSA["KamuSM TSA<br/>(internet)"]

    Client -->|HTTPS| API
    API --> SDK
    SDK -->|PKCS#11| Token
    SDK -->|HTTP| TSA
```

### 7.2 Cok Makineli (Enterprise)

```mermaid
flowchart TB
    Users["N kullanici<br/>(masaustu PC)"]

    subgraph Agent[Agent Masaustu]
        direction LR
        AgentSvc["DigiMR Agent<br/>tray + localhost:5555"]
        LocalToken["Kullanici Token'i"]
        AgentSvc --- LocalToken
    end

    subgraph Central[Merkezi Sunucu]
        API["DigiMR API<br/>:7701 :7702"]
    end

    HSM["Kurumsal HSM"]
    TSA["KamuSM TSA"]

    Users -.->|Web/App| API
    API -->|gRPC| AgentSvc
    API --> HSM
    API --> TSA
```

- **Agent:** Kullanicinin kendi token'i ile uzaktan imzalama (`QueryRemoteTokensAsync`, `RemoteToken` provider)
- **Merkezi HSM:** Kurumsal muhur/e-seal sertifikasi

---

## 8. Veri Akisi — Performans Notlari

- **Senkron imza:** PDF ~1 MB, Token: ~500ms (PIN login dahil: ilk kez ~2s)
- **Batch imza:** Tek authentication ile 50 PDF x 50ms ~= 2.5s (token session reuse)
- **HSM + session pool:** Paralel 8 worker ile 1000 PDF ~= 15s
- **gRPC vs REST:** Buyuk belgeler (>5MB) icin gRPC %30-50 daha hizli (binary format)
- **Validation:** 1 MB PDF, 1 imza ~= 200ms; CRL/OCSP online kontrol varsa +500-1000ms

Olcum: `tests/DigitalSignature.Tests/PerformanceBenchmarkTests.cs` (urun deposu), `dotnet test --filter FullyQualifiedName~PerformanceBenchmarkTests`

---

## 9. Ilgili Dokumantasyon

- [GENEL_BAKIS.md](GENEL_BAKIS.md) — Hizli navigasyon + giris
- [SDK_REFERANS.md](SDK_REFERANS.md) — SDK tam API referansi
- [API_REFERANS.md](API_REFERANS.md) — REST API tam referansi
- [CHANGELOG.md](CHANGELOG.md) — Surum gecmisi + migration
- [SAGLAYICILAR.md](SAGLAYICILAR.md) — 8 saglayici tipi detayi
