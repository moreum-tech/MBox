# Tablet Relay-Sign API — Entegrasyon Rehberi

DigiMR imza tabletine iş göndermek için tek endpoint kullanılır:

```
POST http://<tablet-ip>:7777/api/v1/relay-sign
Content-Type: application/json
```

> Ağ üzerinden erişim tablet tarafında `DIGIMR_TABLET_NETWORK=1` ile açılmış olmalıdır;
> aksi halde endpoint yalnız tabletin kendi içinden (127.0.0.1) çağrılabilir.

Çalışma modeli: Çağıran uygulama, **alıcının beklediği opak JSON payload'ını** gönderir.
Tablet bu payload'ın içeriğini bilmek zorunda değildir — yalnızca `_digimr...` işaretçili
düğümleri işler. İmza tamamlanınca **aynı payload**, imzalı PDF'ler yerine yazılmış ve
giriş işaretçileri silinmiş olarak `callbackUrl`'e POST edilir.

---

## 1. En kapsamlı örnek istek

```jsonc
{
  "callbackUrl": "https://uygulamam.example.com/api/imza-sonuc",
  "callbackAuthHeader": "X-Api-Key",
  "callbackAuthValue": "gizli-anahtar-123",
  "contentField": "content",
  "signatureMethod": "handwritten",
  "handwrittenVerification": "nfc",
  "addTimestamp": true,
  "tsaProviderKey": "turktrust",

  "payload": [
    {
      "_digimrIdentity": {
        "tcKimlikNo": "10000000146",
        "adSoyad": "TEST KULLANICI",
        "required": true
      }
    },
    {
      "id": "SOZ-2026-0042",
      "name": "hizmet-sozlesmesi.pdf",
      "klasorNo": "K-17",
      "content": "JVBERi0xLjcK... (base64 PDF)",
      "_digimrType": { "key": "sozlesme", "label": "Sözleşme", "required": true, "allowAdd": false },
      "_digimrSign": {
        "signerName": "TEST KULLANICI",
        "placements": [
          { "page": "all",  "x": 340, "y": 90, "width": 170, "height": 60 },
          { "page": "last", "x": 60,  "y": 90, "width": 170, "height": 60 }
        ],
        "footers": [
          { "page": "all", "text": "Bu belge DigiMR tablet üzerinde elden imzalanmıştır.",
            "align": "center", "y": 12, "fontSize": 8 },
          { "page": "last", "text": "Sözleşme No: SOZ-2026-0042", "align": "left", "y": 24, "fontSize": 7 }
        ]
      },
      "_digimrIndex": {
        "fields": [
          { "name": "sozlesmeNo", "label": "Sözleşme No", "type": "text", "editable": false, "value": "SOZ-2026-0042" },
          { "name": "aciklama",   "label": "Açıklama",    "type": "text", "editable": true,  "value": "" }
        ]
      }
    },
    {
      "id": "EK-2026-0042-1",
      "name": "aydinlatma-metni.pdf",
      "content": "JVBERi0xLjcK... (base64 PDF)",
      "_digimrType": { "key": "ek", "label": "Ek Belge" },
      "_digimrSign": {
        "placements": [
          { "bookmarkName": "IMZA_ALANI", "width": 170, "height": 60 }
        ]
      }
    },
    {
      "id": "BILGI-2026-0042",
      "name": "tarife-bilgilendirme.pdf",
      "content": "JVBERi0xLjcK... (base64 PDF)",
      "_digimrSign": { "show": true }
    },
    {
      "_digimrType": { "key": "kimlik-fotokopi", "label": "Kimlik Fotokopisi", "allowAdd": true, "required": false }
    }
  ]
}
```

Başarılı yanıt:

```json
HTTP 201
{ "sessionId": "5aeb331baa384e8abd013ea3aa22af92",
  "uiUrl": "http://127.0.0.1:7777/sign-app/?mode=push" }
```

Hatalı istek `HTTP 400 { "error": "..." }` döner (geçersiz base64, bilinmeyen TSA
sağlayıcısı, bulunamayan bookmark, belge limiti aşımı vb. iş **oluşturulmadan** reddedilir).

---

## 2. Üst düzey alanlar

| Alan | Zorunlu | Açıklama |
|---|---|---|
| `callbackUrl` | **Evet** | Sonucun POST edileceği mutlak http/https adres. Hem imzalı paket hem `_digimrOutcome` bildirimleri buraya gider. |
| `callbackAuthHeader` / `callbackAuthValue` | Hayır | Verilirse her callback isteğine bu HTTP başlığı eklenir (ör. `X-Api-Key: gizli-anahtar-123`). |
| `contentField` | Hayır | Base64 PDF'in bulunduğu kardeş alanın adı. Varsayılan `"content"`. Alıcınız PDF'i başka alanda bekliyorsa (ör. `"data"`) burada bildirin. |
| `signatureMethod` | Hayır | `"handwritten"` (el imzası, varsayılan) \| `"token"` (eToken e-imza) \| `"mobile"` (mobil imza). |
| `phoneNumber` | Koşullu | `signatureMethod:"mobile"` için zorunlu; `handwrittenVerification:"sms-otp"` için zorunlu. |
| `handwrittenVerification` | Hayır | El imzasında kimlik kapısı: `"none"` \| `"sms-otp"` \| `"mrz"` \| `"nfc"`. Gönderilmezse tabletin varsayılan ayarı geçerlidir. `"nfc"` seçilirse çip okunamadığında operatör MRZ kamera yedeğine düşebilir. |
| `addTimestamp` | Hayır | `true` → imzalı PDF'e RFC 3161 zaman damgası (DocTimeStamp) eklenir. Gönderilmezse tablet varsayılanı. |
| `tsaProviderKey` | Hayır | Zaman damgası sağlayıcı anahtarı (tablette tanımlı olmalı). Bilinmeyen/kapalı anahtar istek anında 400 ile reddedilir. Boşsa tablet varsayılan sağlayıcısı. |
| `payload` | **Evet** | Alıcıya olduğu gibi iletilecek opak JSON (genellikle dizi). İçindeki `_digimr...` işaretçileri aşağıda. |

---

## 3. Payload işaretçileri

### 3.1 `_digimrSign` — imzalat / göster

Bir düğüme `_digimrSign` konursa tablet o düğümün `contentField` alanındaki base64 PDF'i işler.

- `placements` **varsa** → belge imzalanır (ve gösterilir).
- `placements` **yoksa** → belge yalnız gösterilir; `"show": false` ile tamamen gizlenir.
- `signerName` → imza görünümünde yazılacak ad soyad. `_digimrIdentity.adSoyad` ile
  **çelişirse iş reddedilir** (tek ad soyad gönderin).

Her `placements[]` elemanı:

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `page` | `1` | Sayı (`3`) veya seçici string. Seçiciler: `"first"`, `"last"`, `"last-N"` (sondan N önce), `"all"`, `"2-4"` (aralık), `"1,3,last"` (liste, virgülle birleştirilebilir). Çoklu seçici aynı koordinatla her sayfaya ayrı imza yerleştirir. |
| `x`, `y` | `50, 50` | Sol-alt köşe referanslı PDF koordinatı (punto). |
| `width`, `height` | `200, 80` | İmza kutusu boyutu (punto). |
| `bookmarkName` | — | PDF yer imi ile konumlama. Verilirse `page/x/y`'yi **geçersiz kılar**; yer imi bulunamazsa iş 400 ile reddedilir. |

`footers[]` — imzadan bağımsız sayfa alt notları:

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `page` | tüm sayfalar | Aynı sayfa seçici sözdizimi. |
| `text` | — (zorunlu) | Basılacak metin. |
| `align` | `"center"` | `"left"` \| `"center"` \| `"right"`. |
| `y` | `12` | Alt kenardan yükseklik (punto). |
| `fontSize` | `8` | Yazı boyutu. |

### 3.2 `_digimrIdentity` — beklenen imzacı kimliği

Payload'ın herhangi bir yerinde **bir kez** bulunur:

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `tcKimlikNo` | — | Beklenen TC kimlik no; doğrulama sonucunda eşleşme kontrol edilir. |
| `adSoyad` | — | Beklenen ad soyad. Kimlik istenen işte **zorunlu** (imza görünümüne de yazılır). |
| `required` | `true` | `true` → kimlik doğrulanmadan imza atılamaz (sert kapı). |

### 3.3 `_digimrType` — belge tipleri ve tablette belge ekleme

| Alan | Varsayılan | Açıklama |
|---|---|---|
| `key` | — (zorunlu) | Tip anahtarı (tekil; aynı key'in ilk tanımı geçerli). |
| `label` | `key` | Operatör ekranındaki görünen ad. |
| `allowAdd` | `true` | Operatör tablette bu tipe yeni belge (kamera/tarama) ekleyebilir mi. |
| `required` | `false` | `true` → bu tipten belge eklenmeden iş gönderilemez. |

**İçeriği boş** bir tip düğümü (örnekteki `kimlik-fotokopi`) şablon görevi görür:
operatör tablette o tipe belge ekleyince PDF **bu düğümün içine** yazılır — düğümdeki
iş alanlarınız (id, klasör no vb.) korunur. Boş düğüm yoksa payload dizisine yeni bir
düğüm eklenir. Eklenen düğümler callback'te `"_digimrAdded": true` ve
`"_digimrOrigin"` (ör. `"camera"`) ile işaretlenir.

### 3.4 `_digimrIndex` — belge indeks alanları

Belgeye bağlı üstveri formu; operatör `editable:true` alanları tablette doldurabilir.
Alanlar: `name` (zorunlu), `label`, `type` (varsayılan `"text"`), `editable`
(varsayılan `false`), `value`. Bu işaretçi **silinmez**, güncel değerlerle callback'te geri döner.

---

## 4. Callback sözleşmesi (sonuç)

### 4.1 Başarılı imza

`callbackUrl`'e gönderdiğiniz **payload'ın aynısı** POST edilir; farklar:

- Her işaretçili düğümün `contentField` alanı **imzalı PDF'in base64'ü** ile değiştirilmiştir
  (imzasız/salt-görüntüleme belgelerde orijinal içerik korunur).
- Giriş işaretçileri (`_digimrSign`, `_digimrType`, `_digimrIdentity`) **silinmiştir**.
- Kimlik doğrulaması yapıldıysa payload köküne şu düğüm eklenir:

```json
"_digimrVerification": {
  "method": "nfc",
  "tcKimlikNo": "10000000146",
  "adSoyad": "TEST KULLANICI",
  "tcMatch": true,
  "nameMatch": true,
  "verifiedAtUtc": "2026-08-12T14:03:22.0000000Z"
}
```

(Payload dizi ise diziye `{"_digimrVerification": {...}}` elemanı olarak eklenir.)

- Delil zinciri açıksa her belge düğümüne `_digimrEvidence` eklenir:

```json
"_digimrEvidence": {
  "version": 1,
  "documentHash": "c2FoYS4uLg==",
  "documentHashAlgorithm": "SHA-256",
  "biometric": "<şifreli biyometrik imza verisi>",
  "biometricEncrypted": true,
  "biometricStandard": "ISO/IEC 19794-7:2007",
  "identityEvidence": "<şifreli kimlik delili>",
  "identityEvidenceMethod": "nfc"
}
```

- Operatörün tablette eklediği belgeler `"_digimrAdded": true`, `"_digimrOrigin"` ve
  güncellenmiş `_digimrIndex` ile gelir.

### 4.2 İmzasız kapanış (iptal / red / süre dolumu)

İş imzalanmadan kapanırsa aynı `callbackUrl`'e şu gövde POST edilir — başarı
payload'ından **`_digimrOutcome` anahtarının varlığıyla** ayırt edin:

```json
{
  "_digimrOutcome": {
    "sessionId": "5aeb331baa384e8abd013ea3aa22af92",
    "outcome": "cancelled",
    "reason": "Operatör işi kapattı",
    "at": "2026-08-12T14:10:05.0000000Z"
  }
}
```

`outcome` değerleri: `"cancelled"` (operatör iptal etti) \| `"refused"` (imzacı reddetti)
\| `"expired"` (oturum süresi doldu).

### 4.3 Dikkat edilecekler

- Oturumlar **bellektedir**: tablet uygulaması kapanırsa bekleyen işler kaybolur; callback
  gelmeyen işi makul bir süre sonra yeniden gönderin.
- Callback endpoint'iniz `2xx` dönmelidir; aksi halde iletim başarısız sayılır.
- Tüm belgeler imzalanamadıysa **eksik paket iletilmez** (yarım sonuç gelmez).
