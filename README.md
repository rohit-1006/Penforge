# PenForge

> Modular Penetration Testing Framework — Python CLI

A lightweight, extensible penetration testing framework built around a plugin and exploit-chain architecture. PenForge scans target ports concurrently, runs registered plugins against each open port, and executes configurable exploit chains — all from a single command.

> **For authorized use only.** Only test systems you own or have explicit written permission to assess. Unauthorized use is illegal.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
- [Architecture](#architecture)
- [Built-in Plugins](#built-in-plugins)
- [Writing Custom Plugins](#writing-custom-plugins)
- [Exploit Chains](#exploit-chains)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [Changelog](#changelog)
- [References](#references)
- [License](#license)

---

## Overview

PenForge is designed around the idea that every penetration test runs the same core loop: scan a port, gather information, probe for weaknesses, and attempt exploitation. Rather than hard-coding that logic, PenForge exposes a clean plugin interface so each step is independently swappable and extendable.

Plugins register against the framework and fire automatically on every open port. Exploit chains are port-specific sequences of functions that run after plugins complete. Both mechanisms run concurrently across all target ports using Python threads.

---

## Features

- Concurrent TCP port scanning across all specified ports
- Plugin system — register any number of plugins, all run on every open port
- Exploit chains — assign ordered sequences of exploit functions to specific ports
- Thread-safe result collection via a shared queue
- Three built-in plugins: banner grabbing, buffer overflow probing, and brute-force credential testing
- Custom shellcode exploit chain included as a reference implementation
- Minimal dependencies — uses the Python standard library only

---

## Installation

### Requirements

- Python 3.8 or higher
- No external dependencies — uses only the Python standard library

### Steps

```bash
# Clone the repository
git clone https://github.com/rohit-1006/Penforge.git
cd Penforge

# Run directly — no install needed
python3 penforge.py --help
```

---

## Usage

### Basic Scan

```bash
python3 penforge.py --target 192.168.1.10 --ports 80,443,22
```

### Scan with Multiple Ports

```bash
python3 penforge.py --target 10.0.0.5 --ports 21,22,23,25,80,443,3306,8080
```

### Full CLI Reference

```
Usage: python3 penforge.py [OPTIONS]

Required:
  --target IP        Target IP address
  --ports PORTS      Comma-separated list of ports to scan (e.g. 80,443,9999)
```

### Sample Output

```
[INFO] Scanning 192.168.1.10:80
[SUCCESS] Port 80 is open
[INFO] Banner from 192.168.1.10:80: HTTP/1.1 200 OK Server: Apache/2.4.41...
[INFO] No buffer overflow detected on 192.168.1.10:80
[INFO] No brute-force success on 192.168.1.10:80

[INFO] Scanning 192.168.1.10:9999
[SUCCESS] Port 9999 is open
[VULN] Potential buffer overflow on 192.168.1.10:9999
[INFO] Custom shellcode sent to 192.168.1.10:9999; check for shell

[INFO] Scanning 192.168.1.10:443
[INFO] Port 443 is closed
```

---

## Architecture

PenForge is built around two core extension points:

### Plugins

Plugins implement `PluginBase` and are registered on the framework instance. Every registered plugin runs against **every open port** discovered during the scan. Plugins run concurrently per port in separate threads.

```
Target Port (open)
       │
       ├── BannerGrabPlugin.run()
       ├── BufferOverflowPlugin.run()
       ├── BruteForcePlugin.run()
       └── YourCustomPlugin.run()     ← add your own
```

### Exploit Chains

Exploit chains are **port-specific** lists of callable functions. When a port in the chain mapping is found open, each function in the chain executes in sequence, in order.

```
Port 9999 (open)
       │
       ├── [Plugin phase — all registered plugins run]
       └── [Chain phase — exploit functions run in order]
              └── custom_shell_exploit()
```

### Execution Flow

```
penforge.execute()
    │
    ├── For each port → thread: _scan_and_exploit(port)
    │       ├── scan_port()           → TCP connect test
    │       ├── run_plugins_on_port() → all plugins, threaded
    │       └── run_exploit_chain_on_port() → chain functions, sequential
    │
    └── Drain results queue → print all findings
```

All results are written to a thread-safe `queue.Queue` and printed after all threads complete.

---

## Built-in Plugins

### BannerGrabPlugin

Connects to the open port, sends an HTTP `HEAD` request, and records the first 100 characters of the response. Useful for identifying server software and version strings.

```
[INFO] Banner from 10.0.0.1:80: HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
```

### BufferOverflowPlugin

Sends a 1500-byte padding payload followed by a 4-byte marker (`\xDE\xAD\xBE\xEF`) and NOP sled. Checks whether the service crashes or returns an empty response, which may indicate a buffer overflow vulnerability.

```
[VULN] Potential buffer overflow on 10.0.0.1:9999
```

> This is a basic probe for educational and CTF use. Real-world buffer overflow testing requires service-specific offsets and controlled lab environments.

### BruteForcePlugin

Iterates over a small default credential list (`admin`, `password`, `123456`, `letmein`) and sends a `LOGIN user:<password>` probe to each open port. Reports success if the response contains `success`.

Includes a 0.5-second delay between attempts as basic rate limiting.

```
[VULN] Brute-force success on 10.0.0.1:21 with password: admin
```

> Expand the password list in `BruteForcePlugin` for more thorough testing.

---

## Writing Custom Plugins

Subclass `PluginBase`, implement the `run` method, and register the plugin before calling `execute()`.

```python
from penforge import PluginBase, PenForgeAdvanced
import queue

class MyPlugin(PluginBase):
    def run(self, target: str, port: int, results: queue.Queue):
        # Your logic here
        results.put(f"[INFO] MyPlugin ran on {target}:{port}")

framework = PenForgeAdvanced("192.168.1.10", [80, 443])
framework.register_plugin(MyPlugin())
framework.execute()
```

**Rules for plugins:**
- All output must go to `results.put(...)` — do not call `print()` directly inside plugins
- Handle all exceptions internally so one plugin failure does not block others
- Keep plugins stateless where possible — they run concurrently across ports

---

## Exploit Chains

Define exploit functions with the signature `(target: str, port: int, results: queue.Queue) -> None` and register them as a chain against a specific port:

```python
def my_exploit(target: str, port: int, results: queue.Queue):
    # exploit logic
    results.put(f"[INFO] Exploit ran on {target}:{port}")

framework = PenForgeAdvanced("192.168.1.10", [9999])
framework.add_exploit_chain(9999, [my_exploit])
framework.execute()
```

Multiple functions in a chain execute **in order**, sequentially. Use this to model multi-step exploitation sequences.

```python
framework.add_exploit_chain(9999, [
    stage_one_exploit,
    stage_two_exploit,
    post_exploitation
])
```

---

## Project Structure

```
Penforge/
├── penforge.py             # Full framework (single file)
│   ├── PenForgeAdvanced    # Core framework class
│   ├── PluginBase          # Abstract base class for plugins
│   ├── BannerGrabPlugin    # Banner grabbing plugin
│   ├── BufferOverflowPlugin # Buffer overflow probe plugin
│   ├── BruteForcePlugin    # Credential brute-force plugin
│   ├── custom_shell_exploit # Example shellcode exploit chain function
│   └── main()              # CLI entry point
└── README.md
```

---

## Troubleshooting

**`Connection refused` on all ports**
The target may have a firewall blocking the scan. Confirm the target is reachable with `ping` and that the ports are correct.

**Brute-force plugin shows no results**
The default probe (`LOGIN user:<pwd>\r\n`) is a generic format. Real services (SSH, FTP, etc.) require protocol-specific login sequences. Extend `BruteForcePlugin` with the appropriate protocol for your target.

**Buffer overflow plugin shows no vulnerability on a known-vulnerable service**
The built-in payload is a generic probe. Real buffer overflow testing requires the exact service binary, offset calculation, and a controlled environment. The included plugin is suitable for CTF-style services, not production software.

**All ports showing as closed**
Verify the target IP is correct and reachable. The default socket timeout is 2 seconds — targets with high latency may need a longer timeout (edit `sock.settimeout(2)` in `scan_port()`).

---

## Contributing

Contributions are welcome. To get started:

1. Fork the repository and create a feature branch: `git checkout -b feature/your-feature`
2. Keep pull requests focused — one plugin, fix, or feature per PR
3. Test against a local lab (DVWA, Metasploitable, HackTheBox, or a custom VM) before submitting
4. Open a pull request against `main` with a clear description

For significant architectural changes, open an issue first to discuss the approach.

### Reporting Bugs

Open a GitHub issue with:
- Python version and OS
- The exact command you ran
- Full error output
- Target environment (local VM, CTF box, etc.)

### Roadmap

- [ ] UDP port scanning support
- [ ] Service-specific brute-force modules (SSH, FTP, HTTP Basic Auth)
- [ ] Plugin discovery from a `plugins/` directory (auto-load at runtime)
- [ ] JSON output format for scan results
- [ ] Rate limiting and connection throttling options via CLI flags
- [ ] CVE lookup integration for detected service banners
- [ ] Async I/O rewrite for improved performance at scale

---

## Changelog

### v1.0.0

- Initial release
- `PenForgeAdvanced` core class with plugin and exploit chain architecture
- Concurrent port scanning via Python threads
- Thread-safe result queue
- `BannerGrabPlugin` — HTTP banner grabbing
- `BufferOverflowPlugin` — generic overflow probe
- `BruteForcePlugin` — credential testing with rate limiting
- `custom_shell_exploit` — shellcode chain example
- CLI interface (`--target`, `--ports`)

---

## References

| Resource | URL |
|----------|-----|
| OWASP Testing Guide | https://owasp.org/www-project-web-security-testing-guide/ |
| Python `socket` Documentation | https://docs.python.org/3/library/socket.html |
| Python `threading` Documentation | https://docs.python.org/3/library/threading.html |
| Metasploitable (Lab Target) | https://github.com/rapid7/metasploitable3 |
| HackTheBox | https://www.hackthebox.com |
| TryHackMe | https://tryhackme.com |

---

## License

This project is licensed under the [MIT License](LICENSE).
