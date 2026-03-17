# CloakBrowser Security Audit — Is the Closed-Source Binary Safe?

**TL;DR:** We ran 8 automated security tests inside Docker against CloakBrowser's proprietary Chromium binary. **No malicious behavior detected.** But you're still running a closed-source executable you can't fully verify — read on for what we found and how to verify it yourself.

---

## Why This Audit Exists

[CloakBrowser](https://github.com/CloakHQ/CloakBrowser) is a stealth browser automation tool that ships a **closed-source, patched Chromium binary** with 33 C++ fingerprint spoofing patches. The Python wrapper is MIT-licensed and fully readable — but the binary itself is proprietary.

**The question:** When you run `pip install cloakbrowser` and it downloads a 441 MB `chrome` binary from `cloakbrowser.dev`, how do you know it's not doing something malicious?

**You don't — not with certainty.** But you *can* observe its behavior. That's what this audit does.

---

## How CloakBrowser Works Internally

```
User Code (Python)
    │
    ▼
┌──────────────────────────────────────────────────┐
│  cloakbrowser.launch()                           │
│                                                  │
│  1. ensure_binary()   → download patched chrome  │
│  2. resolve_geoip()   → timezone/locale from IP  │
│  3. build_stealth_args() → fingerprint CLI flags │
│  4. playwright.launch()  → start browser         │
│  5. patch_humanize()     → human-like behavior   │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│  Playwright / Patchright (CDP backend)           │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│  Patched Chromium Binary (CLOSED SOURCE)         │
│  441 MB ELF x86-64 (Linux) / PE (Windows)       │
│  Based on ungoogled-chromium 145.0.7632.x        │
│  33 C++ source patches compiled in:              │
│  • Canvas fingerprint randomization              │
│  • WebGL/WebGPU vendor/renderer spoofing         │
│  • Audio context fingerprint noise               │
│  • Screen/hardware/memory spoofing               │
│  • CDP automation signal suppression             │
│  • Font enumeration masking                      │
└──────────────────────────────────────────────────┘
```

### The Trust Model

| Component | Source Available | Can You Audit It? |
|-----------|:-:|---|
| Python wrapper (`cloakbrowser/`) | Yes (MIT) | Fully readable, well-written |
| JavaScript wrapper (`js/`) | Yes (MIT) | Fully readable |
| Chromium binary (`chrome`) | **No** (Proprietary) | **Cannot inspect the 33 C++ patches** |
| SHA-256 checksums | Fetched from same server | Self-referential — if server is compromised, both change |

### Binary Download Chain

```
pip install cloakbrowser
    → first launch triggers ensure_binary()
    → downloads from cloakbrowser.dev (primary) or GitHub Releases (fallback)
    → fetches SHA256SUMS from same server
    → verifies hash matches
    → extracts to ~/.cloakbrowser/chromium-{version}/
    → runs with --no-sandbox (disabled Chromium sandbox)
```

### Auto-Update Behavior

A **background daemon thread** checks for updates every hour:
- Queries GitHub Releases API for newer `chromium-v*` releases
- Queries PyPI for newer wrapper versions
- Downloads silently, activates on next launch

Disable with: `CLOAKBROWSER_AUTO_UPDATE=false`

---

## Audit Methodology

We built a Docker container that:
1. Installs CloakBrowser and downloads the binary
2. Runs the binary visiting only `about:blank` (no real websites)
3. Monitors everything: network traffic, file access, DNS queries, process tree, environment variables
4. Sets decoy secrets (fake AWS keys, GitHub tokens) and checks if they're exfiltrated
5. Blocks all network access and checks if the browser still works (malware often crashes without its C2 server)

### Test Environment

| Property | Value |
|----------|-------|
| Date | March 17, 2026 |
| Binary | `chromium-145.0.7632.159.7` (Linux x64) |
| SHA-256 | `f1783cf24eb9abdf262a607c2a41be23e28343000849dde81a452adb0ff9d8fa` |
| Size | 441 MB |
| Base image | `python:3.12-slim` (Debian Trixie) |
| Tools | tcpdump, strace, readelf, ldd, iptables, strings |

---

## Test Results

### Test 1: Binary Strings Analysis

Extracted **2,916,580** strings from the binary and searched for suspicious content.

| Check | Result | Details |
|-------|--------|---------|
| Suspicious URLs/domains | **Clean** | All URLs are standard Chromium internals: OCSP certificate endpoints, crbug.com references, XML namespaces, `foo.com` test data |
| Data exfiltration keywords | **Clean** | Matches like "capture", "record", "tracking" are all standard Chromium DevTools/WebRTC/media APIs |
| Cryptocurrency/mining strings | **Clean** | "Wallet" matches are all standard Chromium autofill/payments (`autofill_wallet_sync_bridge.cc`) and Linux KWallet keychain integration |
| Hardcoded IPs | **Clean** | Found standard DNS resolvers (1.0.0.1, 149.112.112.112 = Quad9, 185.222.222.222 = CleanBrowsing) — all well-known public DNS |
| Base64 payloads | **Clean** | Long base64 strings decode to garbage — they're Chromium accessibility event name concatenations, not encoded payloads |

### Test 2: Network Connection Monitoring

Launched browser visiting `about:blank` for 30 seconds with full packet capture.

| Check | Result | Details |
|-------|--------|---------|
| Total packets | ~120 non-localhost | All accounted for |
| Destination IPs | `151.101.0.223` (pypi.org), `140.82.121.5` (github.com) | **Python wrapper update check**, not the binary |
| Unexpected connections | **None** | Zero connections to unknown servers |
| Binary phoning home | **No** | All network activity traced to the wrapper's `ensure_binary()` update check |

**The 120 packets explained:** The Python wrapper checks PyPI for a newer `cloakbrowser` package version and GitHub for a newer binary on every launch. This is the wrapper code, not the Chromium binary. Disable with `CLOAKBROWSER_AUTO_UPDATE=false`.

### Test 3: File System Access Audit

Ran browser under `strace` monitoring all file open/read/write syscalls.

| Check | Result | Details |
|-------|--------|---------|
| SSH keys (`.ssh/`) | **Not accessed** | |
| AWS credentials (`.aws/`) | **Not accessed** | |
| Environment files (`.env`) | **Not accessed** | |
| Crypto wallets | **Not accessed** | |
| Cloud credentials (`.gcloud`, `.azure`, `.kube`) | **Not accessed** | |
| Browser passwords | **Not accessed** | |
| Docker/npm tokens | **Not accessed** | |
| Files opened | Standard only | `/etc/localtime`, `/etc/os-release`, Vulkan/GPU config, Chromium crash reports dir, own browser profile in `/tmp/` |

### Test 4: Process Tree Audit

| Process | Expected? | Purpose |
|---------|:-:|---------|
| `chrome` (main) | Yes | Main browser process with stealth flags |
| `chrome --type=zygote` (x2) | Yes | Standard Chromium process forking |
| `chrome --type=gpu-process` | Yes | GPU/SwiftShader rendering |
| `chrome --type=utility` (network) | Yes | Network service |
| `chrome --type=utility` (storage) | Yes | Storage service |
| `chrome --type=renderer` (x2) | Yes | Page rendering |
| `chrome_crashpad_handler` (x2) | Yes | Crash reporting (local only) |
| **Unknown processes** | **None** | |

All processes are standard Chromium architecture. No mystery child processes, no crypto miners, no reverse shells.

### Test 5: DNS Query Audit

Captured all DNS queries during browser operation on `about:blank`.

| Domain Queried | Expected? | Purpose |
|----------------|:-:|---------|
| `pypi.org` | Yes | Wrapper update check |
| **Other domains** | **None** | |

Zero DNS queries from the binary itself.

### Test 6: Binary Section Analysis

| Check | Result | Details |
|-------|--------|---------|
| File type | `ELF 64-bit LSB pie executable, x86-64` | Standard Linux executable |
| Build ID | `19c3479a7be0866e7c5b6398310340f974489cfe` | Unique build identifier present |
| Shared libraries | **All standard** | glib, nss, X11, cairo, pango, cups, dbus, alsa — standard Chromium dependencies |
| Suspicious libraries | **None** | No unknown `.so` files linked |
| Crashpad notes | Present | Standard Chromium crash reporting metadata |

### Test 7: Environment Variable Sniffing (Canary Test)

Set decoy secrets in environment, then monitored if the browser read or exfiltrated them.

**Decoy secrets planted:**
```
AWS_SECRET_ACCESS_KEY=DECOY_aws_key_12345_CANARY
DATABASE_URL=postgresql://admin:DECOY_password@db.example.com/prod
GITHUB_TOKEN=ghp_DECOY_token_67890_CANARY
STRIPE_SECRET_KEY=sk_live_DECOY_stripe_CANARY
PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY-----DECOY_CANARY
```

| Check | Result |
|-------|--------|
| `/proc/self/environ` reads | **PASS** — 0 attempts to read environment file |
| Cross-process `/proc/*/environ` | **PASS** — No snooping on other processes |
| Canary strings in network traffic | **PASS** — No decoy secrets found in captured packets |

**The binary did not attempt to read or exfiltrate environment variables.**

### Test 8: Behavior When Network is Blocked

Blocked all outbound traffic with `iptables`, then launched the browser.

| Check | Result |
|-------|--------|
| Browser launch | **PASS** — Launched successfully |
| Browser close | **PASS** — Closed cleanly |
| Crash/hang | **None** — No dependency on external server |

Malware that phones home to a C2 server typically crashes or hangs when network is blocked. The CloakBrowser binary operates normally without network access.

---

## Overall Verdict

### What We Found

```
┌─────────────────────────────────────────────────────────────┐
│  8/8 TESTS PASSED                                           │
│                                                             │
│  ✓ No suspicious strings/URLs in binary                     │
│  ✓ No unexpected network connections (only PyPI/GitHub      │
│    update check from Python wrapper)                        │
│  ✓ No sensitive file access (.ssh, .aws, .env, wallets)     │
│  ✓ No unknown processes spawned                             │
│  ✓ No suspicious DNS queries                                │
│  ✓ Standard ELF binary with standard shared libraries       │
│  ✓ No environment variable sniffing or exfiltration         │
│  ✓ Works fine with network completely blocked                │
└─────────────────────────────────────────────────────────────┘
```

### What This Means

**The binary behaves like a legitimate patched Chromium.** In our testing, it:
- Only accesses files a normal browser would access
- Doesn't read your secrets, credentials, or sensitive files
- Doesn't connect to any unknown servers
- Doesn't spawn suspicious processes
- Works fine offline

### What This Does NOT Mean

This audit **cannot prove** the binary is safe. A sophisticated backdoor could:

| Evasion Technique | Our Test Catches It? |
|-------------------|:---:|
| Always-on data exfiltration | Yes |
| Time bomb (activates after N days) | **No** |
| Conditional trigger (specific sites/conditions) | **No** |
| Piggyback exfiltration (hides data in normal browsing traffic) | **No** |
| Targeted payload (only activates for specific users) | **No** |
| Encrypted exfiltration (blends with HTTPS traffic) | **No** |

**The only guarantee is auditable source code** — which CloakBrowser's binary does not provide.

### Risk Comparison

| Tool | Source Available | Binary Auditable | Risk Level |
|------|:-:|:-:|---|
| **Camoufox** | Fully open source | Yes — compile yourself | Lowest |
| **Patchright** | Fully open source | Yes — compile yourself | Lowest |
| **SeleniumBase** | Fully open source | Uses stock ChromeDriver | Low |
| **CloakBrowser** | Wrapper only | **No** — proprietary binary | **Medium** |
| Random GitHub binary | No | No | High |

---

## Mitigation Recommendations

If you choose to use CloakBrowser, reduce risk with:

```bash
# 1. Disable auto-update (prevents silent binary replacement)
export CLOAKBROWSER_AUTO_UPDATE=false

# 2. Run in Docker with restricted network
docker run --network=none cloakbrowser-app  # No network at all
# Or allow only your target sites:
docker run --network=custom-bridge cloakbrowser-app

# 3. Use your own Chromium build (bypasses their binary entirely)
export CLOAKBROWSER_BINARY_PATH=/path/to/your/chromium

# 4. Pin the binary version (prevent unexpected updates)
# Verify SHA-256: f1783cf24eb9abdf262a607c2a41be23e28343000849dde81a452adb0ff9d8fa
```

---

## Run the Audit Yourself

```bash
# Clone this repo
git clone https://github.com/YOUR_USERNAME/cloakbrowser-security-audit.git
cd cloakbrowser-security-audit

# Clone CloakBrowser into audit directory
git clone https://github.com/CloakHQ/CloakBrowser.git
cd CloakBrowser

# Build the audit container
docker build -f Dockerfile.security-audit -t cloakbrowser-audit .

# Run all 8 tests
docker run --rm --cap-add=NET_ADMIN cloakbrowser-audit

# Interactive inspection (browse artifacts manually)
docker run -it --cap-add=NET_ADMIN cloakbrowser-audit bash
# Then: cat /tmp/binary_strings.txt | less
#        cat /tmp/strace_output.txt | less
#        bash /audit/07_env_sniff_test.sh
```

**Note:** `--cap-add=NET_ADMIN` is needed for Test 8 (iptables network blocking). Without it, Test 8 will skip gracefully.

### What Each Test Does

| Test | Script | Technique | Duration |
|------|--------|-----------|----------|
| 1. Strings Analysis | `01_strings_analysis.sh` | `strings` + grep for URLs, exfil keywords, crypto, IPs, base64 | ~5s |
| 2. Network Monitor | `02_network_monitor.sh` | `tcpdump` during 30s browser session on `about:blank` | ~35s |
| 3. File Access | `03_file_access_audit.sh` | `strace` all `open/openat/read/write/connect` syscalls | ~25s |
| 4. Process Tree | `04_process_monitor.sh` | `ps auxf` during browser session | ~25s |
| 5. DNS Audit | `05_dns_audit.sh` | `tcpdump port 53` during browser session | ~25s |
| 6. Binary Analysis | `06_binary_comparison.sh` | `readelf`, `ldd`, `file` on the binary | ~2s |
| 7. Env Sniffing | `07_env_sniff_test.sh` | Plant decoy secrets, `strace` for `/proc/*/environ`, capture network | ~35s |
| 8. Network Blocked | `08_blocked_network_test.sh` | `iptables -j DROP`, launch browser, check behavior | ~25s |

**Total runtime:** ~3 minutes

---

## Artifacts

After running the audit, these files are available inside the container for manual inspection:

| File | Contents |
|------|----------|
| `/tmp/binary_strings.txt` | All 2.9M strings extracted from the binary |
| `/tmp/network_capture.txt` | Full packet capture during Test 2 |
| `/tmp/strace_output.txt` | All syscalls (file + network) during Test 3 |
| `/tmp/dns_queries.txt` | DNS queries during Test 5 |
| `/tmp/strace_env.txt` | Strace log from canary test (Test 7) |
| `/tmp/env_pcap.pcap` | Network capture during canary test (Test 7) |

---

## Conclusion

CloakBrowser's binary **passed all 8 tests** with no indicators of malicious behavior. The wrapper code is well-written with proper security practices (checksums, path traversal protection, atomic downloads).

However, **passing behavioral tests is not the same as being provably safe.** The binary remains closed-source, the license prohibits reverse engineering, and the checksum verification is self-referential (same server hosts both binary and checksums).

**If you need guaranteed safety:** Use [Camoufox](https://github.com/nicetry-i/camoufox) (open-source Firefox with C++ patches) or [Patchright](https://github.com/nicetry-i/patchright) (open-source Playwright patches).

**If you're comfortable with the risk:** CloakBrowser appears to be doing exactly what it claims — and nothing more — based on our testing.

---

*Audit conducted March 2026. Results are specific to binary version `145.0.7632.159.7` (SHA-256: `f1783cf...`). Future versions may differ.*
