# DSPD — Device-Bound Stateless Password Derivation

[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Paper](https://img.shields.io/badge/paper-arXiv-red)](https://arxiv.org)

> **One master key. No database. No cloud. Every password is unique, strong, and bound to your device.**

DSPD is a novel password derivation scheme that generates strong, unique passwords for every application from a single master key — with **zero stored database**. Unlike existing stateless password managers (LessPass, Spectre), DSPD binds derivation to a **hardware-resident device secret**, meaning your master key alone is never enough for an attacker.

---

## How It Works

```
password = Argon2id(
    master_key,
    HKDF(app_name + version)  XOR  device_secret,
    t=3, m=64MB, p=2
)
```

| Input | What it is | Who provides it |
|---|---|---|
| `master_key` | Single passphrase you remember | You |
| `app_name` | Site/app name ("github.com") | You / auto-detected |
| `device_secret` | 32-byte random, stored in OS keystore | Generated once, lives in hardware |
| `version` | Rotation counter (default 1) | Auto-managed |

**The device secret never leaves your device.** Even if an attacker steals your master key, they cannot derive any password without physical access to your device.

---

## Installation

```bash
pip install dspd
```

Or install from source:

```bash
git clone https://github.com/YOUR_USERNAME/dspd.git
cd dspd
pip install .
```

---

## Quick Start

### Command Line

```bash
# Initialize on first use (generates device secret)
dspd init

# Get password for any app
dspd get github.com

# Rotate a single app's password (after a breach)
dspd rotate github.com

# List all apps with custom version counters
dspd list

# Run performance benchmark on your device
dspd benchmark

# Backup your device secret (store safely!)
dspd backup
```

### Python API

```python
from dspd import DSPDManager

mgr = DSPDManager()

# Derive password
password = mgr.get_password("github.com", master_key="my-passphrase")
print(password)   # e.g. "aK9#mZxQ2!vBnLpW"

# Rotate one app's password (breach response)
result = mgr.rotate_password("github.com", master_key="my-passphrase")
print(result["new_password"])   # different password, all others unchanged

# Low-level API
from dspd import derive_password, get_device_secret

device_secret = get_device_secret()
pw = derive_password(
    master_key    = "my-passphrase",
    app_name      = "github.com",
    device_secret = device_secret,
    version       = 1,
)
```

---

## Security Properties

| Property | DSPD | LessPass | Spectre |
|---|---|---|---|
| No password database | ✅ | ✅ | ✅ |
| Memory-hard KDF (Argon2id) | ✅ | ❌ (PBKDF2) | ✅ (scrypt) |
| Device binding | ✅ **Novel** | ❌ | ❌ |
| Per-app password rotation | ✅ **Novel** | ❌ | ❌ |
| Master key alone = insufficient | ✅ | ❌ | ❌ |
| Server-free operation | ✅ | ✅ | ✅ |

### Argon2id Parameters

| Parameter | Value | Purpose |
|---|---|---|
| Time cost (t) | 3 | Iterations |
| Memory cost (m) | 65,536 KB (64 MB) | Memory hardness — defeats GPU attacks |
| Parallelism (p) | 2 | Thread count |
| Output length | 32 bytes | Hash output |

**Benchmark (Intel Core i5-7200U @ 2.50GHz):** 108.9 ms average — well within acceptable latency.

---

## Brute-Force Resistance

Assuming a 10× RTX 4090 GPU cluster:

| Scheme | 4-word passphrase (~10¹²) | 6-word passphrase (~10¹⁸) |
|---|---|---|
| LessPass (PBKDF2) | 13.2 hours | 1,510 years |
| Spectre (scrypt) | 77.2 days | 211,399 years |
| **DSPD (Argon2id)** | ~3 years | **2,882,708 years** |
| **DSPD + device secret** | **Impossible** | **Impossible** |

*DSPD's key advantage: without the device secret, offline GPU attacks cannot even compute the correct Argon2id salt — making the computation irrelevant.*

---

## Device Secret Storage

DSPD stores the device secret using the OS-native secure keystore:

| Platform | Storage Backend |
|---|---|
| Windows | Windows Credential Manager |
| macOS | macOS Keychain |
| Linux | SecretService (GNOME Keyring / KWallet) |
| Fallback | `~/.dspd/device_secret.bin` (chmod 600) |

---

## Password Recovery / Backup

If you get a new device, use the backup/restore flow:

```bash
# On old device — export backup string
dspd backup
# → Copy the base64 string somewhere secure (printed paper, encrypted USB)

# On new device — restore
dspd restore <your-base64-backup-string>
```

> ⚠️ **WARNING:** Without the device secret backup, passwords cannot be recovered on a new device. Always back up after `dspd init`.

---

## Project Structure

```
dspd/
├── dspd/
│   ├── __init__.py       # Public API exports
│   ├── core.py           # DSPD algorithm (HKDF + Argon2id)
│   ├── device.py         # Device secret management
│   ├── version_store.py  # Per-app version counters
│   ├── manager.py        # High-level DSPDManager class
│   └── cli.py            # Command-line interface
├── tests/
│   └── test_dspd.py      # Unit test suite
├── pyproject.toml
├── requirements.txt
├── LICENSE
└── README.md
```

---

## Running Tests

```bash
pip install pytest
pytest tests/ -v
```

---

## Research Paper

This implementation accompanies the paper:

> **Talha R.** (2025). *DSPD: A Device-Bound Stateless Password Derivation Scheme with Version-Controlled Rotation for Multi-Platform Environments.* Bahria University, Islamabad.

---

## License

MIT License — see [LICENSE](LICENSE).

---

## Contributing

Pull requests welcome. For major changes, open an issue first.

1. Fork the repo
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes
4. Push and open a PR
