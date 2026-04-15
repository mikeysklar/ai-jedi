---
title: "CircuitPython MCP — Closed-Loop Hardware Debugging with Claude"
description: "Give Claude Code a live REPL connection to your CircuitPython board for hands-free hardware debugging, sensor reads, and iterative development."
tags: [circuitpython, mcp, hardware, repl, debugging]
image: /assets/images/cards/circuitpython-mcp.svg
---

## TL;DR — What This Does

```
┌─────────────────────────────────────────────────────────┐
│           CIRCUITPYTHON  MCP  REPL  LOOP                │
│                                                         │
│  You ──▶ Claude Code ──▶ MCP Server ──▶ Board (USB)    │
│                                                         │
│  "read the temperature"                                 │
│       │                                                 │
│       ▼                                                 │
│  Claude writes code ──▶ executes on board ──▶ reads     │
│  result ──▶ fixes errors ──▶ re-runs ──▶ done          │
│                                                         │
│  Closed-loop: Claude sees stdout + stderr in real time  │
└─────────────────────────────────────────────────────────┘
```

| | Approach |
| --- | --- |
| The old way | Write code, flash, open serial monitor, read error, alt-tab back, fix, repeat forever. |
| The MCP way | Tell Claude what you want. It writes, runs, reads errors, fixes, and re-runs — all without leaving the conversation. |

**Source repo:** [github.com/mikeysklar/circuitpython-mcp](https://github.com/mikeysklar/circuitpython-mcp)

---

## How It Works

The MCP server (`server.py`) opens a persistent USB serial connection to your CircuitPython board and exposes it as a set of MCP tools. Claude Code calls these tools directly — no copy-paste, no terminal switching.

Under the hood it uses CircuitPython's **raw REPL mode** (Ctrl-A) for reliable, machine-readable code execution. Code goes in, stdout/stderr come back, Claude acts on the results.

---

## Setup (One-Time)

### 1. Clone and install

```bash
git clone https://github.com/mikeysklar/circuitpython-mcp.git
cd circuitpython-mcp
pip install mcp pyserial
# or with uv:
uv pip install mcp pyserial
```

### 2. Verify the server runs

```bash
python server.py
# Should hang silently (waiting for MCP protocol input). Ctrl-C to stop.
```

### 3. Register with Claude Code

```bash
claude mcp add --scope user circuitpython-repl -- \
  python /ABSOLUTE/PATH/TO/circuitpython-mcp/server.py
```

Use the **absolute path** to `server.py`. Example:

```bash
claude mcp add --scope user circuitpython-repl -- \
  python ~/Documents/GitHub/circuitpython-mcp/server.py
```

`--scope user` makes it available across all your projects.

### 4. Confirm registration

```bash
claude mcp list
# Should show: circuitpython-repl
```

### 5. Check inside Claude Code

Type `/mcp` in a Claude Code session:

```
circuitpython-repl: connected
```

You're ready.

---

## MCP Tools Reference

Once connected, Claude has access to these 8 tools:

| Tool | What it does | Key args |
| --- | --- | --- |
| `list_ports` | Discover serial ports — find your board | — |
| `connect_board` | Open serial connection | `port`, `baud` (default 115200) |
| `disconnect_board` | Close serial connection | — |
| `repl_exec` | Execute Python code on the board | `code`, `timeout` (default 10s) |
| `soft_reset` | Soft reset (re-runs main.py/code.py) | — |
| `interrupt` | Ctrl-C — stop running code | — |
| `read_output` | Read pending output without sending code | `timeout` (default 2s) |
| `connection_status` | Check if connected and to which port | — |

### The core loop: `repl_exec`

This is the tool Claude uses most. It sends Python code to the board's raw REPL, waits for execution, and returns stdout and stderr separately. Claude reads the output, diagnoses issues, and calls `repl_exec` again with a fix — **closed-loop debugging**.

```
repl_exec("import board; print(dir(board))")
```

Returns:
```
[stdout]
['A0', 'A1', 'D5', 'D6', 'LED', 'NEOPIXEL', 'SCL', 'SDA', ...]
```

---

## Usage — Just Talk to Claude

You don't need to call tools manually. Just describe what you want in natural language:

```
"List the available serial ports"
"Connect to /dev/tty.usbmodem14201"
"What pins does this board have?"
"Read the temperature from the BME280 on I2C"
"The NeoPixel isn't lighting up — debug it"
"Run this code and show me any errors"
"The board seems stuck, interrupt it"
"Soft reset the board"
```

### Example: Closed-Loop Sensor Debugging

```
You:    "Connect to the board and read the BME280 temperature sensor"

Claude: [calls list_ports, finds /dev/tty.usbmodem14201]
        [calls connect_board]
        [calls repl_exec with BME280 I2C code]
        "Got an error — the I2C address is wrong. Let me scan the bus..."
        [calls repl_exec with I2C scan code]
        "Found the sensor at 0x76 instead of 0x77. Retrying..."
        [calls repl_exec with corrected address]
        "Temperature: 23.4C, Humidity: 45.2%"
```

No manual intervention needed. Claude drives the entire debug cycle.

---

## Port Names by OS

| OS | Typical port path |
| --- | --- |
| macOS | `/dev/tty.usbmodem*` |
| Linux | `/dev/ttyACM0` or `/dev/ttyUSB0` |
| Windows | `COM3`, `COM4`, etc. |

Use `list_ports` to discover the correct one — don't guess.

---

## Troubleshooting

**Permission denied on Linux:**
```bash
sudo usermod -aG dialout $USER
# log out and back in
```

**Server shows "failed" in `/mcp`:**
```bash
# Check dependencies are installed:
python -c "import mcp, serial; print('OK')"

# Check the path is absolute:
claude mcp list
```

**Board not responding:**
- Try `interrupt` (sends Ctrl-C) to break out of stuck code
- Try `soft_reset` to reboot without power cycling
- Unplug and replug the USB cable as a last resort

**Timeout on `repl_exec`:**
- Increase the timeout: Claude can pass `timeout=30` for slow operations
- Long-running loops should use `read_output` to check for pending prints

---

## Tips for Best Results

- **Start with `list_ports`** — always discover before connecting
- **Use `repl_exec` for quick tests** — `print(dir(board))`, `import os; print(os.listdir("/"))`, etc.
- **Let Claude iterate** — the whole point is closed-loop debugging. Give it the goal, not the code
- **`soft_reset` after editing code.py** — this re-runs your main script without power cycling
- **One board at a time** — the server holds a single serial connection globally

---

## Links

- **Source:** [github.com/mikeysklar/circuitpython-mcp](https://github.com/mikeysklar/circuitpython-mcp)
- **CircuitPython docs:** [docs.circuitpython.org](https://docs.circuitpython.org)
- **MCP protocol:** [modelcontextprotocol.io](https://modelcontextprotocol.io)
- **Claude Code:** [docs.anthropic.com/en/docs/claude-code](https://docs.anthropic.com/en/docs/claude-code/overview)
