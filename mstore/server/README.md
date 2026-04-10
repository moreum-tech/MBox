[Turkce](README.tr.md)

# MStore Server

MStore server and CLI binary. The tar.gz archive includes both `mstore-server` and `mstore` (CLI).

**Download:** [mstore-linux-amd64.tar.gz](https://github.com/moreum-tech/MBox/releases/download/mstore-v0.2.0/mstore-linux-amd64.tar.gz) (v0.2.0)

---

## Installation

```bash
tar xzf mstore-linux-amd64.tar.gz
sudo mv mstore-server mstore /usr/local/bin/
mstore-server --config config.toml
```

## Configuration

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

### Environment Variables

| Variable | Description |
|----------|-------------|
| `MSTORE_MASTER_KEY` | 32-byte base64 encryption key |
| `MSTORE_CONFIG` | Config file path (default: `config.toml`) |
| `MSTORE_GRPC_BIND` | gRPC bind address (default: `0.0.0.0:9011`) |
| `RUST_LOG` | Log level (`info`, `debug`, `mstore_object=trace`) |

## CLI Usage

```bash
mstore mb mstore://my-bucket              # Create bucket
mstore cp file.txt mstore://my-bucket/     # Upload
mstore ls mstore://my-bucket/              # List
mstore cp mstore://my-bucket/f.txt ./      # Download
mstore rm mstore://my-bucket/file.txt      # Delete
mstore head mstore://my-bucket/file.txt    # Object info
mstore search mstore://my-bucket "query"   # Full-text search
```

### CLI Environment

```bash
export MSTORE_ENDPOINT=http://127.0.0.1:9011
export MSTORE_ACCESS_KEY=myaccesskey
export MSTORE_SECRET_KEY=mysecretkey
```

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9010 | HTTP | S3-compatible API |
| 9011 | gRPC | SDK, CLI, replication |

## Update

1. Stop the server
2. Download and replace the binaries
3. Restart the server
