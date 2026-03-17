[English](README.md)

# MStore C# SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum

```bash
dotnet add package MStore.Client --version 0.1.0
```

## Kullanım

```csharp
using MStore.Client;

var client = new MStoreClient("http://127.0.0.1:9011");
await client.CreateBucketAsync("my-bucket");
await client.PutObjectAsync("my-bucket", "hello.txt", bytes);
```

Gereksinimler: .NET 8.0+
