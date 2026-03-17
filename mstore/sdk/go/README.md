[Turkce](README.tr.md)

# MStore Go SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install

```bash
go get github.com/moreum-tech/mstore-go@v0.1.0
```

## Usage

```go
client, _ := mstore.NewClient("http://127.0.0.1:9011")
client.CreateBucket(ctx, "my-bucket")
client.PutObject(ctx, "my-bucket", "hello.txt", data)
```

Requirements: Go 1.21+
