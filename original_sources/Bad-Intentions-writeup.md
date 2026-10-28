# Bad Intentions - OT/IoT Challenge Writeup

## Challenge Information

- **Challenge:** Baud Intentions
- **Category:** OT/IoT
- **Target:** `http://0.cloud.chals.io:11697`
- **Description:** A legacy industrial interface is exposed online and provides access to a simulated PLC/UART console.

## Initial Access

Opening the target in a browser revealed a web-based UART terminal:

```text
SCADA-7 :: PLCTERM-4 | UART 115200 8N1 | ttyUSB0
[BOOT] PLCTERM-4 firmware 2.3.1-OT
[BOOT] UART interface ready.
[WARN] Maintenance mode: LOCKED
root@plc4:~$
```

The terminal accepts commands through the `/cmd` endpoint using a JSON POST request:

```http
POST /cmd
Content-Type: application/json

{"command":"help"}
```

## Command Enumeration

Running `help` showed the available interface:

```text
sysinfo              device & firmware info
status               PLC operational status
diag [flags]         run hardware diagnostics
mem                  memory map overview
auth <key>           authenticate session
read <register>      read a register value
clear                clear screen
help                 this message
```

### System Information

The `sysinfo` command disclosed useful device metadata:

```text
DEVICE  : PLCTERM-4 / SCADA-7 Node
FIRMWARE: 2.3.1-OT (Build 20240307)
CPU     : ARM Cortex-M4 @ 168 MHz
REGISTERS: 0x00 - 0x7F
UART    : 115200 8N1
SERIAL  : SN-7743-ALPHA
MAINT   : LOCKED
```

### PLC Status

`status` revealed the protected register range:

```text
Coil 0x01      : ON    Motor-A
Coil 0x02      : OFF   Valve-B
Reg  0x10      : 0x00A4  (pressure 164 PSI)
Reg  0x12      : 0x0031  (temp sensor, raw)
Reg  0x40      : 0x0001  (controller state: RUN)
Reg  0x42      : 0x0231  (firmware build id)
Reg  0x45      : 0x47F6  (UART RX counter)
Reg  0x4D-0x4F : [LOCKED - maintenance/debug]
```

The memory map confirmed that registers `0x4D` through `0x4F` contained maintenance/debug data:

```text
0x00-0x0F  System control
0x10-0x1F  Sensor inputs
0x20-0x2F  Coil outputs
0x40-0x49  Runtime telemetry
0x4A-0x4C  Reserved
0x4D-0x4F  Maintenance/debug [LOCKED]
0x50-0x7F  User config
```

Attempting to read a protected register without authentication failed:

```text
read 0x4D
[READ] 0x4D: ACCESS DENIED - not authenticated.
```

The diagnostic command also required authentication:

```text
diag
[DIAG] Error: unauthenticated session.
```

## Authentication

The serial number and maintenance/debug register range suggested that the authentication key was built from the exposed device information. Testing the serial-number-related format `MAINT-7743` succeeded:

```text
auth MAINT-7743
[AUTH] Key accepted. Session elevated.
```

The important clue was the serial number:

```text
SN-7743-ALPHA
```

The challenge accepted the maintenance key using the numeric serial component:

```text
MAINT-7743
```

## Unlocking the Debug Registers

Authentication alone did not immediately expose the registers:

```text
read 0x4D
[READ] 0x4D: LOCKED - run diag sequence first.
```

The diagnostic help showed that explicit output options were required:

```text
diag --help

Usage:
  diag [options]

Options:
  --verbose    show extended diagnostic output
  --raw        include raw register/debug data
  --help, -h   show this help message
```

Running the full diagnostic sequence unlocked the maintenance range:

```text
diag --verbose --raw
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

## Extracting the Flag

The three debug registers contained ASCII bytes and decoded data.

### Register `0x4D`

```text
Raw: 4B 50 4D 47 5F 43 54 46 7B 61 45 58 35 61 4B 61 54 32
Data: KPMG_CTF{aEX5aKaT2
```

### Register `0x4E`

```text
Raw: 59 44 77 6D 4F 72 6F 6E 6D 69 52 4B 76 72 35 6C 62 55
Data: YDwmOronmiRKvr5lbU
```

### Register `0x4F`

```text
Raw: 58 48 51 70 63 77 54 74 77 46 5F 64 34 6C 79 4D 7D
Data: XHQpcwTtwF_d4lyM}
```

Concatenating the three register values gives the complete flag:

```text
KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}
```

## Final Command Sequence

The shortest successful sequence was:

```text
auth MAINT-7743
diag --verbose --raw
read 0x4D
read 0x4E
read 0x4F
```

## Flag

`KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}`

