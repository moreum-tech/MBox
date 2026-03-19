# DigiMR — Online İmza Doğrulama Ortamları

Üretilen örnek PDF'leri (`C:\Users\bolat\AppData\Local\Temp\digimr-test\examples\`) aşağıdaki
araçlarla doğrulayabilirsiniz.

---

## 1. DSS Demo — Birincil ETSI Validator

**URL**: https://ec.europa.eu/digital-building-blocks/DSS/webapp-demo/validation

**Amaç**: PAdES/CAdES/XAdES format ve seviye doğrulaması (ETSI EN 319 102-1)

### Adımlar

1. URL'ye gidin → "Validate a signature" sekmesini seçin
2. "Upload a signed document" → PDF dosyasını yükleyin
3. "Validate" düğmesine tıklayın

### Test sertifikasi ile beklenen sonuclar

| Alan | Değer | Açıklama |
|------|-------|----------|
| **Signature format** | `PAdES-BASELINE-B` | ✅ Artık `PDF-NOT-ETSI` değil |
| **signing-time** | ✅ Present | ✅ PadesAttrGenerator fix ile çözüldü |
| **signing-certificate-v2** | ✅ Present | ✅ |
| **Qualification** | `INDETERMINATE` | Self-signed cert, AATL/EUTL'de değil |
| **Sub indication** | `NO_CERTIFICATE_CHAIN_FOUND` | Beklenen — test sertifikası |
| **Timestamp** | `INDETERMINATE` | Test ortaminda test TSA kullanildi |
| **Crypto** | ✅ PASSED | İmza matematiksel olarak doğru |

### Gerçek eToken sertifikasıyla beklenen sonuçlar

| Alan | Değer |
|------|-------|
| **Signature format** | `PAdES-BASELINE-B` veya `PAdES-BASELINE-T` |
| **Qualification** | `INDETERMINATE` (E-Güven, TÜRKTRUST EUTL'de yok) |
| **Timestamp** | `PASSED` (Kamu SM TSA ile) |
| **Crypto** | ✅ PASSED |

> **Not**: Türk CA'lar (E-Güven, Kamu SM, TÜRKTRUST) Adobe AATL veya AB EUTL'de yer almaz.
> Bu nedenle `INDETERMINATE/NO_CERTIFICATE_CHAIN_FOUND` beklenen bir sonuçtur.
> **5070 sayılı Kanun kapsamında imza hukuken tam geçerlidir.**

---

## 2. DSS Demo — Trusted List Ekleme (Gelişmiş)

Türk sertifikasını PASSED yapmak için kendi trust store'unuzu ekleyebilirsiniz:

1. DSS Demo → "Validation Settings"
2. "Custom Trusted Lists" → Türk CA sertifikasını PEM formatında yükleyin
3. Yeniden doğrulayın → Qualification TOTAL_PASSED olabilir

**Not**: Bu sadece DSS Demo için local bir ayar, üretim sistemlerde
`TrustedCertificateStore` sınıfı aynı işlevi görür.

---

## 3. VeraPDF — PDF/A ve İmza Yapı Kontrolü

**URL**: https://demo.verapdf.org/

**Amaç**: PDF yapısal doğrulama, standart uyum

1. "Upload file" → imzalı PDF
2. "Validate" → PDF/A profili seçin
3. Sonuçlar: ByteRange bütünlüğü, imza yapısı

---

## 4. PDF Tools Validator (iText DITO)

**URL**: https://itextpdf.com/products/itext-dito

Kurumsal PDF imza doğrulama — ücretsiz deneme mevcut.

---

## 5. Türkiye'ye Özgü Validators

| Sağlayıcı | URL | Format |
|-----------|-----|--------|
| E-Güven | https://eguven.com | PAdES, CAdES |
| TÜRKTRUST | https://www.turktrust.com.tr | PAdES, XAdES |
| Kamu SM | https://kamusm.bilgem.tubitak.gov.tr | PAdES, CAdES, KEP |
| e-İmzatr | https://www.eimzatr.com | PAdES, CAdES |

> **Not**: Bu servislerin çoğu Türk CA sertifikalarını güvenir olarak kabul eder.
> Gerçek NES ile imzalanan belgeler burada VALID/GEÇERLI sonuç verecektir.

---

## 6. Adobe Acrobat (Yerel)

Adobe Acrobat Reader / Pro ile imzalı PDF'i açın:

- **"Signature is UNKNOWN"** görürseniz: Türk CA Adobe AATL'de değil (beklenen)
- **"Signature is VALID"** görürseniz: Sertifika AATL'e eklenmiş

Adobe'de Türk CA'yı güvenilir yapmak:
1. Acrobat → Preferences → Trust Manager
2. "European Union Trusted Lists (EUTL)" yerine Türk CA sertifikasını manuel ekleyin

---

## 7. Üretilen Örnek Dosyalar

Konum: `C:\Users\bolat\AppData\Local\Temp\digimr-test\examples\`

| Dosya | Seviye | Türk Profil | TSA |
|-------|--------|-------------|-----|
| `pades_b_b.pdf` | B-B | P1 karşılığı | Yok |
| `pades_b_t.pdf` | B-T | P2 karsiligi | Test TSA |
| `pades_b_lt.pdf` | B-LT | P3/P4 karsiligi | Test TSA + DSS |
| `pades_b_lta.pdf` | B-LTA | Arsiv | Test TSA + DSS + ATS |
| `pades_turkey_p1.pdf` | B-B | **P1** | Yok |
| `pades_turkey_p2.pdf` | B-T | **P2** (EPES) | Test TSA |
| `pades_turkey_p3.pdf` | B-LT | **P3** (EPES) | Test TSA + DSS |
| `pades_turkey_p4.pdf` | B-LT | **P4** (EPES, OCSP) | Test TSA + DSS |

Yeniden üretmek için:
```bash
dotnet test tests/DigitalSignature.Tests --filter "Category=ExampleGeneration" -c Release
```

### DSS Demo'da kontrol edilecekler

`pades_b_b.pdf` yükleyince şunları görmelisiniz:
1. `Signature format: PAdES-BASELINE-B` ✅
2. `signing-time: Present` ✅ (önceki fix ile)
3. `signing-certificate-v2: Present` ✅
4. `Qualification: INDETERMINATE` — beklenen (self-signed cert)

`pades_b_t.pdf` için ek olarak:
5. `Signature timestamp: Present` ✅ (test TSA token)

---

## 8. eToken ile Gerçek Sertifika Testi

Gerçek NES sertifikası üretmek için `ETokenSigningIntegrationTests` kullanın:

```bash
dotnet test tests/DigitalSignature.Tests --filter "Category=Integration&Requires=eToken" -c Release
```

Çıktı: `C:\Users\bolat\AppData\Local\Temp\digimr-test\muhasebe_kaydi_imzali.pdf`

Bu dosyayı DSS Demo'ya yüklerseniz:
- `PAdES-BASELINE-B` formatı görürsünüz ✅
- `signing-time: Present` ✅
- Qualification: `INDETERMINATE` (E-Güven EUTL'de yok, beklenen)
- Kamu SM TSA ile: Timestamp `PASSED` ✅

---

*DigiMR tum imza formatlari (PAdES, CAdES, XAdES, JAdES) DSS Demo'da dogrulanabilir.*
