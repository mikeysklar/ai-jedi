---
title: "Claude on Apple Watch — Talk to AI from Your Wrist"
description: "Bridge Claude Code to iMessage so you can text an AI from your Apple Watch — no app install, no cloud service, just your Mac and a running session."
tags: [imessage, apple-watch, claude-code, automation, loop, macos]
image: /assets/images/cards/claude-on-apple-watch.svg
order: 4
---

```
  ┌──────────┐    iMessage     ┌──────────────┐   polls chat.db   ┌───────────────┐
  │  ⌚ Apple │ ─────────────▶ │   Messages    │ ◀──────────────── │  Claude Code   │
  │   Watch   │ ◀───────────── │   (Mac)       │ ──────────────▶  │  /loop session │
  └──────────┘    iMessage     │  chat.db      │   sends reply     └───────┬───────┘
                               └──────────────┘     via osascript          │
                                                                     ┌─────▼─────┐
                                                                     │   Claude   │
                                                                     │   (LLM)   │
                                                                     └───────────┘
```

## ⌚ TL;DR — What This Does

> *"The best interface for AI is one that's already on your wrist."*

Your Apple Watch can already send and receive iMessages. Your Mac already stores those messages in a SQLite database. This hack connects the two: a Python script polls `chat.db` for new texts you send yourself from your watch, hands them to a running Claude Code session, and fires the reply back through iMessage. Claude answers on your wrist — no app, no API key on a server, no Siri shortcut chain.

The whole thing is a single Python file, stdlib only, driven by Claude Code's `/loop` command.

**Source repo:** [github.com/mikeysklar/claude-imessage](https://github.com/mikeysklar/claude-imessage)

---

## 🧩 The Analogy

Think of it like a **walkie-talkie relay station**:

- Your **Apple Watch** is the field radio — you talk into it (type a text) and hear back (read the reply).
- **Messages.app on your Mac** is the base station — it receives the transmission and logs it to a file (`chat.db`).
- The **Python script** is the radio operator — it watches the log, picks up new transmissions, and hands them to the person who can actually answer.
- **Claude Code** is the expert in the back room — it reads the question, thinks, writes the answer, and hands it back to the radio operator to transmit.

Nobody installed anything on the watch. The watch just thinks it's texting.

---

## 🏗️ How It Works (Under the Hood)

```
 ┌──────────────────────────────────────────────────────────┐
 │                    YOUR MAC (always on)                   │
 │                                                          │
 │  ~/Library/Messages/chat.db                              │
 │  ┌────────────────────────────────────────────┐          │
 │  │  ROWID │ text              │ is_from_me    │          │
 │  │  4201  │ "what's the wea…" │ 1             │ ◀── you  │
 │  │  4202  │ "It's 72°F and…"  │ 1             │ ◀── bot  │
 │  │  4203  │ "remind me to…"   │ 1             │ ◀── you  │
 │  └────────────────────────────────────────────┘          │
 │       ▲ poll                        send ▼               │
 │  ┌────┴─────────────────────────────────┴────┐           │
 │  │       imessage-autoreply.py               │           │
 │  │  poll  → new messages as JSON             │           │
 │  │  send  → osascript → Messages.app         │           │
 │  │  commit → advance watermark               │           │
 │  └──────────────┬────────────────────────────┘           │
 │                 │                                         │
 │       ┌─────────▼─────────┐                              │
 │       │   Claude Code     │                              │
 │       │   /loop 2m tick   │                              │
 │       │   reads, thinks,  │                              │
 │       │   replies         │                              │
 │       └───────────────────┘                              │
 └──────────────────────────────────────────────────────────┘
```

The script has four commands:

| Command | What it does |
| --- | --- |
| `init` | Seed the watermark to the current max ROWID (skip history) |
| `poll` | Return new inbound messages as JSON since the watermark |
| `send '<text>'` | Fire an iMessage reply via AppleScript and record the ROWID |
| `commit` | Advance the watermark past all processed messages |

**The watermark trick:** Instead of filtering `is_from_me = 0` (which Apple breaks on self-chats — see below), the script tracks which ROWIDs it sent itself and excludes them from poll results. Everything else is inbound.

---

## 🛠️ Setup (One-Time)

### 1. Clone the repo

```bash
git clone https://github.com/mikeysklar/claude-imessage.git
cd claude-imessage
```

### 2. Install the script

Copy it somewhere stable — Claude Code will shell out to it:

```bash
cp imessage-autoreply.py ~/.claude/imessage-autoreply.py
```

### 3. Grant permissions

Your terminal (or IDE) needs two macOS permissions:

| Permission | Why | How to grant |
| --- | --- | --- |
| **Full Disk Access** | Read `~/Library/Messages/chat.db` | System Settings → Privacy & Security → Full Disk Access → add your terminal |
| **Automation (Messages)** | Send iMessages via AppleScript | First `send` will prompt automatically — click Allow |

### 4. Set your handle (if not default)

The script defaults to `mikeysklar@icloud.com`. If your self-chat uses a different handle:

```bash
export IMESSAGE_AUTOREPLY_HANDLE="your-handle@icloud.com"
```

### 5. Initialize the watermark

```bash
python3 ~/.claude/imessage-autoreply.py init
# → watermark initialized to 42017
```

This seeds the script to only process messages from *now* forward.

### 6. Test the plumbing

Send yourself a text from your phone or watch, then:

```bash
python3 ~/.claude/imessage-autoreply.py poll
# → [{"rowid": 42018, "text": "hello from watch"}]
```

Send a reply back:

```bash
python3 ~/.claude/imessage-autoreply.py send "got it"
# → sent; recorded sent_rowids=[42019]
```

Check your watch — you should see "got it" arrive as an iMessage.

---

## 🚀 Running It — The Dedicated Claude Session

The real magic is letting Claude drive the loop. Open a **dedicated terminal window** (or tmux pane) and start a Claude Code session that will stay running:

```bash
claude
```

Inside that session, kick off the loop with a tick prompt:

```
/loop 2m Poll for new iMessage texts, respond helpfully, and send the reply.

1. Run: python3 ~/.claude/imessage-autoreply.py poll
2. If the JSON array is empty, do nothing.
3. For each message, compose a helpful reply.
4. Run: python3 ~/.claude/imessage-autoreply.py send "<your reply>"
5. Run: python3 ~/.claude/imessage-autoreply.py commit
```

That's it. Claude now checks for new texts every 2 minutes, thinks about them, and replies. Leave the terminal open and walk away with your watch.

```
 You (watch): "what's the capital of montana"
     ⇩  (iMessage)
 Mac chat.db: new row arrives
     ⇩  (poll, every 2 min)
 Claude Code: sees the question, thinks
     ⇩  (send)
 Messages.app: sends "Helena" back
     ⇩  (iMessage)
 You (watch): "Helena" appears on your wrist
```

### Keeping the session alive

- **Don't close the terminal.** The loop lives inside the Claude Code process — close it and replies stop.
- **Use tmux or screen** for durability:

```bash
tmux new -s imessage-claude
claude
# start the /loop inside
# detach with Ctrl-b d
# reattach later with: tmux attach -t imessage-claude
```

- **Laptop sleep kills it.** If your Mac sleeps, the poll stops. Use `caffeinate -i` or Amphetamine to keep it awake, or just run it on a Mac mini that stays on.

---

## 🍎 The Notes-to-Self Quirk

If you text yourself enough, macOS quietly reclassifies your self-chat into "Notes to Self." When this happens, Apple marks **both** inbound and outbound messages as `is_from_me = 1`. A naive SQL filter like `WHERE is_from_me = 0` will miss every message you send from your watch.

The workaround: **don't filter on `is_from_me` at all.** Instead, when the script sends a reply, it records the ROWIDs of the rows it just created. On the next poll, those ROWIDs are excluded. Everything else is treated as inbound. Simple, and it works regardless of Apple's Notes-to-Self flag.

---

## 🧪 The attributedBody Parsing

Most modern iMessage rows leave `message.text` NULL. The actual content lives in `message.attributedBody`, which is a NeXTSTEP typedstream (not a bplist — `plistlib` will choke). The script parses just enough of the binary to extract UTF-8 text:

1. Find the `NSString` class marker
2. Skip past the `\x84\x01\x2b` type prefix
3. Read the length prefix (1 byte for short strings, multi-byte for longer)
4. Decode that many bytes as UTF-8

This covers plain text messages from a watch — which is all you need.

---

## 💡 Tips & Gotchas

- ⌚ **Keep replies short.** You're reading on a tiny screen. Claude will naturally write paragraphs — the tick prompt can include "keep replies under 2 sentences" to fix this.
- 🔋 **2-minute poll interval is a sweet spot.** Fast enough to feel responsive, slow enough to not hammer the database. You can go down to `/loop 30s` if you need near-realtime.
- 🔒 **Only works for your self-chat.** The script is hardcoded to a single handle. This is a feature, not a bug — you don't want Claude auto-replying to your boss.
- 💤 **Watermark persists across restarts.** The state file at `~/.claude/imessage-autoreply/state.json` remembers where it left off. Messages that arrived while Claude was offline won't be retroactively answered (unless you manually rewind the watermark).
- 🔧 **Debug with poll directly.** If replies aren't arriving, run `poll` manually to see what the script sees. If the JSON is empty, either no new messages arrived or the watermark is ahead of them.
- 🖥️ **Dedicated Mac recommended.** A Mac mini or old MacBook that stays awake and plugged in is ideal. Your daily-driver laptop will sleep, close lids, and lose the session.
- 📱 **Works from iPhone too.** Anything that sends an iMessage to your self-chat works — watch, phone, iPad. The watch is just the most fun because it feels like sci-fi.

---

## ⚠️ Known Limits

- **Single handle only** — wired to one self-chat (e.g., `mikeysklar@icloud.com`). Other handles (`+1234567890`, Gmail) have their own Apple quirks.
- **Requires a running Claude Code session** — close the CLI, replies stop. There's no daemon mode.
- **AppleScript `buddy` form only** — if Apple breaks this in a future macOS, the send path needs rework.
- **Plain text only** — images, tapbacks, and rich messages from the watch are ignored.
- **No conversation memory across restarts** — each `/loop` session is a fresh context. If you want continuity, you'd need to feed history back in.

---

## 🔗 Links

- **Source:** [github.com/mikeysklar/claude-imessage](https://github.com/mikeysklar/claude-imessage)
- **Claude Code docs:** [docs.anthropic.com/en/docs/claude-code](https://docs.anthropic.com/en/docs/claude-code/overview)
- **Companion howto:** [CircuitPython MCP]({{ "/howtos/circuitpython-mcp/" | relative_url }}) — another way to give Claude hands in the physical world
- **Companion howto:** [Think Out Loud]({{ "/howtos/think-out-loud-desktop-mic/" | relative_url }}) — voice input for when you're at the desk
