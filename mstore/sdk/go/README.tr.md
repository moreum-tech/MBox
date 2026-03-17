[English](README.md)

# MStore Go SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum

```bash
go get github.com/moreum-tech/mstore-go@v0.1.0
```

## Kullanım

```go
client, _ := mstore.NewClient("http://127.0.0.1:9011")
client.CreateBucket(ctx, "my-bucket")
client.PutObject(ctx, "my-bucket", "hello.txt", data)
```

Gereksinimler: Go 1.21+
