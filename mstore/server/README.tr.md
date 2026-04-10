[English](README.md)

# MStore Sunucu

MStore sunucu ve CLI binary. tar.gz arsivi hem `mstore-server` hem `mstore` (CLI) icerir.

**Indir:** [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz) (v0.2.0)

---

## Kurulum

```bash
tar xzf mstore-linux-amd64.tar.gz
sudo mv mstore-server mstore /usr/local/bin/
mstore-server --config config.toml
```

## Yapilandirma

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

### Ortam Degiskenleri

| Degisken | Aciklama |
|----------|----------|
| `MSTORE_MASTER_KEY` | 32-byte base64 sifreleme anahtari |
| `MSTORE_CONFIG` | Yapilandirma dosyasi yolu (varsayilan: `config.toml`) |
| `MSTORE_GRPC_BIND` | gRPC baglanti adresi (varsayilan: `0.0.0.0:9011`) |
| `RUST_LOG` | Log seviyesi (`info`, `debug`, `mstore_object=trace`) |

## CLI Kullanimi

```bash
mstore mb mstore://my-bucket              # Bucket olustur
mstore cp file.txt mstore://my-bucket/    # Yukle
mstore ls mstore://my-bucket/             # Listele
mstore cp mstore://my-bucket/f.txt ./     # Indir
mstore rm mstore://my-bucket/file.txt     # Sil
mstore head mstore://my-bucket/file.txt   # Nesne bilgisi
mstore search mstore://my-bucket "sorgu"  # Tam metin arama
```

### CLI Ortam Degiskenleri

```bash
export MSTORE_ENDPOINT=http://127.0.0.1:9011
export MSTORE_ACCESS_KEY=myaccesskey
export MSTORE_SECRET_KEY=mysecretkey
```

## Portlar

| Port | Protokol | Amac |
|------|----------|------|
| 9010 | HTTP | S3 uyumlu API |
| 9011 | gRPC | SDK, CLI, replikasyon |

## Guncelleme

1. Sunucuyu durdurun
2. Binary dosyalari indirip degistirin
3. Sunucuyu yeniden baslatin
