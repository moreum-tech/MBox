[Turkce](README.tr.md)

# MStore C# SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install

```bash
dotnet add package MStore.Client --version 0.1.0
```

## Usage

```csharp
using MStore.Client;

var client = new MStoreClient("http://127.0.0.1:9011");
await client.CreateBucketAsync("my-bucket");
await client.PutObjectAsync("my-bucket", "hello.txt", bytes);
```

Requirements: .NET 8.0+
