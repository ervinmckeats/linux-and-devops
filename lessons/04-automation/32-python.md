# Lesson 32. Python for operations

## Goal
Know when Bash stops being enough, and write small, reliable Python tools for ops tasks: parsing, APIs, reports.

## Bash or Python?

| Bash | Python |
|---|---|
| glue between commands | data structures, JSON/YAML, APIs |
| < ~100 lines | anything bigger or with logic |
| runs everywhere | needs Python (almost always there), libraries need a venv |

If you need arrays of dicts, retries with backoff, or HTTP calls with auth: use Python.

## Environment

Never install packages into the system Python with `sudo pip`.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests pyyaml
pip freeze > requirements.txt
deactivate

# CLI tools in their own isolated envs
pipx install httpie
```

## Basics

```python
hosts = ["web1", "web2", "db1"]           # list
ports = {"web": 80, "db": 5432}           # dict
for h in hosts:
    if h.startswith("web"):
        print(f"{h} -> {ports['web']}")

def is_big(size_mb: int, limit: int = 100) -> bool:
    return size_mb > limit

try:
    value = int("abc")
except ValueError as e:
    print(f"bad value: {e}")
```

## Files, JSON, YAML

```python
import json, yaml
from pathlib import Path

cfg = yaml.safe_load(Path("config.yaml").read_text())       # never yaml.load on untrusted input
data = json.loads(Path("state.json").read_text())
Path("out.json").write_text(json.dumps(data, indent=2))

for log in Path("/var/log/nginx").glob("*.log"):
    print(log.name, log.stat().st_size)
```

## Running commands

```python
import subprocess

result = subprocess.run(
    ["systemctl", "is-active", "nginx"],     # list form: no shell, no injection
    capture_output=True, text=True, timeout=10,
)
print(result.returncode, result.stdout.strip())

subprocess.run(["rsync", "-a", "src/", "dst/"], check=True)   # raises on failure
```

Avoid `shell=True` with user input.

## HTTP APIs

```python
import os, time, requests

def get_with_retry(url, attempts=5):
    for i in range(attempts):
        try:
            r = requests.get(url, timeout=5,
                             headers={"Authorization": f"Bearer {os.environ['API_TOKEN']}"})
            if r.status_code < 500:
                r.raise_for_status()
                return r.json()
        except requests.RequestException as e:
            print(f"attempt {i + 1} failed: {e}")
        time.sleep(2 ** i)                  # exponential backoff: 1, 2, 4, 8 s
    raise RuntimeError(f"giving up on {url}")
```

Secrets come from environment variables or a secret manager, never from the code.

## A real tool: log report

```python
#!/usr/bin/env python3
"""Top IPs and error rate from an nginx access log."""
import argparse
import re
import sys
from collections import Counter

LINE = re.compile(r'^(\S+) \S+ \S+ \[[^\]]+\] "[^"]*" (\d{3}) ')

def main() -> int:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("logfile")
    p.add_argument("-n", type=int, default=10, help="how many IPs to show")
    args = p.parse_args()

    ips, statuses, total = Counter(), Counter(), 0
    with open(args.logfile, encoding="utf-8", errors="replace") as f:
        for line in f:                      # streams: works on huge files
            m = LINE.match(line)
            if not m:
                continue
            total += 1
            ips[m.group(1)] += 1
            statuses[m.group(2)[0] + "xx"] += 1

    if total == 0:
        print("no valid lines", file=sys.stderr)
        return 1
    for ip, count in ips.most_common(args.n):
        print(f"{count:8d}  {ip}")
    print(f"5xx rate: {statuses['5xx'] / total:.2%} of {total} requests")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

## Libraries worth knowing

| Library | For |
|---|---|
| `requests` / `httpx` | HTTP |
| `pyyaml`, `json` | config files |
| `boto3` | AWS API |
| `kubernetes` | Kubernetes API |
| `paramiko`, `fabric` | SSH |
| `psutil` | CPU, memory, processes |
| `prometheus_client` | expose metrics from your tool |
| `click` / `argparse` | CLI interfaces |
| `pytest` | tests for your tools |

## Practice

1. Run the log report on the test log from lesson 11. Compare with your awk one-liners.
2. Write `check_urls.py` that reads URLs from a YAML file, checks each with a timeout and retry, prints status and latency, and exits 1 if any fails.
3. Write a script that uses `psutil` to print the top 5 processes by memory as JSON.
4. Add a `pytest` test for one function of your tool.
5. Put the tools in your Git repo with `requirements.txt` and a short README.

## Check yourself

- Why use a venv instead of `sudo pip install`?
- Why pass a list to `subprocess.run` instead of a string with `shell=True`?
- Why sleep `2 ** i` between retries instead of a fixed time?

---

<p align="left">
  <a href="media/32-python.mp4"><img width="640" src="media/32-python.gif" alt="Lesson 32 animation: Python for operations"></a>
</p>

## Next
[Lesson 33. Server hardening](../05-security/33-server-hardening.md)
