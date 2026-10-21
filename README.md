<p align="center">
  <img src="docs/cover.png" alt="php2shell" width="700">
</p>

<h1 align="center">php2shell</h1>
<p align="center">
  <strong>CVE-2026-109618 & CVE-2026-109619 — Independent Research & Verification</strong>
</p>
<p align="center">
  <a href="https://github.com/ikow/php2shell-poc/stargazers"><img src="https://img.shields.io/github/stars/ikow/php2shell-poc?style=flat-square" alt="Stars"></a>
  <img src="https://img.shields.io/badge/CVSS-9.2%20Critical-red?style=flat-square" alt="CVSS 9.2">
  <img src="https://img.shields.io/badge/PHP-5.4.0%E2%80%938.5.11-orange?style=flat-square" alt="Affected Versions">
  <img src="https://img.shields.io/badge/Fixed-8.2.35%20%7C%208.3.36%20%7C%208.4.27%20%7C%208.5.12-green?style=flat-square" alt="Fixed Versions">
</p>

---

Two critical pre-authentication RCE vulnerabilities in PHP core, exploiting `session.upload_progress` (enabled by default) and the memory allocator's OOM bailout. **14 years of vulnerable code** (PHP 5.4.0, 2012 — present).

**Author:** [Qi Deng](https://github.com/ikow) &nbsp;|&nbsp; **Writeup:** [qideng.substack.com](https://qideng.substack.com/p/php2shell-dissecting-phps-critical) &nbsp;|&nbsp; **Date:** 2026-10-21

## Impact

| What | Scale |
|------|-------|
| Websites running PHP | **78%** of all websites |
| WordPress alone | **43%** of all websites |
| Affected versions | PHP 5.4.0 through 8.5.11 (14 years) |
| Authentication required | **None** |
| Application vulnerability required | **None** (Chain D — mod_php) |

## Verified Attack Chains

### Chain A — Session File Injection + LFI → RCE

Upload a file with PHP code in the filename → `session.upload_progress` stores it raw in the session file → include via any LFI → code execution.

```
Conditions: upload_progress=On (default) + application has LFI
Reliability: 100%
```

### Chain B — var_destroy UAF → Handler Overwrite → RCE

`var_destroy()` frees back-reference chunks then fires `__destruct` without `serialize_lock`. Inner `unserialize()` reuses the freed var_hash → heap-use-after-free → overwrite function handler → RCE.

```
Conditions: application calls unserialize() with controlled data
ASAN: heap-use-after-free CONFIRMED at var_unserializer.c:851
```

### Chain C — Session Handler Mismatch → `__destruct` → RCE

Session written with `php_serialize` handler (pipe `|` preserved in filename) → read by `php` handler → splits at `|` → deserializes injected object → `__destruct` fires during var_hash cleanup **even when decode returns FAILURE**.

```
Conditions: two directories with different session.serialize_handler
Key finding: __destruct fires on BOTH success and failure paths
```

### Chain D — OOM Corruption → Stale State → RCE ⚡

Multipart POST triggers OOM during request startup → `longjmp()` skips `init_executor` → Apache mod_php runs shutdown on stale `EG()` globals → use-after-free → RCE.

```
Conditions: NONE — any PHP endpoint on Apache mod_php
SAPI: Apache mod_php ✅ | FPM ⚠️ (child exits) | FrankenPHP ✅
```

**This is a pre-auth, zero-dependency RCE against default Apache mod_php installations.**

## Official CVE Details

| CVE | GHSA | Description |
|-----|------|-------------|
| CVE-2026-109618 | [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) | RCE via OOM corruption — `longjmp()` leaves request state invalid |
| CVE-2026-109619 | [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) | RCE via stale global state — worker retains freed request memory |

**Patch:** [Commit 3584a98](https://github.com/php/php-src/commit/3584a98bda) — 300 files changed, replaces `longjmp()` with delayed soft OOM + hard `_exit()`.

## Quick Start

```bash
# Clone
git clone https://github.com/ikow/php2shell-poc.git && cd php2shell-poc

# Start lab (PHP-FPM + nginx + Apache mod_php)
cd lab && docker compose up -d

# Verify Chain A: Session injection
python3 ../poc/verify_session_injection.py --target http://127.0.0.1:18080

# Verify Chain B: UAF (requires ASAN build)
USE_ZEND_ALLOC=0 ASAN_OPTIONS="detect_leaks=0" php ../poc/verify_uaf_asan.php

# Verify Chain C: Handler mismatch
python3 ../poc/verify_handler_mismatch.py --target http://127.0.0.1:18080

# Verify Chain D: OOM on Apache mod_php
python3 ../poc/verify_oom_trigger.py --host 127.0.0.1 --port 18081 --strategy headers
```

## Repository Structure

```
php2shell-poc/
├── README.md
├── poc/
│   ├── verify_session_injection.py    # Chain A: session file injection
│   ├── verify_uaf_asan.php            # Chain B: var_destroy UAF (ASAN)
│   ├── verify_handler_mismatch.py     # Chain C: handler mismatch
│   └── verify_oom_trigger.py          # Chain D: OOM trigger strategies
├── lab/
│   ├── docker-compose.yml             # FPM + nginx + Apache mod_php
│   ├── setup.sh                       # One-click lab setup
│   ├── php.ini                        # FPM configuration
│   └── php-apache.ini                 # Apache mod_php (low memory for OOM)
└── docs/
    ├── ANALYSIS.md                    # Full technical analysis
    └── cover.png                      # Cover image
```

## Mitigation

```ini
; Immediate — reduces attack surface but does NOT fix all chains
session.upload_progress.enabled = Off
session.use_only_cookies = On
```

| Mitigation | Chains blocked |
|-----------|---------------|
| `session.upload_progress.enabled = Off` | A, C (partial) |
| `session.use_only_cookies = On` | A, C (session fixation) |
| Update to PHP 8.2.35+ | **All chains** |

> **The only complete fix is patching PHP.** The OOM corruption (Chain D) is in the Zend engine core, not in session handling.

## Timeline

| Date | Event |
|------|-------|
| 2012-03-01 | `session.upload_progress` introduced in PHP 5.4.0 |
| 2026-10-20 | Details leaked before embargo; PHP advances release |
| 2026-10-21 | PHP 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 released |
| 2026-10-21 | This research published |

## References

- [PHP Security Advisory](https://www.php.net/archive/2026.php)
- [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) — CVE-2026-109618
- [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) — CVE-2026-109619
- [Patch commit](https://github.com/php/php-src/commit/3584a98bda) — 300 files, Zend engine + extensions
- [PR #22113](https://github.com/php/php-src/pull/22113) — Session fixation fix
- [BleepingComputer coverage](https://www.bleepingcomputer.com/news/security/php-ships-emergency-patches-for-critical-php2shell-flaws/)

## Disclaimer

This repository contains **verification POCs only** — tools to confirm whether a system is vulnerable. No weaponized exploits are included. Full exploit code is available to [paid subscribers](https://qideng.substack.com/p/php2shell-dissecting-phps-critical).

For authorized security testing and educational purposes only.

## License

MIT
