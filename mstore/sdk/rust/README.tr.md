[English](README.md)

# MStore Rust SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum

```toml
[dependencies]
mstore-client = "0.1.0"
```

## Kullanım

```rust
use mstore_client::MStoreClient;

let mut client = MStoreClient::new("http://127.0.0.1:9011").await?;
client.create_bucket("my-bucket").await?;
client.put_object("my-bucket", "hello.txt", b"Hello".to_vec()).await?;
```
