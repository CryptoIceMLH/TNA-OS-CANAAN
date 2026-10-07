# TNA-OS Canaan

### The firmware your Avalon was supposed to ship with.

**Version 0.3.21** · Avalon Nano 3s · Avalon Q

---

Stock firmware treats your miner like an appliance. TNA-OS treats it like a **machine you
own.**

It rips out the stock software down to the metal and rebuilds the whole thing — the mining
engine, the dashboard, the network stack, the safety system — into one fast, modern,
self-hosted operating system. No cloud account. No phone-home. No leash. It boots straight
into a slick web control centre served **by the miner itself**, and it answers to nobody but
you.

One firmware. Both machines. Total control.

> **⚡ Flash it now → [TNA Flasher](https://lnbits.molonlabe.holdings/tnaflasher/public)**
> · 📊 [Nano 3s engineering report](https://github.com/CryptoIceMLH/Nano3S)
> · 📊 [Avalon Q engineering report](https://github.com/CryptoIceMLH/AvalonQ)

> **🧪 Honest heads-up:** TNA-OS Canaan is under active, ongoing development. The core mining,
> monitoring, pool, power and safety features below are solid and in daily use — but anything
> tagged **🧪 experimental** is still being hardened and may change or misbehave build to build.
> The hype is real; where something's still cooking, it says so.

---

## This is what control feels like

**160 hash chips. All the power in your hands.**
The Avalon Q packs 160× A3197S — over **19,000 compute cores** of SHA-256 muscle. TNA-OS
gives you full control of the chain: global frequency, voltage and presets — plus
**🧪 per-ASIC tuning** to walk a single chip to its own clock while the rest hold steady.
Balance a board. Nurse a weak die. Push a strong one. This is control stock firmware won't
even show you.

> 🧪 **Per-ASIC tuning — manual and auto-tune — is experimental.** Setting a chip's clock by
> hand, and the auto-tuner that does it for you, are two halves of the same work-in-progress:
> the capability is live and genuinely powerful, but still being hardened build to build.
> **Global frequency, voltage and presets are the rock-solid, everyday path**; treat per-chip
> tuning (manual or auto) as the bleeding edge.

**Sub-second telemetry. Watch every core breathe.**
Live hashrate. Per-chip die temperatures. Per-chip health. Four independent fan tachometers
on the Q, each read live. Power draw, efficiency in J/TH, share counts per pool — refreshed
**every second**, smoothed across 1-minute, 10-minute, hourly and daily windows, and drawn
straight onto built-in history charts. No external logger. No spreadsheet. Just open a
browser and *see everything.*

**Stratum V2, native.**
Run up to **8 pools at once** — V1 and V2 side by side. Failover that catches a dead pool
without dropping a beat, or quota-weighted multipool that splits your hashpower exactly how
you want and instantly redistributes it when a pool falls over. Modern protocol, modern
routing, zero compromise.

**Kill the rail. Bring it back. No reboot.**
Cut power to the hashboard and the *brain stays alive* — dashboard, network, telemetry, all
still up. Bring it back and watch voltage and frequency ramp from a safe floor and reconnect
to your pools with a fresh session, automatically. Instant on/off for the part that burns
watts, zero downtime for the part that runs the show.

**A safety system that never sleeps.**
A layered thermal ladder that ramps the fans, then steps the clock down, then slams the board
into reset before anything cooks. A hardware watchdog that resurrects a stuck board on its
own. A PID fan controller holding your target temperature to the degree. Immersion-cooling
mode for the tank builds. It's engineered to run **unattended, for months, and not care.**

**Bitcoin only — or it stops.**
Point a pool at non-Bitcoin SHA-256 work and TNA-OS catches it, throttles to a safe idle
clock, and lights up a warning on the screen and the dashboard. Checked *per pool*, so a
decoy can't sneak past. Your hashpower mines what you told it to. Full stop.

**Open to everything. Owned by you.**
A clean REST/JSON API on the same address as the dashboard. A cgminer-compatible endpoint so
your existing tools just work. Home Assistant in three lines of YAML.
And the whole thing runs **100% local** — pull the internet cable and it keeps right on
mining and reporting.

---

## Two machines, one brain

| | **Avalon Nano 3s** | **Avalon Q** |
|---|---|---|
| Hash chips | 12× A3197S | 160× A3197S |
| Compute cores | ~1,440 | ~19,200 |
| Power | USB-C PD | Mains PSU (~4 kW class) |
| Class | Quiet home unit | Full-size home miner |

Same firmware. Same dashboard. Same API. Same controls. The Q and the 3s run **identical
software from the same source** — the only things that differ are the numbers the hardware
reports back. Learn one, you know both.

---


# User Manual

Everything below is the practical, do-it guide. Skim the headers, jump to what you need.

## Getting started

### 1 · Flash the firmware

Flash your miner straight from your browser — no command line needed:

**👉 [TNA Flasher — lnbits.molonlabe.holdings/tnaflasher/public](https://lnbits.molonlabe.holdings/tnaflasher/public)**

First, grab your self a USB A male cable (other end depends on you pc)
On Windows, the browser can only claim the miner if the WinUSB driver is bound to 29F1:0230. If the Flash step can't find the device: install <a href="https://zadig.akeo.ie/" target="_blank">Zadig</a>, select the 29F1:0230 device, choose WinUSB, and click Replace/Install Driver — reboot. macOS and Linux need no driver step.

Then put the miner into flashing (BOOT) mode:
1. Power the miner **off**.
2. Hold the small recovery button next to the USB port on Q or in the pin hole for nano3s.
3. Keep holding it and connect the USB cable to your computer.
4. Power the miner **on**, then release the button after about 2 seconds.
5. Open the TNA Flasher, pick the image for **your board**, and follow the steps.
6. The miner reboots itself- when it does power off 
7. Remove your usb cable
8. insert wifi dongle or network cable
9. Power you miner on

> **Always flash the image that matches your board** — Nano 3s and Avalon Q are different
> images.

### 2 · Find your miner

- On first boot the miner pulls an IP address from your router automatically.
- Get that address from your router's device list, or just read it off the miner's
  screen.
- Open **`http://<miner-ip>/`** in any browser. That's your dashboard.

### 3 · Get it online (WiFi)

Skip this if you're on wired ethernet. Otherwise, a fresh miner broadcasts its own setup
network:
1. Connect your phone or laptop to the **`TNA-Setup`** WiFi network.
2. A pop up should open on mobile devices, For pc open  `http://192.168.4.1/`.
3. Enter your home WiFi name and password.
4. The miner saves them and reboots onto your network. Done.

---

## Using the web interface

The UI has three main pages: the **Dashboard** (live monitoring), **Settings** (all
configuration), and **System** (power, reboot, live logs).

Every page's card layout is **yours to arrange** — hit **✎ Edit layout** (top-left of the
page), then drag a card by its header and resize it from the bottom-right corner. Your layout
is saved in that browser, per machine; **↺ Reset** restores the default. Telemetry refreshes
about every 1–3 seconds.

---

## The Dashboard (live monitoring)

Each card is a tile you can move or resize. Here's what every one shows.

### Hash Rate
The headline number is the **1-minute rolling average** in TH/s (derived from accepted-share
difficulty). 1 m reacts fast but is noisy; the **10-minute** figure is the smoother trend — use
it for a stable read. **Expected** is the theoretical rating at your current frequency
(chip rating × MHz) — a reference, not a chip reading; a healthy miner usually beats it.

### Miner Status
Live gauges, updated every second.

**BDOC** and **IMMERSION** badges light up when those modes are active.

### Pools
A status badge per pool — connected / disconnected / dead — with its quota %.

### Shares
**Accepted / Sent / Rejected** since the daemon started, plus a per-pool breakdown. 

### Best Difficulty
The highest-difficulty share this miner has ever found ("since boot"). If it ever equals the current Bitcoin network difficulty, **you've found a block.**

### Bitcoin Network
Live network stats — block height, difficulty, estimated network hashrate — from a public
source (not your pool - on the to do to allow user to point it to their own mempool instance). Click the rows to toggle between sats/min ↔ solo-block odds and fiat ↔
BTC views.

### Hash Rate Chart
Your hashrate over time with selectable **1m / 10m / 1h / 1d** traces plus **temperature** and
**fan** overlays (click the coloured legend dots to toggle each). If a pool ever serves
non-Bitcoin work, this card takes over with a **SHITCOIN DETECTED** notice — the miner throttles to 100 MHz until you point it back at a Bitcoin pool.

### Hardware Schematic
The per-board and per-chip view — and the hashboard **Power ON / OFF** buttons. Covered next.

---

## The Hardware Schematic (per-board & per-chip)

Each hashboard is drawn as a grid of chips.

- **Chip display dropdown** (per board): choose what every chip cell shows — **Health**,
  **Hashrate**, **Temp °C**, **Freq MHz**, or **Chip ID**. Click a single chip to override just
  that one.
- **Per-board controls:**
  - **↻ Reset** — a 200 ms reset pulse to that board; chips re-init and mining restarts (asks to
    confirm first).
  - **Enabled / Disabled** 🧪— Disabled holds the board in reset until you re-enable it.
  - **Freq override**🧪 — run this board at its own frequency (slider 200–600 MHz) instead of
    following the global setting.
  - **✓ Apply / Cancel** — board controls *stage* your change; Apply pushes it to hardware.
- **Chip temperature bars** show the coldest and hottest die on the board, plus hashboard air
  **intake** (cold side) and **exhaust** (hot side). A ⚠ marks a board whose temp probe stopped
  responding. Each board also shows its state (ACTIVE / DISABLED / RE-INIT / OVERHEAT / FAILED /
  EMPTY) and thermal zone (WARN / HOT / DANGER).

### 🧪 Per-chip frequency editor — WIP, don't use
The schematic has a **⚙ Per-chip freq** button that the UI itself tags **"WIP — Don't use."** It
opens a grid where each chip can be set to **Global** or a fixed **100–600 MHz** (0 = follow the
global frequency; re-clocks slew at 9 MHz/s), with Apply / Cancel / **Reset to global**. It's
locked while Auto-Tune is running. **Per-chip clocking is not yet validated on hardware — only
the global frequency is proven.** Treat it as experimental; use the global frequency for real
mining.

---

## Settings — performance tuning

The **Settings** page (titled *MINER CONFIGURATION*) holds all configuration. Most controls
**stage** together and are pushed by the top **Save** button. A red **"Reboot required after
save"** appears when a change (hostname, pool mode, auto power-on..) needs a
reboot to take effect.

**Complexity levels.** A small ⚠ toggle in the Mining card header switches the UI between
**Safe**, **Advanced** and **Pro** levels — single-click cycles Safe ↔ Advanced, double-click
jumps to Pro. Safe gives you dropdowns of factory-tested values; Advanced lets you free-type
numbers and reveals the Difficulty card; Pro unlocks BDOC (below).

- **Frequency (MHz)** — the target clock all boards ramp to. Higher = more hashrate, more heat,
  more power. In Safe mode you pick from the chip's factory profile (unsafe values show in red);
  in Advanced you can type any value.
- **Voltage** — labelled **Core Voltage (mV)** on the Nano 3s, **String Voltage (V)** on the
  Avalon Q (21.5–26.0 V across the 10-chip series string). **It must match the factory profile
  for your chosen frequency** — too low starves the chips, too high can damage them. Only change
  it if you know the V/F table.
- **Custom presets** — save the current frequency + voltage pair under a name and recall it in
  one click later. Pick a preset to load it into the form, then **Save** to apply; **Delete**
  removes it.
- **PSU Mode (Nano 3s only)** — **USB-C PD** (power capped at 140 W) or **Bypass** (you feed your
  own DC supply and enter its inject voltage, 26–30 V — used for the power estimate only).

### 🔥 BDOC — expert overclocking (Pro level)
**BDOC (Balls-Deep Overclocking)** removes *all* frequency and voltage safety clamps for full
manual control. It's gated behind three escalating confirmations (the last one literally
"FUCK AROUND AND FIND OUT"), because **wrong values can instantly destroy chips or brick the
board.** With BDOC on, the normal Warn/Hot/Danger ladder is disabled and a single **BDOC
overheat temp** (40–120 °C) is the only reset trigger. Experts only, entirely at your own risk.

---

## Settings — pools & work distribution

**Pool Mode** decides how your hashpower is shared:

- **Fallback (Primary + Backup)** — mine on Pool 1; if it dies, auto-switch to Pool 2, and
  switch back when Pool 1 recovers. Only the first two pools are used.
- **Multipool (up to 8 pools with quotas)** — split hashrate across up to 8 pools by quota.

**Adding pools.** In multipool mode, **Add Pool** creates another slot (up to 8; you can delete
down to a minimum of 2). Each pool has:

| Field | What to enter |
|---|---|
| **Stratum Host** | Pool hostname only — no `stratum+tcp://` prefix, no port. |
| **Stratum Port** | The pool's TCP port (commonly 3333, 4444, 25). |
| **Username** | Your worker/username — usually your BTC address, optionally with a `.worker` suffix. |
| **Password** | Most pools accept `x`. |
| **Quota** | 1–100. This pool gets **Quota ÷ (sum of all quotas)** of the hashrate. Equal values = even split; e.g. **2 : 1 : 1 = 50% / 25% / 25%**. |
| **Protocol** | **Auto-detect** (tries V2, falls back to V1), **Stratum V1**, or **Stratum V2**. |
| **Pool Authority Key** | **V2 only** — the pool's authority public key, from its Stratum V2 connection page. Without it the miner can't verify the pool. |

Extras: **Enable Extranonce Subscribe** (V1 pools push fresh work without reconnecting) and
**Stratum TCP Keepalive** (heartbeat for pools that drop idle clients — safe to leave on).

**What happens when a pool goes down.** In **multipool**, that pool's quota is **redistributed to
the survivors** automatically — the per-pool card shows the *effective* quota next to your
*configured* quota so you can see it happen. In **fallback**, the miner moves to the backup and
returns to the primary when it recovers. A dead pool shows a grace-period alert, then
"not connected".

> A worker the pool doesn't recognise shows up as rejects — that's an account / worker-name issue
> with the pool, not the miner.

---

## Settings — fans & cooling

**Fan Controller** runs in one of two modes:

- **Manual** — two duty sliders (0–100%). On the Q they're **Front Fans (1-2)** and **Back Fans
  (3-4)**; in immersion mode they become **Pump** and **Radiator**. Below **40%** you get a
  ⚠ *Danger* badge (usually too little airflow for normal mining); at **100%** a *MAX* badge
  (no headroom left for the thermal ladder to push harder).
- **PID (Auto)** — set a **Target Temp** (°C, 30–90) and the controller holds the *hottest* board
  at that temperature by varying fan duty. The **P / I / D** gains come pre-tuned; only touch them
  if you see fans hunting (oscillating) or responding slowly — **P** = how hard fans react, **I**
  = corrects long-term drift, **D** = damps rapid swings.
- **Invert Fan Polarity** (Pro) — flip this if a fan runs backwards (0% spins fast, 100% slow).

---

## Settings — thermal thresholds (the temperature cards)

This is the per-board **thermal ladder** in Normal mode — each a °C value (40–100):

| Threshold | Default | What happens at it |
|---|---|---|
| **Warn** | 75 °C | Fans forced to **100%** and a warning is logged. |
| **Hot** | 80 °C | That board's frequency is cut **−25 MHz**. |
| **Danger / Reset** | 85 °C | That board is **reset and held in reset until you re-enable it** — other boards keep mining. |

Plus:
- **Immersion offset** (default **+15 °C**) — added to all three thresholds when immersion mode
  is on.
- **BDOC reset** (50–120 °C) — the single trigger used while BDOC is active (the Warn/Hot ladder
  is disabled then).
- **Reboot cooldown** (default **50 °C**) — before a full reboot, the miner waits for every board
  to cool to this temperature (hard timeout 120 s).
- **HB-Out trip** (Avalon Q, off by default) — also fire the Danger cutoff on the hashboard-outlet
  (hot-side) sensor, on top of the ASIC die temps.

Live, each board shows its zone (WARN / HOT / DANGER) and toasts as it crosses each step
("Entering HOT — freq dropping", "DANGER — board reset imminent", "Cooled, back to NORMAL").
**Save thresholds** applies immediately.

---

## Settings — behaviour, network & display

**Behaviour**
- **Auto Power-On** — ON = energise the rail and start mining at boot; OFF = boot **idle** (UI and
  telemetry up, hashboard off until you press Power ON). The Avalon Q ships **OFF**, the Nano 3s
  ships **ON**. Takes effect on the next boot (needs a reboot).
- **Immersion Mode** — dielectric-fluid cooling: raises every thermal threshold by the offset and
  remaps PWM so one channel drives a fixed-duty **pump** and another a **PID radiator fan**.
  Sub-config: pump channel + duty, radiator channel, and temp offset.
- **Ignore Temp Sensor Fault** — keep hashing through temp-sensor (I²C) errors. Use **only** if the
  probe is genuinely absent — with a real probe, a fault means a real cooling problem.

**Difficulty (Advanced/Pro)** — **Initial Suggested Stratum Difficulty** (default 1000; the pool
has final say), plus **Job Interval** (default 20 ms), **VR-Frequency**, and **Proxy Miner
Difficulty** (default 10000). Leave these alone unless you know why you're changing them.

**Network** — **Hostname** (shown in the header and pool worker reports; reboot to apply) and
**IP Assignment**: **DHCP** (automatic, recommended) or **Static** (you set IP / gateway / subnet
/ DNS). Static applies after a reboot. The card also shows your Ethernet status, IP and MAC.

**Display & lights**
- **Display (OLED/LCD)** — Screen On/off, brightness (40–255), push a text **message to the
  screen**, and **rotate 90°**.
- **LED Bar (ARGB)** — by default the LED follows the thermal zone (green → yellow → orange →
  red). Tick **Override** to set a fixed colour instead; the brightness slider (0–255) applies
  either way — so you can keep the thermal colours but dim them. **DANGER red is always full
  brightness** as a safety signal.

---

## 🧪 Per-ASIC tuning & Auto-Tune (experimental)

Two surfaces, both flagged **WIP — Don't use** in the UI itself:
- The **per-chip frequency editor** on the schematic (above).
- The **Auto-Tune** card on Settings — *"Auto-Tune (per-chip frequency + voltage)"* — with goals
  **Hold Temperature**, **Hold Power**, **Max Hashrate** and **Max Efficiency**. It drives per-chip
  frequency and Vcore, locks manual tuning while active, and only runs in the Normal thermal zone
  (it never overrides the safety ladder).

Per-chip clocking isn't validated on hardware yet, so both are experimental on purpose — **use the
global frequency/voltage for real mining** and leave Auto-Tune disabled for now.

---

## The System page — power & maintenance

- **Power ON** — energise the ASIC rail, enumerate the chips and start mining (~15 s). The control
  board and UI are already up.
- **Power OFF** — stop mining and cut the rail. The control board, UI, network and telemetry stay
  fully online — only the "heater" goes off. (Asks to confirm.)
- **Reboot Control Board** — a full reset of the K230D (both cores), ~60–90 s of downtime.
  Required for boot-time settings like Auto Power-On.
- **Realtime Logs** — a live log stream you can filter by source (main, chain, controller, rx,
  stratum, psu, safety, api, config), flip to **Errors only**, text-filter, pause, and **capture +
  download** for support.

> The hashboard is a heater you switch on and off independently of the "brain" —
> Power OFF stops the burn while everything else stays online, and Power ON ramps voltage and
> frequency back up from a safe floor and reconnects your pools automatically.

---

## Saving changes

Most Settings controls stage together — click **Save** at the top of the page to push and persist
them. A red **"Reboot required after save"** means the change needs **Reboot Control Board** to
take effect. A few controls save on their own dedicated buttons and apply immediately: **Thermal
thresholds**, **Immersion config**, **LED**, **Display/OLED**, and **Presets**.

---

## Automation & integrations

TNA-OS serves a full REST/JSON API right alongside the dashboard:

```bash
# Read everything
curl http://<miner-ip>/api/system/info

# Change frequency + voltage
curl -X PATCH http://<miner-ip>/api/system \
  -H "Content-Type: application/json" \
  -d '{"frequency": 300, "coreVoltage": 3650}'
```

Great for custom dashboards, fleet tools, and **Home Assistant** (a REST sensor is three
lines of YAML). There's also a **cgminer-compatible** endpoint so existing miner-monitoring
software works with little or no change.

Full reference — every field, every endpoint, ready-to-paste recipes — is in the
**[API reference](https://github.com/CryptoIceMLH/TNA-OS-CANAAN/blob/main/API.md)**.

---

## Security — read this

The miner's API is **open by design** so any app on your network can monitor and control it.
That's powerful, and it means:

- ✅ **Run TNA-OS on a trusted home network.**
- ⛔ **Never expose the miner straight to the internet.** For remote access, use your own VPN
  or firewall.
- Anyone who can reach the miner on your LAN can read its telemetry and change its settings.

Your pool credentials live on the miner and are **never** returned by the API or written to the logs.

---

## Troubleshooting

| Symptom | What to do |
|---|---|
| Can't find the miner | Check your router's device list, or read the IP off the OLED. |
| Dashboard won't load | Confirm the IP, then hard-refresh the browser (clear site data). |
| No WiFi yet | Join the `TNA-Setup` network and provision it (see Getting Started). |
| Low hashrate right after Power ON | Normal — give it a few seconds; voltage and frequency ramp up from a safe floor. |
| Shares getting rejected | Usually a pool account / worker-name issue — double-check your worker with the pool. |
| Miner throttled with a warning | A pool may be serving non-Bitcoin work (shitcoin protection). Check your pool URLs. |
| Running hot | Fans go to 100% automatically; check airflow and ambient temp. Danger temp holds the board in reset until it cools. |

---

## What's next — and what's on the bench 🧪

TNA-OS Canaan is under active development. Everything in the feature list above is **shipping
and working today, except the items tagged 🧪**. Here's what's still cooking:

- **🧪 Per-ASIC tuning (manual + auto-tune)** — live now, but experimental; the manual per-chip
  editor and the auto-tuner that drives it are being hardened build to build.
- **🧪 Solar integration** — mine straight from your solar/battery setup, with PV, battery and
  charger telemetry built into the dashboard. The plumbing is in; still experimental.
- **🧪 Heat capture** — put your miner's waste heat to work (space and water heating). Early
  days, experimental.
- **Dashboard polish** — ongoing refinement of the editable layout and per-chip views.

The 🧪 features are in the firmware and improving fast — powerful, but treat them as
experimental until they lose the flask.

---

## Resources

- **⚡ Flash your miner:** [TNA Flasher](https://lnbits.molonlabe.holdings/tnaflasher/public)
- **📊 Engineering report — Avalon Nano 3s (A3197S ASIC):** [github.com/CryptoIceMLH/Nano3S](https://github.com/CryptoIceMLH/Nano3S)
- **📊 Engineering report — Avalon Q:** [github.com/CryptoIceMLH/AvalonQ](https://github.com/CryptoIceMLH/AvalonQ)
- **🔌 API reference:** [API.md](https://github.com/CryptoIceMLH/TNA-OS-CANAAN/blob/main/API.md)

---

## Mission & support

The mission is to liberate Bitcoin infrastructure from corporate dependencies through energy
independence, sovereign communications, and experimental technology.

If you find TNA-OS useful and would like to support continued research and development, please
consider supporting:

**👉 [molonlabe.holdings/#funding](https://www.molonlabe.holdings/#funding)**

Every sat helps. ⚡

---

**This build:** v0.3.16 — confirmed working on both the Avalon Nano 3s and the Avalon Q.

**TNA-OS Canaan — your miner, your rules.**
