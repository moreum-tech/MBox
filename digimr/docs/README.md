[Türkçe](README.tr.md)

# DigiMR Documentation

Complete product documentation for DigiMR — electronic signature engine, REST/gRPC API and SDKs.

> **You do not need to install anything to read these.** Every document below is the
> same text that ships inside `digimr-docs.tar.gz` with each release, so you can
> evaluate what DigiMR does — API surface, signature formats, providers, deployment
> topologies — before downloading a single binary.

**Language note:** the Turkish documents are the authoritative source and are complete.
English coverage is currently limited to the overview and the changelog; every English
page links to its Turkish counterpart.

---

## Start here

| If you want to… | Read |
|---|---|
| See what DigiMR is and what it can sign | [en/OVERVIEW.md](en/OVERVIEW.md) · [tr/GENEL_BAKIS.md](tr/GENEL_BAKIS.md) |
| Decide whether it fits your use case | [tr/SENARYO_REHBERI.md](tr/SENARYO_REHBERI.md) — decision guide |
| See real code before installing | [tr/ORNEKLER.md](tr/ORNEKLER.md) — 17 worked scenarios (C# + curl) |
| Look up an endpoint | [tr/API_REFERANS.md](tr/API_REFERANS.md) — REST API reference |
| Look up an SDK type or method | [tr/SDK_REFERANS.md](tr/SDK_REFERANS.md) — .NET SDK reference |
| Install and run it | [tr/KURULUM_REHBERI.md](tr/KURULUM_REHBERI.md) — deployment guide |

## Installation & operations

| Document | What it covers |
|---|---|
| [tr/KURULUM_REHBERI.md](tr/KURULUM_REHBERI.md) | Deployment guide — binaries, systemd, Docker, TLS, sizing |
| [tr/AGENT_KURULUM.md](tr/AGENT_KURULUM.md) | Desktop Agent — smart-card / USB-token signing on the user's machine |
| [tr/ARCHITECTURE.md](tr/ARCHITECTURE.md) | System architecture, component diagrams, performance figures |
| [tr/CHANGELOG.md](tr/CHANGELOG.md) · [en/CHANGELOG.md](en/CHANGELOG.md) | Release history and migration notes |

## Developer reference

| Document | What it covers |
|---|---|
| [tr/API_REFERANS.md](tr/API_REFERANS.md) | REST API — every endpoint, request/response schema, error codes |
| [tr/SDK_REFERANS.md](tr/SDK_REFERANS.md) | .NET SDK — full type and method reference |
| [tr/ORNEKLER.md](tr/ORNEKLER.md) | 17 end-to-end usage scenarios |
| [tr/KULLANIM_KILAVUZU.md](tr/KULLANIM_KILAVUZU.md) | Tutorial walkthrough from first signature onward |
| [tr/TABLET_RELAY_SIGN.md](tr/TABLET_RELAY_SIGN.md) | Tablet relay-sign API integration |

## Signing concepts

| Document | What it covers |
|---|---|
| [tr/IMZA_FORMATLARI.md](tr/IMZA_FORMATLARI.md) | CAdES / XAdES / PAdES / ASiC formats and levels (B-B … B-LTA) |
| [tr/SAGLAYICILAR.md](tr/SAGLAYICILAR.md) | Signing providers — PKCS#11, PKCS#12, HSM, mobile, KEP |
| [tr/MOBIL_IMZA.md](tr/MOBIL_IMZA.md) | Mobile signature (operator-based) integration |
| [tr/KEP_ENTEGRASYONU.md](tr/KEP_ENTEGRASYONU.md) | KEP (registered e-mail) integration |
| [tr/KAMUSM_TSA_ENTEGRASYONU.md](tr/KAMUSM_TSA_ENTEGRASYONU.md) | Kamu SM timestamp (TSA) integration |
| [tr/DOGRULAMA_ARACLARI.md](tr/DOGRULAMA_ARACLARI.md) | Third-party online signature validators |
| [tr/TURK_MEVZUATI.md](tr/TURK_MEVZUATI.md) | Turkish e-signature legislation compliance analysis |

---

## Downloads

Documentation also ships as a release asset — see the [DigiMR product page](../README.md)
for the current version and download links.

---

info@moreum.com · [www.moreum.com](https://www.moreum.com)
