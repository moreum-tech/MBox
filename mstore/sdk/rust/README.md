[Turkce](README.tr.md)

# MStore Rust SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install

```toml
[dependencies]
mstore-client = "0.1.0"
```

## Usage

```rust
use mstore_client::MStoreClient;

let mut client = MStoreClient::new("http://127.0.0.1:9011").await?;
client.create_bucket("my-bucket").await?;
client.put_object("my-bucket", "hello.txt", b"Hello".to_vec()).await?;
```
