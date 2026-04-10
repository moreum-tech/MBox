[Turkce](README.tr.md)

# DigiMR Token Agent

Local agent for hardware token and smartcard access. Runs on port 5555 and bridges the DigiMR API server to PKCS#11 devices on the user's machine.

In hybrid mode, the signing PIN never leaves the user's computer.

The Token Agent binary is included in the [API server package](https://github.com/moreum-tech/MBox/releases/download/digimr-v1.1.0/digimr-linux-amd64.tar.gz).

---

## Installation

```bash
tar xzf digimr-linux-amd64.tar.gz
chmod +x DigiMR.Agent
./DigiMR.Agent
# Runs on http://localhost:5555
```

## Configuration

```json
{
  "Kestrel": {
    "Endpoints": {
      "Local": { "Url": "http://127.0.0.1:5555" }
    }
  }
}
```

## Usage

The Agent is used automatically when signing with `RemoteToken` provider in the DigiMR API. The user's browser connects to `http://localhost:5555` via SignalR, while the API server never sees the PIN.

## System Requirements

- CPU: 1 vCPU
- RAM: 512 MB
- OS: Windows 10+, Ubuntu 20.04+
- PKCS#11 driver for your token brand installed (AKiS, SafeNet, Gemalto, etc.)
