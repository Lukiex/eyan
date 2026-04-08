# 🐱 eyan — Domain Alive Checker

> Hunt alive domains like a cat.

**eyan** is a fast, lightweight terminal tool for checking whether domains are alive or dead. Built for security researchers, penetration testers, and bug bounty hunters running **Kali Linux**.

---

## ✨ Features

- ✅ Check a **single domain** or a **bulk list** from a file
- ⚡ **Multi-threaded** scanning for fast bulk checks
- 🎨 Clean, **color-coded terminal output** (alive/dead/redirects)
- 🌐 Shows **HTTP status code**, **latency (ms)**, and **resolved IP**
- 🔁 Follows **redirects** and shows the final destination
- 🔒 Automatic **HTTP fallback** if HTTPS/SSL fails
- 💾 **Save results** to a text file with `-o`
- 🐱 ASCII cat banner on every run

---

## 📸 Preview

```
   /\_____/\
  /  o   o  \      eyan — Domain Alive Checker
 ( ==  ^  == )     Hunt alive domains like a cat
  )         (      version 1.0  |  by Luke.SK
 (           )
  ( (  ) (  ) )
 (__(__)___(__)__)

  STATUS    DOMAIN                               CODE   LATENCY     IP
────────────────────────────────────────────────────────────────────────
✔ ALIVE   google.com                           [200]  142.3ms     142.250.80.46
✔ ALIVE   github.com                           [200]  198.1ms     140.82.121.4
✘ DEAD    notasite.xyz                         [---]  N/A         Connection refused
```

---

## 🛠️ Installation

**1. Clone the repo**
```bash
git clone https://github.com/Lukiex/eyan.git
cd eyan
```

**2. Install dependency**
```bash
pip install requests
```

**3. Make it executable and install globally**
```bash
chmod +x eyan.py
sudo cp eyan.py /usr/local/bin/eyan
```

Now you can run `eyan` from anywhere in your terminal.

---

## 🚀 Usage

```
eyan [-h] (-d DOMAIN [DOMAIN ...] | -f FILE) [-o FILE] [-t SECS] [--threads N] [--alive-only]
```

### Options

| Flag | Description |
|------|-------------|
| `-d DOMAIN` | Single domain or space-separated list of domains |
| `-f FILE` | Path to a file with one domain per line |
| `-o FILE` | Save results to an output file |
| `-t SECS` | Request timeout in seconds (default: 5) |
| `--threads N` | Number of concurrent threads (default: 10) |
| `--alive-only` | Only print alive domains to terminal |
| `-h, --help` | Show help message |

---

## 📖 Examples

**Check a single domain:**
```bash
eyan -d google.com
```

**Check multiple domains at once:**
```bash
eyan -d google.com facebook.com github.com example.xyz
```

**Scan from a file:**
```bash
eyan -f domains.txt
```

**Full options — file input, custom timeout, threads, save output:**
```bash
eyan -f domains.txt -t 8 --threads 20 -o results.txt
```

**domains.txt format:**
```
google.com
github.com
# this is a comment and will be skipped
notarealsite.xyz
facebook.com
```

---

## 📤 Output File Format

When using `-o results.txt`, the saved file looks like:

```
# eyan Domain Check Results — 2025-03-30 14:22:01
# Total: 4 | Alive: 3 | Dead: 1

=== ALIVE ===
[ALIVE] google.com    code=200  latency=142.3ms  ip=142.250.80.46
[ALIVE] github.com    code=200  latency=198.1ms  ip=140.82.121.4
[ALIVE] facebook.com  code=200  latency=310.5ms  ip=157.240.241.35

=== DEAD ===
[DEAD]  notarealsite.xyz  error=Connection refused / DNS failed
```

---

## ⚙️ Requirements

- Python 3.6+
- `requests` library (`pip install requests`)
- Linux / macOS terminal (optimized for **Kali Linux**)

---

## 📌 Use Cases

- 🔍 **Bug bounty recon** — quickly check which subdomains in a list are live
- 🛡️ **Penetration testing** — identify active targets before scanning
- 🌐 **Domain monitoring** — verify if your domains are reachable
- 📋 **Asset discovery** — filter large domain lists down to alive hosts only

---

## 👤 Author

**Luke.SK**
> Built for the terminal. Built for the hunt.

---

## ⚠️ Disclaimer

This tool is intended for **legal and authorized use only**. Always ensure you have permission before scanning domains or systems you do not own. The author is not responsible for any misuse.

---

## 📄 License

MIT License — free to use, modify, and distribute.

