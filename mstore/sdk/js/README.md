[Turkce](README.tr.md)

# MStore JavaScript/TypeScript SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install

```bash
npm install @moreum/mstore@0.1.0
```

## Usage

```typescript
import { MStoreClient } from "@moreum/mstore";

const client = new MStoreClient("http://127.0.0.1:9011");
await client.createBucket("my-bucket");
await client.putObject("my-bucket", "hello.txt", Buffer.from("Hello"));
```

Requirements: Node.js 18+
