[Turkce](README.tr.md)

# MStore Python SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install

```bash
pip install mstore==0.1.0
```

## Usage

```python
from mstore import MStoreClient

client = MStoreClient("http://127.0.0.1:9011")
client.create_bucket("my-bucket")
client.put_object("my-bucket", "hello.txt", b"Hello")
```

Requirements: Python 3.9+
