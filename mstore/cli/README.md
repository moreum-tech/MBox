[Turkce](README.tr.md)

# MStore CLI (mstore-ctl)

Command-line tool for bucket and object management.

---

## Versions

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Platforms

| Platform | Architecture | Filename |
|----------|-------------|----------|
| Linux | x86_64 | `mstore-ctl-linux-amd64.tar.gz` |
| Linux | aarch64 | `mstore-ctl-linux-arm64.tar.gz` |
| Windows | x86_64 | `mstore-ctl-windows-amd64.zip` |
| Windows | aarch64 | `mstore-ctl-windows-arm64.zip` |
| macOS | x86_64 | `mstore-ctl-darwin-amd64.tar.gz` |
| macOS | aarch64 | `mstore-ctl-darwin-arm64.tar.gz` |

## Installation

```bash
# Linux / macOS
tar xzf mstore-ctl-linux-amd64.tar.gz
sudo mv mstore-ctl /usr/local/bin/
```

```powershell
# Windows
Expand-Archive mstore-ctl-windows-amd64.zip -DestinationPath .
```

## Usage

```bash
mstore-ctl mb mstore://my-bucket              # Create bucket
mstore-ctl cp file.txt mstore://my-bucket/     # Upload
mstore-ctl ls mstore://my-bucket/              # List
mstore-ctl cp mstore://my-bucket/f.txt ./f.txt # Download
mstore-ctl rm mstore://my-bucket/file.txt      # Delete
mstore-ctl head mstore://my-bucket/file.txt    # Object info
mstore-ctl search mstore://my-bucket "query"   # Search
```

## Configuration

```bash
export MSTORE_ENDPOINT=http://127.0.0.1:9011
export MSTORE_ACCESS_KEY=myaccesskey
export MSTORE_SECRET_KEY=mysecretkey
```
