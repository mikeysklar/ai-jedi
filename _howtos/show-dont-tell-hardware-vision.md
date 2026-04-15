---
title: "Show, Don't Tell — Giving Claude Eyes on Your Hardware"
description: "Closed-loop hardware debugging by piping a USB webcam and an oscilloscope straight into Claude. Real case study: landing a hardware-brightness PR on Adafruit's piomatter library."
tags: [hardware, debugging, webcam, oscilloscope, claude-code, vision, rgb-matrix]
image: /assets/images/cards/show-dont-tell-hardware-vision.svg
---

<img src="https://github.com/user-attachments/assets/75854dc8-b555-40e4-b6f4-a031aa956045" alt="Webcam pointed at a 64x64 RGB matrix panel during closed-loop debugging" style="max-width: 75%; height: auto;" />

## 👁️ TL;DR

> *"Don't describe what you see to Claude. Show it."*

Hardware bugs live in the physical world — a panel is *slightly* too bright, a waveform *almost* has the right duty cycle, a trace is *kind of* ringing. Trying to put that into words burns tokens and loses information. The fix is stupid-simple: **give Claude eyes**.

This howto walks through the two vision channels that landed a real brightness-control PR on Adafruit's [piomatter](https://github.com/adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter) library:

1. 📷 **A $40 USB webcam** pointed at the device under test, shared into Claude via a Chrome extension
2. 🔬 **A DSO Nano V3 oscilloscope**, with Claude reading CSV waveforms and screenshot BMPs directly off its USB mass-storage mount

Both close the loop. Claude writes code → runs it → *sees the result* → iterates. No "can you describe the LED color" back-and-forth.

---

## 🛠️ The Hardware

### 📷 The webcam — ELP 1080P USB Camera ($38.77)

[ELP 1080P USB Camera with Box Housing, Autofocus, 100° wide-angle, OV2710 sensor](https://www.amazon.com/dp/B00UV89ASO) — $38.77 on Amazon.

Why this one:
- **Autofocus** — dead simple, no manual focus ring to babysit
- **100° wide-angle, no distortion** — fits a 64x64 panel in frame from ~15cm away
- **UVC class** — plug-and-play on macOS/Linux/Windows, no drivers
- **100fps** — overkill for stills, but means no motion blur on scrolling animations
- **Box housing** — sits flat on a desk or tripod without a bracket

Any UVC webcam will work. Pick whichever one can autofocus close enough to see what you care about.

### 🔬 The scope — DSO Nano V3

Cheap pocket scope (~$90) with a killer feature for this workflow: **USB mass storage mode**. When you plug it into a Mac, it mounts like a flash drive. Every `SAVE` writes two files:

- `DSxxxxx.CSV` — the full waveform as sample pairs (time, voltage)
- `DSxxxxx.BMP` — a 320×240 screenshot of the scope display

Claude reads both directly. No screen-scraping, no photographing the scope face, no OCR.

---

## 🧠 The Insight: Closing the Loop on Physics

Claude Code already has a closed debugging loop for software: write code → run tests → read errors → fix. But hardware bugs don't surface through `pytest`. They surface as:

- A panel that *looks* dim but measures fine
- A waveform that *almost* has the right duty cycle
- A color that *sort of* matches the framebuffer
- A sensor that returns the right value *except at noon*

The fix is to turn those physical signals into data Claude can read:

```
┌────────────────────────────────────────────────────────────────┐
│                  VISION-DRIVEN HARDWARE LOOP                   │
│                                                                │
│   Claude writes code ──▶ runs on Pi ──▶ drives the panel      │
│                                              │                 │
│                              ┌───────────────┴──────────────┐  │
│                              ▼                              ▼  │
│                       📷 Webcam frame            🔬 Scope CSV   │
│                       (color / uniformity)      (duty cycle)   │
│                              │                              │  │
│                              └───────────────┬──────────────┘  │
│                                              ▼                 │
│                          Claude reads the result, diagnoses,   │
│                          edits the code, re-runs.              │
└────────────────────────────────────────────────────────────────┘
```

> *"The board can't lie to you. A photo of the board can't lie to you. A prose description of what you think the board is doing absolutely can."*

---

## 📖 Case Study — piomatter Hardware Brightness

The whole pipeline was forged during a real PR: adding hardware-level brightness control to Adafruit's `piomatter` RGB matrix driver for the Raspberry Pi 5.

- **Issue:** [adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter#81](https://github.com/adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter/issues/81) — Feature request: hardware-level brightness control (OE pin PWM)
- **PR:** [adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter#82](https://github.com/adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter/pull/82) — `matrix.brightness = 0.5`

The core change: PWM the HUB75 OE (output enable) pin to control how long each row is lit per scan cycle. Math lives in `render.h`:

```c
active_time = schedule_ent.active_time * brightness;
```

Simple enough. The hard part was *proving* it worked across four different panels and at the silicon level. That's where the eyes came in.

### 🎨 Phase 1 — Webcam verification across four panels

The fast way to know whether a panel dimmed was to point the webcam at it, run a script that cycled through `brightness=0.1` and `brightness=1.0` with an R/G/B/W quadrant framebuffer, and let Claude grab frames at each step.

Claude would:
1. SSH to the Pi, update the test script
2. Tell the Pi to flip the brightness
3. Ask me to snap the frame from the Chrome extension *or* grab it directly if the extension was open
4. Compare against the previous frame and the expected behavior
5. If anything looked off, edit `render.h` and re-run

Four panels, two brightness levels each, end result:

| Panel | brightness=0.1 | brightness=1.0 | Result |
|-------|:--:|:--:|:--:|
| [64x64 2mm — ADA 5362](https://www.adafruit.com/product/5362) | <img width="220" alt="ADA 5362 0.1" src="https://github.com/user-attachments/assets/43af2dc6-9f9c-433e-b87e-ab9340e8a496" /> | <img width="220" alt="ADA 5362 1.0" src="https://github.com/user-attachments/assets/47a88c1c-fbcf-4534-92c3-fa89764bd004" /> | ✅ |
| [64x32 2.5mm — ADA 5036](https://www.adafruit.com/product/5036) | <img width="220" alt="ADA 5036 0.1" src="https://github.com/user-attachments/assets/e8d49a9c-44ef-4315-a024-2238418fafa5" /> | <img width="220" alt="ADA 5036 1.0" src="https://github.com/user-attachments/assets/63bcc67b-709e-42d5-b32e-28bf1216d9ce" /> | ✅ |
| [64x32 4mm — ADA 2278](https://www.adafruit.com/product/2278) | <img width="220" alt="ADA 2278 0.1" src="https://github.com/user-attachments/assets/5b3c580d-9a23-436c-80f2-cb16c532debb" /> | <img width="220" alt="ADA 2278 1.0" src="https://github.com/user-attachments/assets/acfccc5c-3cdd-40d8-969d-e3193fb45e27" /> | ✅ |
| [64x64 3mm — ADA 4732](https://www.adafruit.com/product/4732) | <img width="220" alt="ADA 4732 0.1" src="https://github.com/user-attachments/assets/5ee1d857-7821-46b3-89ab-8c89a07f9c26" /> | <img width="220" alt="ADA 4732 1.0" src="https://github.com/user-attachments/assets/758dba3b-1004-47f0-a367-9a4da5ed66c8" /> | ✅ |

That took maybe an hour of elapsed time, almost all of it physical panel-swapping. Claude handled the code + framebuffer + frame comparison.

### 🔬 Phase 2 — Scope verification when the review asked for proof

On the PR, Adafruit's [@ladyada](https://github.com/ladyada) asked the right question:

> *"Did you verify the PWM changes duty second with a scope?"*

Visual dimming isn't the same as a duty-cycle change — a fast enough PWM at any duty *looks* dim. So: plug a DSO Nano V3 probe onto HUB75 pin 15 (OE), capture at four brightness levels, let Claude do the analysis.

(Sidebar: my first Bonnet "had a bad R2 unit" — the red2 channel connector was flaky. Claude got stuck on weird output for twenty minutes until I swapped the Bonnet. Lesson: when the software story stops making sense, suspect the hardware.)

The DSO Nano mounts as a USB disk. Every `SAVE` press drops a `.CSV` + `.BMP` pair. Claude reads the CSVs directly:

```bash
> Read DS0001.CSV from /Volumes/DSO_NANO and compute
> the average voltage and the percentage of samples below 0V.
```

It parsed 4094 samples per capture and computed the duty cycles. Results:

| Brightness | V_avg   | % LOW (OE active) | % HIGH (OE inactive) |
|-----------:|--------:|------------------:|---------------------:|
| 0.0        | +1.550V |              3.7% |                96.3% |
| 0.1        | +1.298V |              6.0% |                94.0% |
| 0.5        | +0.270V |             33.7% |                66.3% |
| 1.0        | −0.474V |             53.6% |                46.4% |

Both `V_avg` and the OE-active duty cycle move monotonically with brightness — exactly what you want. The residual ~3.7% at `brightness=0.0` is fixed protocol overhead (row addressing, latching, post-latch delay) that can't be scaled out. Claude wrote that explanation *after* reading the CSVs and noticing the residual.

The scope screenshots (BMPs) went into the PR as visual confirmation:

| brightness=0.0 | brightness=0.1 |
|:-:|:-:|
| <img width="260" alt="scope 0.0" src="https://github.com/user-attachments/assets/48dcd43d-c9ff-47b1-85c7-d3e847320146" /> | <img width="260" alt="scope 0.1" src="https://github.com/user-attachments/assets/8c4abd93-8db8-4d55-ad7e-1760ec409471" /> |

| brightness=0.5 | brightness=1.0 |
|:-:|:-:|
| <img width="260" alt="scope 0.5" src="https://github.com/user-attachments/assets/1b392fb0-278a-4a4b-9575-a54a43e1e932" /> | <img width="260" alt="scope 1.0" src="https://github.com/user-attachments/assets/e5a89777-e63e-4bbb-bf1c-8d0bbec95c61" /> |

PR got merged. 🎉

---

## 🖥️ CLI Usage — Wiring It Up

### 1️⃣ Webcam via Chrome extension

Use the [Claude in Chrome](https://claude.ai/download) browser extension. It can see whatever tab you're on. The trick is to point it at a *page that shows the webcam feed*.

Easiest option: a one-file HTML page that renders the live camera in the browser.

```bash
cat > ~/webcam.html <<'EOF'
<!doctype html>
<meta charset="utf-8">
<title>Webcam Feed</title>
<style>body{margin:0;background:#111} video{width:100vw;height:100vh;object-fit:contain}</style>
<video id="v" autoplay playsinline muted></video>
<script>
navigator.mediaDevices.getUserMedia({video: {width: 1920, height: 1080}})
  .then(s => document.getElementById('v').srcObject = s);
</script>
EOF

open -a "Google Chrome" ~/webcam.html
```

Grant the camera permission once. Now any Claude session with the Chrome extension can read the frame via screenshot or page-text tools. Tell Claude:

```
> Look at the webcam tab and tell me if the panel is showing
> the R/G/B/W quadrant framebuffer at roughly uniform brightness.
```

### 2️⃣ Oscilloscope via USB mass storage

Plug the DSO Nano V3 in, flip it to USB mode (Menu → USB Storage). On macOS it mounts under `/Volumes/`. Tell Claude where to find it:

```bash
ls /Volumes/DSO_NANO/
# DS0001.BMP  DS0001.CSV  DS0002.BMP  DS0002.CSV ...
```

Then Claude can analyze waveforms directly:

```
> Read /Volumes/DSO_NANO/DS0001.CSV. Compute:
>  - average voltage
>  - min, max
>  - percentage of samples below 0V
>  - estimated fundamental frequency
> Then read DS0002 through DS0004 and compare.
```

The CSV format is trivial — two columns (time, voltage) with a header. Claude handles the parsing with `pandas` or plain `csv`.

For the BMPs, just ask Claude to read them as images:

```
> Show me DS0001.BMP and tell me what the timebase and vertical scale are.
```

### 3️⃣ A full closed-loop cycle

Put it all together in a Claude Code session on the Pi project:

```
> Goal: add hardware brightness control to piomatter.
>
> Workflow:
>   1. Edit render.h / piomatter.h / pymain.cpp
>   2. Rebuild and push to the Pi (ssh pi5)
>   3. Run the R/G/B/W test pattern at brightness=0.1
>   4. Check the webcam tab — does the panel look dimmed uniformly?
>   5. Ask me to hit SAVE on the scope
>   6. Read /Volumes/DSO_NANO/DSxxxx.CSV and verify duty cycle
>   7. Iterate until both visual and scope check out
```

Claude will drive steps 1, 2, 3, 4, and 6 directly. Steps 5 (pressing SAVE) and panel swaps are the only humans-in-the-loop parts. Everything else is closed.

---

## 💡 Tips & Gotchas

- 🪞 **Mount the webcam on a tripod**, not a stack of books. You'll move it twenty times per session and every nudge kills the frame-to-frame comparison.
- 🔦 **Kill ambient light drift.** Daylight changes color temperature through the day — your "is this panel dim" test will drift with it. Work in a consistently lit room or close the blinds.
- 💾 **The DSO Nano's USB mount is read-only-ish.** You can read files; don't try to write to it. Unmount cleanly (macOS `diskutil eject /Volumes/DSO_NANO`) before unplugging or you'll corrupt the FAT.
- 🧪 **Always capture at the extremes first** (brightness=0.0 and 1.0). If those look wrong there's no point sampling intermediate values.
- 🔌 **Suspect the hardware second, but suspect it.** When Claude's analysis stops matching the physical world, swap the suspect cable/bonnet/probe before assuming a software bug. See "bad R2 unit" above.
- 📸 **Let Claude name the captures.** Don't manually rename `DS0001.CSV` → `brightness_0.1.csv`; tell Claude the mapping and let it track it. Much faster than doing it yourself.
- 🎯 **Use the webcam for "did it change," the scope for "did it change correctly."** Different layers of the same question.

---

## 🔗 Links

- **Adafruit piomatter issue:** [#81 — Feature request: hardware brightness](https://github.com/adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter/issues/81)
- **The PR that landed it:** [#82 — hardware brightness via OE PWM](https://github.com/adafruit/Adafruit_Blinka_Raspberry_Pi5_Piomatter/pull/82)
- **The webcam:** [ELP 1080P USB Camera, $38.77](https://www.amazon.com/dp/B00UV89ASO)
- **The scope:** [DSO Nano V3](https://www.seeedstudio.com/Dso-Nano-V3-p-1358.html)
- **Claude in Chrome extension:** [claude.ai/download](https://claude.ai/download)
- **Companion howto:** [CircuitPython MCP — Closed-Loop Hardware Debugging]({{ "/howtos/circuitpython-mcp/" | relative_url }})
