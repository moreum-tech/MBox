# DigiMR — Mobil İmza Rehberi

Türkiye'de mobil imza, **5070 sayılı Elektronik İmza Kanunu** ve **BTK Genelge 2013/6** kapsamında GSM operatörlerinin sunduğu bir hizmettir.

**Desteklenen operatörler:**

| Operatör | Durum |
|----------|-------|
| **Turkcell** | Test edildi |
| **Vodafone** | Kod hazır, operatör testi bekliyor |
| **Türk Telekom** | Kod hazır, operatör testi bekliyor |

---

## 1. Nasıl Çalışır?

DigiMR tamamen **sunucu taraflıdır** — müşteri uygulamasına (Android/iOS/web) ek SDK kurulmaz.

1. Uygulamanız DigiMR API'ye imzalama isteği gönderir (`mobileOperator` + `msisdn` belirtir)
2. DigiMR, operatörün MSSP gateway'ine bağlanır
3. Operatör, kullanıcının telefonuna onay bildirimi gönderir
4. Kullanıcı SIM kartındaki PIN'i girerek onaylar
5. Operatör imzalı sonucu DigiMR'a döner
6. DigiMR, imzalı belgeyi (PAdES/CAdES/XAdES/JAdES) oluşturup müşteri uygulamasına döner

---

## 2. appsettings.json Yapılandırması

Tüm operatörlerin yapılandırması `MobileSignature` bölümü altında yer alır.

```json
{
  "MobileSignature": {
    "Turkcell": {
      "ApId": "http://MImzaFirmaAdi",
      "ApPwd": "TURKCELL_AP_PASSWORD",
      "MessagingMode": "asynchClientServer",
      "TimeoutSeconds": 120,
      "MssFormat": "http://uri.etsi.org/TS102204/v1.1.2#XML-Signature",
      "DefaultDisplayText": "Dijital imza onayı",
      "ProfileQueryUrl": "https://msign.turkcell.com.tr/MSSP2/services/MSS_ProfileQueryPort",
      "SignatureUrl": "https://msign.turkcell.com.tr/MSSP2/services/MSS_Signature",
      "StatusQueryUrl": "https://msign.turkcell.com.tr/MSSP2/services/MSS_StatusPort",
      "SoapServiceNamespace": "http://turkcelltech.com/mobilesignature/validation/soap",
      "ProxyUrl": "http://proxy.sirket.com:8888"
    },
    "Vodafone": {
      "ApId": "VODAFONE_AP_ID",
      "ApPwd": "VODAFONE_AP_PASSWORD",
      "MessagingMode": "asynchClientServer",
      "TimeoutSeconds": 120,
      "MssFormat": "http://uri.etsi.org/TS102204/v1.1.2#XML-Signature",
      "DefaultDisplayText": "Dijital imza onayi",
      "ProfileQueryUrl": "https://msign.vodafone.com.tr/MSSP/services/MSS_ProfileQueryPort",
      "SignatureUrl": "https://msign.vodafone.com.tr/MSSP/services/MSS_SignaturePort",
      "StatusQueryUrl": "https://msign.vodafone.com.tr/MSSP/services/MSS_StatusPort",
      "SoapServiceNamespace": "http://www.vodafone.com.tr/mobilesignature/soap",
      "ProxyUrl": ""
    },
    "TurkTelekom": {
      "ApId": "TURKTELEKOM_AP_ID",
      "ApPwd": "TURKTELEKOM_AP_PASSWORD",
      "MessagingMode": "asynchClientServer",
      "TimeoutSeconds": 120,
      "MssFormat": "http://uri.etsi.org/TS102204/v1.1.2#XML-Signature",
      "DefaultDisplayText": "Dijital imza onayi",
      "ProfileQueryUrl": "https://msign.ttmobiil.com.tr/MSSP/services/MSS_ProfileQueryPort",
      "SignatureUrl": "https://msign.ttmobiil.com.tr/MSSP/services/MSS_SignaturePort",
      "StatusQueryUrl": "https://msign.ttmobiil.com.tr/MSSP/services/MSS_StatusPort",
      "SoapServiceNamespace": "http://www.ttmobiil.com.tr/mobilesignature/soap",
      "ProxyUrl": ""
    }
  }
}
```

### Yapılandırma Alanları

| Alan | Açıklama |
|------|----------|
| `ApId` | Uygulama kimliği — operatörden alınır (Turkcell'de URL formatında olabilir) |
| `ApPwd` | Uygulama şifresi — operatörden alınır |
| `ProfileQueryUrl` | MSS_ProfileQuery SOAP endpoint URL'si |
| `SignatureUrl` | MSS_Signature SOAP endpoint URL'si |
| `StatusQueryUrl` | MSS_StatusQuery SOAP endpoint URL'si (asenkron modda kullanılır) |
| `SoapServiceNamespace` | Operatöre özgü SOAP wrapper namespace'i |
| `MessagingMode` | `asynchClientServer` veya `synch` |
| `TimeoutSeconds` | Kullanıcı onayı için bekleme süresi (saniye), önerilen: 120 |
| `MssFormat` | İmza formatı URI'si (ETSI standart değeri kullanılır) |
| `DefaultDisplayText` | Kullanıcı SIM ekranında görünecek metin |
| `ProxyUrl` | Opsiyonel HTTP proxy (Turkcell IP whitelist gerektirebilir) |

---

## 3. API İstek Parametreleri

### Temel İstek

```json
POST /api/v1/sign
{
  "documentBase64": "<base64 PDF>",
  "format": "PAdES",
  "provider": {
    "type": "mobile",
    "msisdn": "905321234567",
    "mobileOperator": "Turkcell"
  },
  "parameters": {
    "level": "B_B",
    "reason": "Sözleşme onayı",
    "location": "İstanbul"
  }
}
```

### provider Alanları (type=mobile)

| Alan | Zorunlu | Açıklama |
|------|---------|----------|
| `type` | ✅ | `"mobile"` |
| `msisdn` | ✅ | Abonenin telefon numarası — uluslararası format `905XXXXXXXXX` |
| `mobileOperator` | ✅ | `"Turkcell"`, `"Vodafone"`, `"TurkTelekom"` |
| `displayText` | — | SIM ekranında gösterilecek metin (varsayılan: appsettings'ten) |

**Not:** `mobileOperator` büyük-küçük harf duyarsızdır. `"turkcell"`, `"Turkcell"`, `"TURKCELL"` eşdeğerdir. Alternatif olarak `"turktelekom"`, `"türk telekom"`, `"tt"` → `TurkTelekom` olarak çözümlenir.

### parameters Alanları

| Alan | Tür | Açıklama |
|------|-----|----------|
| `level` | string | İmza seviyesi: `B_B`, `B_T`, `B_LT`, `B_LTA` |
| `reason` | string | İmzalama nedeni (PDF meta) |
| `location` | string | İmzalama yeri |
| `contactInfo` | string | İmzalayan iletişim bilgisi |
| `hashAlgorithm` | string | `SHA256` (varsayılan), `SHA384`, `SHA512` |
| `tsaProviderKey` | string | Zaman damgası sağlayıcısı: `kamusm`, `turktrust` vb. |
| `tsaUrl` | string | Özel TSA URL'si (opsiyonel override) |
| `productionPlace` | object | `{ city, stateOrProvince, postalCode, countryName }` |
| `signerRole` | object | `{ claimedRoles: ["Müdür"] }` |
| `commitmentType` | string | `ProofOfApproval`, `ProofOfCreation` vb. |
| `signaturePolicy` | string | Türk profil politikası: `P1`–`P4` |

### Tam Örnek (PAdES B-T, mobil imza)

```json
{
  "documentBase64": "<base64 PDF>",
  "format": "PAdES",
  "provider": {
    "type": "mobile",
    "msisdn": "905321234567",
    "mobileOperator": "Turkcell",
    "displayText": "Kira sözleşmesi imzalanıyor"
  },
  "parameters": {
    "level": "B_T",
    "tsaProviderKey": "kamusm",
    "reason": "Kira sözleşmesi onayı",
    "location": "İstanbul",
    "contactInfo": "kullanici@email.com",
    "hashAlgorithm": "SHA256",
    "signerRole": { "claimedRoles": ["Kiracı"] },
    "commitmentType": "ProofOfApproval"
  }
}
```

---

## 4. Gerçek Operatör Entegrasyonu — Adım Adım

### Adım 1: Operatörle Sözleşme

Kurumsal satış kanallarından AP (Application Provider) sözleşmesi yapın:

```
Turkcell Kurumsal : https://www.turkcell.com.tr/is-cozumleri
Vodafone Kurumsal : https://www.vodafone.com.tr/kurumsal
Türk Telekom İş  : https://isbilisim.turktelekom.com.tr
```

### Adım 2: AP Kimlik Bilgilerini Al

Operatörden alınacaklar:
- `AP_ID` (Uygulama kimliği)
- `AP_PWD` (Uygulama şifresi)
- Test ve üretim endpoint URL'leri (yukarıdakiler varsayılan üretim URL'leridir)
- IP beyaz liste kaydı (gerekiyorsa)

### Adım 3: DigiMR Yapılandırması

`appsettings.Production.json` dosyasını güncelleyin:

```json
{
  "MobileSignature": {
    "Turkcell": {
      "ApId": "http://MImzaFirmaAdiniz",
      "ApPwd": "OPERATORDEN_ALINAN_SIFRE"
    }
  }
}
```

Endpoint URL'leri varsayılan olarak doğrudur; sadece değiştiyse override edin.

### Adım 4: Test

```bash
curl -X POST http://localhost:7701/api/v1/sign \
  -H "Content-Type: application/json" \
  -d '{
    "documentBase64": "JVBERi0...",
    "format": "PAdES",
    "provider": {
      "type": "mobile",
      "msisdn": "905321234567",
      "mobileOperator": "Turkcell"
    }
  }'
```

---

## 5. Çoklu Operatör Desteği

Aynı kurumsal müşteride farklı çalışanlar farklı operatörlere sahip olabilir. DigiMR her istekte `mobileOperator` parametresine göre doğru gateway'i seçer — ek yapılandırma gerekmez.

```json
// Çalışan A — Turkcell
{ "provider": { "type": "mobile", "msisdn": "905321234567", "mobileOperator": "Turkcell" } }

// Çalışan B — Vodafone
{ "provider": { "type": "mobile", "msisdn": "905421234567", "mobileOperator": "Vodafone" } }

// Çalışan C — Türk Telekom
{ "provider": { "type": "mobile", "msisdn": "905521234567", "mobileOperator": "TurkTelekom" } }
```

---

## 6. Test Ortamı

Unit test'ler test sertifikası ile çalışır (gerçek operatör bağlantısı gerekmez).
Üretimde `mobileOperator` her zaman belirtilmelidir; belirtilmezse `Turkcell` varsayılan olarak kullanılır.

---

## 7. SDK Kullanımı

```csharp
var provider = await sdk.CreateAndAuthenticateProviderAsync(
    SigningProviderType.Mobile,
    new AuthenticationContext
    {
        Msisdn = "905321234567",
        Metadata = new Dictionary<string, string>
        {
            ["MobileOperator"] = "Turkcell"
        }
    });

var parameters = new SignatureParameters
{
    Format = SignatureFormat.PAdES,
    Level = SignatureLevel.B_T,
    TsaUrl = "http://tzd.kamusm.gov.tr"
};

var result = await sdk.SignPdfWithProviderAsync("document.pdf", provider, parameters);
provider.Dispose();
```

---

## 8. Güvenlik Notları

| Konu | Önlem |
|------|-------|
| **MSISDN güvenliği** | Loglamada maskele: `0532***4567` |
| **AP Kimlik Bilgileri** | Ortam değişkeni veya secret manager kullan (`MOBILE_SIGNATURE__TURKCELL__APPWD`) |
| **TLS** | Operatör MSSP'ye TLS 1.2+ zorunludur |
| **Timeout** | 120 saniye (kullanıcı SIM'de onay için) |
| **IP Whitelist** | Turkcell gerektiriyorsa ProxyUrl veya sunucu IP kaydı |
| **Rate Limiting** | Operatör günlük kota uygulayabilir |

---

## 9. Türk Mevzuatı

| Mevzuat | Kapsam |
|---------|--------|
| **5070 sayılı Kanun** | Elektronik imzanın hukuki geçerliliği |
| **BTK Genelge 2013/6** | Mobil imza teknik şartnamesi |
| **ETSI TS 102 204** | MSSP protokol standardı |
| **ETSI EN 319 102-1** | PAdES format ve doğrulama standardı |
| **RFC 3161** | Zaman damgası protokolü |

> Türk operatörlerinin SIM kartındaki sertifikalar, Kamu SM veya BTK onaylı CA tarafından verilir.
> DSS Demo'da `INDETERMINATE` çıkması beklenen bir sonuçtur — EUTL'de kayıtlı olmadıkları için.
> Türk hukuku açısından imza **geçerlidir**.

---

## 10. Sıkça Sorulan Sorular

**S: Kullanıcının imzalaması ne kadar sürer?**
C: Kullanıcı telefona gelen mesajı görüp PIN girmesi ~20-60 saniye. DigiMR 120 saniye bekler.

**S: Her imzada kullanıcı onayı şart mı?**
C: Evet — her mobil imza işlemi için SIM kart PIN'i zorunludur (BTK zorunluluğu).

**S: Toplu imzalama mümkün mü?**
C: `POST /api/v1/sign/batch` ile birden fazla belge imzalanabilir, ancak her belge için kullanıcıdan ayrı onay alınır.

**S: Çoklu operatör aynı anda destekleniyor mu?**
C: Evet — `mobileOperator` parametresine göre Turkcell, Vodafone veya TurkTelekom gateway'i otomatik seçilir.

**S: MSISDN formatı ne olmalı?**
C: Uluslararası format: `905XXXXXXXXX` (başında `+` olmadan). Örnek: `905321234567`.

**S: Vodafone/TurkTelekom için test ortamı var mı?**
C: Operatörden test AP kimlik bilgileri ve test endpoint'i talep edilmelidir. DigiMR kod altyapısı tüm operatörler için hazırdır; yalnızca AP kimlik bilgileri ve operatör tarafı test gerekir.

**S: JAdES formatı da mobil imzayla kullanılabilir mi?**
C: Evet — mobil imza tüm formatları destekler: PAdES, CAdES, XAdES ve JAdES. `format` parametresini değiştirmeniz yeterlidir.
