[English](README.md)

# MStore CLI (mstore-ctl)

Bucket ve nesne yönetimi için komut satırı aracı.

---

## Sürümler

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Platformlar

| Platform | Mimari | Dosya Adı |
|----------|--------|-----------|
| Linux | x86_64 | `mstore-ctl-linux-amd64.tar.gz` |
| Linux | aarch64 | `mstore-ctl-linux-arm64.tar.gz` |
| Windows | x86_64 | `mstore-ctl-windows-amd64.zip` |
| Windows | aarch64 | `mstore-ctl-windows-arm64.zip` |
| macOS | x86_64 | `mstore-ctl-darwin-amd64.tar.gz` |
| macOS | aarch64 | `mstore-ctl-darwin-arm64.tar.gz` |

## Kurulum

```bash
# Linux / macOS
tar xzf mstore-ctl-linux-amd64.tar.gz
sudo mv mstore-ctl /usr/local/bin/
```

```powershell
# Windows
Expand-Archive mstore-ctl-windows-amd64.zip -DestinationPath .
```

## Kullanım

```bash
mstore-ctl mb mstore://my-bucket              # Bucket oluştur
mstore-ctl cp file.txt mstore://my-bucket/     # Yükle
mstore-ctl ls mstore://my-bucket/              # Listele
mstore-ctl cp mstore://my-bucket/f.txt ./f.txt # İndir
mstore-ctl rm mstore://my-bucket/file.txt      # Sil
mstore-ctl head mstore://my-bucket/file.txt    # Nesne bilgisi
mstore-ctl search mstore://my-bucket "sorgu"   # Ara
```

## Yapılandırma

```bash
export MSTORE_ENDPOINT=http://127.0.0.1:9011
export MSTORE_ACCESS_KEY=myaccesskey
export MSTORE_SECRET_KEY=mysecretkey
```
