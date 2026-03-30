#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
eyan - Domain Alive Checker
A sleek terminal tool for checking if domains are alive.
"""

import argparse
import sys
import os
import socket
import time
import concurrent.futures
from datetime import datetime

try:
    import requests
    requests.packages.urllib3.disable_warnings()
except ImportError:
    print("[-] Missing dependency: requests. Run: pip install requests")
    sys.exit(1)

# ─────────────────────────────────────────────────────────────
#  ANSI COLOR CODES
# ─────────────────────────────────────────────────────────────
R  = "\033[0m"        # reset
G  = "\033[92m"       # green
Y  = "\033[93m"       # yellow
RE = "\033[91m"       # red
B  = "\033[94m"       # blue
C  = "\033[96m"       # cyan
M  = "\033[95m"       # magenta
W  = "\033[97m"       # white bold
DIM = "\033[2m"       # dim
BOLD = "\033[1m"

def supports_color():
    return hasattr(sys.stdout, "isatty") and sys.stdout.isatty()

if not supports_color():
    R = G = Y = RE = B = C = M = W = DIM = BOLD = ""

# ─────────────────────────────────────────────────────────────
#  ASCII CAT BANNER
# ─────────────────────────────────────────────────────────────
BANNER = f"""
{C}{BOLD}
   /\\_____/\\
  /  o   o  \\      {W}eyan{R}{C} — Domain Alive Checker
 ( ==  ^  == )     {DIM}Hunt alive domains like a cat{R}{C}
  )         (      {Y}version 1.0{R}{C}  |  {G}by eyan{R}{C}
 (           )
  ( (  ) (  ) )
 (__(__)___(__)__)
{R}"""

# ─────────────────────────────────────────────────────────────
#  HELPERS
# ─────────────────────────────────────────────────────────────

def print_banner():
    print(BANNER)

def print_separator(char="─", length=60, color=DIM):
    print(f"{color}{char * length}{R}")

def normalize_domain(domain: str) -> str:
    domain = domain.strip().lower()
    if domain.startswith("http://") or domain.startswith("https://"):
        return domain
    return f"https://{domain}"

def resolve_ip(domain: str) -> str:
    """Try to resolve the hostname to an IP address."""
    try:
        host = domain.replace("https://", "").replace("http://", "").split("/")[0]
        return socket.gethostbyname(host)
    except socket.gaierror:
        return None

def check_domain(domain: str, timeout: int = 5) -> dict:
    """
    Check if a domain is alive.
    Returns a result dict with status, code, ip, redirect, and latency.
    """
    url = normalize_domain(domain)
    display = domain.strip()
    result = {
        "domain": display,
        "url": url,
        "alive": False,
        "status_code": None,
        "ip": None,
        "redirect": None,
        "latency_ms": None,
        "error": None,
    }

    ip = resolve_ip(url)
    result["ip"] = ip if ip else "unresolved"

    try:
        start = time.time()
        response = requests.get(
            url,
            timeout=timeout,
            allow_redirects=True,
            verify=False,
            headers={"User-Agent": "eyan-checker/1.0"}
        )
        elapsed = round((time.time() - start) * 1000, 1)

        result["alive"] = True
        result["status_code"] = response.status_code
        result["latency_ms"] = elapsed

        if response.url != url:
            result["redirect"] = response.url

    except requests.exceptions.SSLError:
        # Try HTTP fallback
        try:
            http_url = url.replace("https://", "http://")
            start = time.time()
            response = requests.get(
                http_url,
                timeout=timeout,
                allow_redirects=True,
                verify=False,
                headers={"User-Agent": "eyan-checker/1.0"}
            )
            elapsed = round((time.time() - start) * 1000, 1)
            result["alive"] = True
            result["status_code"] = response.status_code
            result["latency_ms"] = elapsed
            result["error"] = "SSL issue (used HTTP)"
        except Exception as e:
            result["error"] = f"SSL + HTTP failed: {str(e)[:40]}"

    except requests.exceptions.ConnectionError:
        result["error"] = "Connection refused / DNS failed"
    except requests.exceptions.Timeout:
        result["error"] = f"Timed out after {timeout}s"
    except Exception as e:
        result["error"] = str(e)[:60]

    return result

# ─────────────────────────────────────────────────────────────
#  DISPLAY
# ─────────────────────────────────────────────────────────────

def status_color(code):
    if code is None:
        return RE
    if 200 <= code < 300:
        return G
    if 300 <= code < 400:
        return Y
    if 400 <= code < 500:
        return M
    return RE

def print_result(r: dict, index: int = None):
    prefix = f"{DIM}[{index}]{R} " if index is not None else ""

    if r["alive"]:
        sc = r["status_code"]
        col = status_color(sc)
        latency = f"{r['latency_ms']}ms" if r["latency_ms"] else "?"
        ip_str = f"{DIM}{r['ip']}{R}" if r["ip"] != "unresolved" else f"{RE}{r['ip']}{R}"

        print(f"{prefix}{G}✔ ALIVE   {R}{W}{r['domain']:<35}{R}  "
              f"{col}[{sc}]{R}  {C}{latency:<10}{R}  {ip_str}")

        if r.get("redirect"):
            print(f"  {DIM}↳ redirected → {r['redirect']}{R}")
        if r.get("error"):
            print(f"  {Y}⚠  {r['error']}{R}")
    else:
        err = r.get("error") or "unknown error"
        print(f"{prefix}{RE}✘ DEAD    {R}{DIM}{r['domain']:<35}{R}  "
              f"{RE}[---]{R}  {DIM}{'N/A':<10}{R}  {RE}{err}{R}")

def print_summary(results: list, elapsed: float):
    alive = [r for r in results if r["alive"]]
    dead  = [r for r in results if not r["alive"]]

    print_separator()
    print(f"\n{BOLD}  Summary{R}")
    print(f"  Total   : {W}{len(results)}{R}")
    print(f"  {G}Alive   : {len(alive)}{R}")
    print(f"  {RE}Dead    : {len(dead)}{R}")
    print(f"  Elapsed : {Y}{elapsed:.2f}s{R}\n")

    if alive:
        avg_lat = sum(r["latency_ms"] for r in alive if r["latency_ms"]) / max(len(alive), 1)
        print(f"  Avg latency (alive) : {C}{avg_lat:.1f}ms{R}\n")

def print_header():
    print(f"\n  {BOLD}{W}{'STATUS':<10}{'DOMAIN':<37}{'CODE':<7}{'LATENCY':<12}IP / ERROR{R}")
    print_separator()

# ─────────────────────────────────────────────────────────────
#  SAVE RESULTS
# ─────────────────────────────────────────────────────────────

def save_results(results: list, output_file: str):
    alive = [r for r in results if r["alive"]]
    dead  = [r for r in results if not r["alive"]]
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    with open(output_file, "w") as f:
        f.write(f"# eyan Domain Check Results — {ts}\n")
        f.write(f"# Total: {len(results)} | Alive: {len(alive)} | Dead: {len(dead)}\n\n")

        f.write("=== ALIVE ===\n")
        for r in alive:
            f.write(f"[ALIVE] {r['domain']}  code={r['status_code']}  "
                    f"latency={r['latency_ms']}ms  ip={r['ip']}\n")

        f.write("\n=== DEAD ===\n")
        for r in dead:
            f.write(f"[DEAD]  {r['domain']}  error={r.get('error', 'unknown')}\n")

    print(f"\n{G}✔ Results saved to: {W}{output_file}{R}")

# ─────────────────────────────────────────────────────────────
#  CORE RUNNERS
# ─────────────────────────────────────────────────────────────

def run_single(domain: str, timeout: int):
    print_banner()
    print(f"  {C}Checking:{R} {W}{domain}{R}\n")
    print_header()
    r = check_domain(domain, timeout)
    print_result(r)
    print_separator()
    status = f"{G}ALIVE{R}" if r["alive"] else f"{RE}DEAD{R}"
    print(f"\n  Result: {status}\n")

def run_list(domains: list, timeout: int, threads: int, output_file: str = None):
    total = len(domains)
    results = []

    print_banner()
    print(f"  {C}Scanning {W}{total}{R}{C} domain(s) with {W}{threads}{R}{C} threads{R}\n")
    print_header()

    start_time = time.time()

    with concurrent.futures.ThreadPoolExecutor(max_workers=threads) as executor:
        future_map = {executor.submit(check_domain, d, timeout): i
                      for i, d in enumerate(domains, 1)}

        for future in concurrent.futures.as_completed(future_map):
            idx = future_map[future]
            try:
                r = future.result()
            except Exception as e:
                r = {"domain": domains[idx - 1], "alive": False,
                     "status_code": None, "ip": None,
                     "latency_ms": None, "error": str(e)}
            results.append(r)
            print_result(r, index=idx)

    elapsed = time.time() - start_time
    print_summary(results, elapsed)

    if output_file:
        save_results(results, output_file)

    return results

# ─────────────────────────────────────────────────────────────
#  CLI ARGUMENT PARSER
# ─────────────────────────────────────────────────────────────

def build_parser():
    parser = argparse.ArgumentParser(
        prog="eyan",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        description=f"""{C}eyan — Domain Alive Checker{R}
  Check if a single domain or a list of domains is alive.

{Y}Examples:{R}
  eyan -d google.com
  eyan -d google.com facebook.com github.com
  eyan -f domains.txt
  eyan -f domains.txt -o results.txt -t 10 --threads 20
""",
        epilog=f"{DIM}Hunt alive domains like a cat.{R}"
    )

    target = parser.add_mutually_exclusive_group(required=True)
    target.add_argument(
        "-d", "--domain",
        nargs="+",
        metavar="DOMAIN",
        help="Single domain or space-separated list of domains to check"
    )
    target.add_argument(
        "-f", "--file",
        metavar="FILE",
        help="Path to a file containing one domain per line"
    )

    parser.add_argument(
        "-o", "--output",
        metavar="FILE",
        help="Save results to a file"
    )
    parser.add_argument(
        "-t", "--timeout",
        type=int,
        default=5,
        metavar="SECS",
        help="Request timeout in seconds (default: 5)"
    )
    parser.add_argument(
        "--threads",
        type=int,
        default=10,
        metavar="N",
        help="Number of concurrent threads for list mode (default: 10)"
    )
    parser.add_argument(
        "--alive-only",
        action="store_true",
        help="Only print alive domains in output (still saves all to file)"
    )

    return parser

# ─────────────────────────────────────────────────────────────
#  ENTRY POINT
# ─────────────────────────────────────────────────────────────

def main():
    parser = build_parser()
    args = parser.parse_args()

    if args.domain:
        domains = args.domain
    elif args.file:
        if not os.path.isfile(args.file):
            print(f"{RE}[!] File not found: {args.file}{R}")
            sys.exit(1)
        with open(args.file, "r") as f:
            domains = [line.strip() for line in f if line.strip() and not line.startswith("#")]
        if not domains:
            print(f"{RE}[!] No domains found in file: {args.file}{R}")
            sys.exit(1)

    if len(domains) == 1:
        run_single(domains[0], args.timeout)
    else:
        run_list(domains, args.timeout, args.threads, args.output)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print(f"\n{Y}[!] Interrupted by user. Goodbye!{R}\n")
        sys.exit(0)
