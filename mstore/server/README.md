[Turkce](README.tr.md)

# MStore Server

MStore server binary. Runs S3 HTTP (port 9010) and gRPC (port 9011) endpoints.

---

## Versions

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Platforms

| Platform | Architecture | Filename |
|----------|-------------|----------|
| Linux | x86_64 | `mstore-server-linux-amd64.tar.gz` |
| Linux | aarch64 | `mstore-server-linux-arm64.tar.gz` |
| Windows | x86_64 | `mstore-server-windows-amd64.zip` |
| Windows | aarch64 | `mstore-server-windows-arm64.zip` |
| macOS | x86_64 | `mstore-server-darwin-amd64.tar.gz` |
| macOS | aarch64 | `mstore-server-darwin-arm64.tar.gz` |

## Installation

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

## Update

1. Stop the server
2. Download and replace the binary
3. Restart the server

Breaking changes are noted in version release notes.
