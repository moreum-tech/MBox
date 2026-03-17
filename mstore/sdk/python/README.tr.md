[English](README.md)

# MStore Python SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum

```bash
pip install mstore==0.1.0
```

## Kullanım

```python
from mstore import MStoreClient

client = MStoreClient("http://127.0.0.1:9011")
client.create_bucket("my-bucket")
client.put_object("my-bucket", "hello.txt", b"Hello")
```

Gereksinimler: Python 3.9+
