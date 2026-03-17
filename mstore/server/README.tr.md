[English](README.md)

# MStore Sunucu

MStore sunucu dosyası. S3 HTTP (port 9010) ve gRPC (port 9011) uç noktalarını çalıştırır.

---

## Sürümler

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Platformlar

| Platform | Mimari | Dosya Adı |
|----------|--------|-----------|
| Linux | x86_64 | `mstore-server-linux-amd64.tar.gz` |
| Linux | aarch64 | `mstore-server-linux-arm64.tar.gz` |
| Windows | x86_64 | `mstore-server-windows-amd64.zip` |
| Windows | aarch64 | `mstore-server-windows-arm64.zip` |
| macOS | x86_64 | `mstore-server-darwin-amd64.tar.gz` |
| macOS | aarch64 | `mstore-server-darwin-arm64.tar.gz` |

## Kurulum

### Linux / macOS

```bash
tar xzf mstore-server-linux-amd64.tar.gz
chmod +x mstore-server
sudo mv mstore-server /usr/local/bin/
mstore-server --config config.toml
```

### Windows

```powershell
Expand-Archive mstore-server-windows-amd64.zip -DestinationPath .
.\mstore-server.exe --config config.toml
```

## Yapılandırma

```toml
[node]
name    = "mstore-01"
address = "127.0.0.1:9010"

[storage]
drives = [{ path = "/data/drive1" }]

[erasure]
data_shards   = 1
parity_shards = 0

[api]
bind = "0.0.0.0:9010"

[auth]
root_access_key = "myaccesskey"
root_secret_key = "mysecretkey"
```

### Ortam Değişkenleri

| Değişken | Açıklama |
|----------|----------|
| `MSTORE_MASTER_KEY` | 32 bayt base64 şifreleme anahtarı |
| `MSTORE_CONFIG` | Yapılandırma dosyası yolu (varsayılan: `config.toml`) |
| `MSTORE_GRPC_BIND` | gRPC bağlantı adresi (varsayılan: `0.0.0.0:9011`) |
| `RUST_LOG` | Log seviyesi (`info`, `debug`, `mstore_object=trace`) |

## Güncelleme

1. Sunucuyu durdurun
2. Yeni binary dosyasını indirip değiştirin
3. Sunucuyu yeniden başlatın

Uyumsuz değişiklikler sürüm notlarında belirtilir.
