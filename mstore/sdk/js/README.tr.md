[English](README.md)

# MStore JavaScript/TypeScript SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum

```bash
npm install @moreum/mstore@0.1.0
```

## Kullanım

```typescript
import { MStoreClient } from "@moreum/mstore";

const client = new MStoreClient("http://127.0.0.1:9011");
await client.createBucket("my-bucket");
await client.putObject("my-bucket", "hello.txt", Buffer.from("Hello"));
```

Gereksinimler: Node.js 18+
