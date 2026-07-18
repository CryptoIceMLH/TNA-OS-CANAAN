# TNA-OS (Canaan Edition) — HTTP API Reference

**Firmware:** v0.3.5 · **Updated:** 2026-07-18 · **Applies to:** Avalon Nano 3s + Avalon Q

REST/JSON API for **every Canaan board running TNA-OS-CANAAN**. This is the integration
contract for building monitoring dashboards, control panels, fleet managers, and automation
against a miner.

**One API, both boards.** The endpoints, request/response shapes, auth and error handling are
**identical** across the supported boards — a client written against one works against the
other unchanged. This is deliberate: both boards run the same firmware from the same source
tree, and the board is a compile-time identity, not a different API.

| Board | ASICs | Power |
|---|---|---|
| **Avalon Nano 3s** | 12× A3197S | USB-C PD |
| **Avalon Q** | 160× A3197S | Mains PSU (series-string rail) |

Where hardware genuinely differs, the **fields** differ — not the endpoints. Those cases are
called out per-board in the tables below (chiefly **Power & voltage**, and the size of
`boards[].chips`). Fields that don't apply to the connected board read `0` / `false`.
**Detect the board from `minerModel`** in `GET /api/system/info`; do not infer it from chip
count or from which fields are non-zero.

- **Base URL:** `http://<miner-ip>` (default HTTP port **80**)
- **Content type:** `application/json` (request bodies and all responses)
- **Firmware:** the running version is in `GET /api/system/info` → `version`
- **Web UI:** the same server also serves the Angular dashboard at `/`

> **Conventions in this doc:** `<miner-ip>` is your miner's LAN address (e.g. `192.168.1.32`).
> Fields most apps need are documented inline; rarely-needed internal/diagnostic
> fields are collected in **Appendix A** to keep the main reference clean.

---

## 0. Security model

**The HTTP API is intentionally unauthenticated** — no token, no login — so any app on
the LAN can monitor and control the miner (3rd-party dashboards, fleet tools, automation).
Treat the API as fully open to everyone on the network: anyone who can reach the miner can
read telemetry **and** change settings (pools, frequency, voltage, Wi‑Fi) and restart/reboot it.

---

## 1. Getting started

```bash
# Read everything (the primary telemetry endpoint)
curl http://<miner-ip>/api/system/info

# Set frequency + voltage
curl -X PATCH http://<miner-ip>/api/system \
  -H "Content-Type: application/json" \
  -d '{"frequency": 300, "coreVoltage": 3650}'
```

```python
import requests
BASE = "http://192.168.1.xx"

info = requests.get(f"{BASE}/api/system/info").json()
print(f"{info['hashRate']/1000:.2f} TH/s | {info['temp']}°C | {info['power']:.0f} W")

requests.patch(f"{BASE}/api/system", json={"frequency": 300, "coreVoltage": 3650})
```

### 1.1 Authentication
**None.** All endpoints are open. The miner is intended to run on a **trusted LAN**.
There is no HTTPS and no token. Do not expose it directly to the internet — put it
behind your own firewall/reverse-proxy if remote access is needed.

### 1.2 CORS
Fully open — every response includes:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET,POST,PATCH,OPTIONS
Access-Control-Allow-Headers: *
```
Browser apps can call the API directly. `OPTIONS` preflight returns `204 No Content`.

### 1.3 Status codes & errors
Every **`/api/*`** request returns **HTTP 200** — there are no 4xx/5xx codes on the API;
check the JSON body instead. (The static file server is separate: it returns `404` for a
missing file and `403` for a path-traversal attempt.)
- Success: the requested data, or `{"ok": true}` for writes.
- Failure: `{"ok": false, "err": "<reason>"}` (e.g. bad JSON, validation error).
- Unknown `/api/*` path: `{"ok": true}` (treated as a no-op).

Always treat a write as **fire-and-forget** and re-read `GET /api/system/info` to
confirm the effect.

### 1.4 Polling rate
Telemetry refreshes **once per second**. Polling faster than 1 Hz just returns the
same snapshot. **1–5 s** is the sensible range for dashboards.

### 1.5 Unit conventions
| Quantity | On the wire | Convert for display |
|---|---|---|
| Hashrate | **GH/s** (number) | `/ 1000` → TH/s |
| Voltage (Vcore) | **millivolts** (integer) | `/ 1000` → V |
| Temperature | **°C** (number) | — |
| Frequency | **MHz** (number) | — |
| Power | **Watts** (number) | — |
| Network rate | **kbps** (number) | — |
| Network total | **MB** (number) | since daemon start |
| Difficulty | share difficulty (number) | — |

---

## 2. Monitoring endpoints (read)

### 2.1 `GET /api/system/info`
The primary snapshot — everything the dashboard shows, refreshed every second.

```bash
curl http://<miner-ip>/api/system/info
```

Returns one large JSON object. The fields apps typically use:

#### Identity
| Field | Type | Example | Notes |
|---|---|---|---|
| `version` | string | `"0.3.5"` | TNA-OS firmware version |
| `ASICModel` | string | `"A3197S"` | Always A3197S on the Nano 3s |
| `deviceModel` | string | `"Avalon Nano 3s"` | |
| `asicCount` | int | `12` | Chips on the single board |
| `smallCoreCount` | int | `1440` | Total cores (120 × 12) |
| `eepromSerial` | string | | Board serial from EEPROM |
| `hostname` | string | `"nano3s"` | |
| `hostip` | string | `"192.168.1.32"` | Active interface IP |
| `macAddr` | string | `"40:a5:ef:12:34:56"` | Device MAC |
| `uptimeSeconds` | int | | Miner daemon uptime |
| `systemUptimeSecs` | int | | Since kernel boot |

#### Hashrate & shares
| Field | Type | Notes |
|---|---|---|
| `hashRate` | number | Current, **GH/s** (instant/short-window) |
| `hashRate_1m` | number | 1-minute average |
| `hashRate_10m` | number | 10-minute average |
| `hashRate_1h` | number | 1-hour average |
| `hashRate_1d` | number | 1-day average |
| `hashEfficiency` | number | GH/s per Watt |
| `sharesAccepted` | int | Total accepted (all pools) |
| `sharesRejected` | int | Total rejected |
| `sharesSubmitted` | int | Total submitted |
| `bestDiff` | number | Best share difficulty seen |
| `bestSessionDiff` | number | Best this session |
| `poolDifficulty` | number | Current pool difficulty |

> **Hashrate is difficulty-based.** Prefer `hashRate_10m` for a stable headline number;
> `hashRate`/`hashRate_1m` are noisier, especially in multipool.

#### Power & voltage

The two boards are powered completely differently, so this is the one part of the API
that is genuinely board-specific. **Nano 3s** = USB-C PD (a powered brick). **Avalon Q** = mains
PSU driving a series-string rail. Fields not applicable to a board read `0` / `false`. Identify the
board from `minerModel`.

**Common**
| Field | Type | Unit | Notes |
|---|---|---|---|
| `power` | number | W | Board power. 3s: measured input. Q: PSU measured output. |
| `maxPower` | number | W | 3s ≈ 134. Q ≈ 4000. |
| `powerValid` | bool | | Readings are meaningful. `false` → treat `power`/`current`/`hashEfficiency` as invalid. |
| `hashEfficiency` | number | GH/W | Hashrate per watt. |

**Nano 3s only (USB-C PD)**
| Field | Type | Unit | Notes |
|---|---|---|---|
| `current` | number | A | Input current |
| `voltage` | int | mV | Input bus voltage × 1000 |
| `vcoreV` / `psuVoltageV` | number | V | Negotiated PD voltage |
| `coreVoltage` | int | mV | ASIC Vcore **setpoint** |
| `coreVoltageActual` | int | mV | **Live Vcore read back from the DC/DC** — what the regulator is actually programmed to, not an echo of `coreVoltage`. Compare the two to see the rail tracking the setpoint (they differ while a change is ramping in). |
| `defaultCoreVoltage` | int | mV | Firmware default |
| `minVoltage` / `maxVoltage` | int | mV | 3100 / 3900 |
| `psuPdActive` | bool | | USB-C PD contract negotiated |
| `psuBypass` | bool | | PSU-bypass mode (external DC injection instead of USB-C PD) |
| `bypassVoltage` | number | V | Injected voltage when `psuBypass` (26–30, default 28) |

**Avalon Q only (mains PSU string rail)**
| Field | Type | Unit | Notes |
|---|---|---|---|
| `psuVoutV` | number | V | String-rail voltage **setpoint**. Writable — see `stringVoltage`. |
| `psuVoutActualV` | number | V | String-rail voltage **measured** |
| `psuIoutA` | number | A | String-rail current **measured** |
| `psuPoutW` | number | W | Output power **measured**. Same value as `power`. |

> **The Q PSU reports OUTPUT ONLY — there is no mains/input telemetry.** So there is no `Vin`,
> `Iin`, `Pin`, or PSU conversion-efficiency figure exposed by the API. Mining efficiency is
> reported instead as **J/TH** (`power / (hashRate/1000)`), the industry convention — **lower is
> better**.
>
> **Do not use these fields — they do not exist:** `psuVinV`, `psuIinA`, `psuPinW`, `psuEfficiency`.
> They were removed; older clients should ignore them.

#### Temperature
| Field | Type | Unit | Notes |
|---|---|---|---|
| `temp` | number | °C | Board NTC probe (the headline temp) |
| `vrTemp` / `boardtemp1` / `boardtemp2` | number | °C | Same probe (aliases) |
| `boardProbeTemp` | number | °C | Board NTC (raw) |
| `overheat_temp` | number | °C | Active danger/shutdown threshold |

> The headline `temp` is the board NTC probe; per-chip die temps are in
> `boards[].chipTemps`. There is no separate SoC/CPU die-temp field.

#### Fans
| Field | Type | Notes |
|---|---|---|
| `fanspeed` | int | Duty cycle 0–100 % |
| `manualFanSpeed` / `fanDuty` | int | Aliases of `fanspeed` |
| `fanRpm` | int | **Live tach RPM** (e.g. `960`). Nano 3s: the single chassis fan. Q: the fastest of `fanRpms`. |
| `fanRpms` | int[] | **Avalon Q only** — the four individual fan RPMs (front pair + back pair). On the Nano 3s this is empty/zero (single fan → read `fanRpm`). |
| `autofanspeed` | int | `0` = manual, `1` = thermal-auto |

> In thermal-auto (`autofanspeed:1`) fans are driven by a PID controller —
> see the `pid*` fields in **3.1** to tune it.

#### Mining integrity
| Field | Type | Notes |
|---|---|---|
| `shitcoinDetected` | bool | `true` when a connected pool serves **non-Bitcoin** SHA-256 work — detected from the job's nbits network difficulty (< 100 T). The firmware then throttles to 100 MHz and flashes the OLED + dashboard. Detected **per-pool**, so a low-quota decoy BTC pool can't mask a shitcoin pool. |

#### Network
| Field | Type | Notes |
|---|---|---|
| `ethRxKbps` / `ethTxKbps` | number | Live throughput (kbps) on the active interface |
| `ethRxTotalMB` / `ethTxTotalMB` | number | Cumulative since daemon start (MB) |
| `ethIPv4` | string | Active IP |
| `ethMac` | string | Interface MAC address |
| `networkMode` | string | Active network: `"ethernet"` \| `"wifi"` \| `"ap"` |
| `ethAvailable` | int | `1` = Ethernet interface present |
| `ethLinkUp` | int | `1` = Ethernet PHY link up |
| `ethConnected` | int | `1` = Ethernet has connectivity |
| `ssid` | string | WiFi SSID (empty in AP/setup mode) |
| `wifiStatus` | string | `"connected"` or `"ap"` |
| `wifiSignalPct` | int | **WiFi signal strength, 0–100.** The field to render as a bar. `0` = no adapter / not associated / Ethernet. |
| `wifiLinkQualityPct` | int | WiFi link quality, 0–100. Same source as `wifiSignalPct`. |
| `wifiRSSI` | int | Signal in **dBm** — but **only from adapters that actually report dBm**, else `0`. See the note below; prefer `wifiSignalPct`. |
| `wifiPass` | string | Always `""` — a compatibility stub; the stored WiFi password is **never** returned by the API |
| `apMode` | bool | `true` = captive setup AP active (no WiFi configured) |
| `apSsid` | string | Setup-AP SSID (`"TNA-Setup"`) while `apMode`, else empty |
| `apIp` | string | Setup-AP IP (`"192.168.4.1"`) while `apMode`, else empty |

> **Use `wifiSignalPct`, not `wifiRSSI`.** WiFi adapters don't agree on units: some report true
> dBm, others a relative 0–100 quality. The firmware detects which and normalises, so
> `wifiSignalPct` is always a 0–100 bar on **any** supported adapter. `wifiRSSI` is only
> populated when the adapter genuinely reports dBm — the Realtek parts these boards ship with
> (built-in RTL8723DU, and the RTL8811CU / RTL8812BU dongles) report relative quality, so they
> leave `wifiRSSI` at `0` rather than publish a raw quality number as a nonsensical positive dBm.

#### Frequency
| Field | Type | Unit | Notes |
|---|---|---|---|
| `frequency` | int | MHz | Current PLL frequency (ramps toward target) |
| `defaultFrequency` | int | MHz | Configured target |
| `chipFrequencies` | int[] | MHz | Per-chip (length = `asicCount`) |

#### Pools — `stratum` object
Supports **up to 8 pools**, **V1 (stratum+tcp) and V2 (Stratum V2 / `sv2://`) mixed**,
with quota-weighted work distribution.

```json
"stratum": {
  "poolMode": 1,
  "activePoolMode": 1,
  "totalBestDiff": 31560000.0,
  "pools": [
    {
      "connected": true,
      "status": "alive",
      "protocol": "v1",
      "url": "stratum+tcp://pool.gobrrr.me:3333",
      "host": "stratum+tcp://pool.gobrrr.me:3333",
      "port": 3333,
      "user": "bc1q...worker",
      "pass": "x",
      "quota": 25,
      "effectiveQuota": 25,
      "poolDifficulty": 1000.0,
      "accepted": 342,
      "rejected": 0,
      "submitted": 342,
      "bestDiff": 125000.0
    }
  ]
}
```

| Pool field | Type | Notes |
|---|---|---|
| `poolMode` | int | `0` = failover, `1` = multipool (quota) |
| `pools[].protocol` | string | `"v1"` or `"v2"` |
| `pools[].status` | string | `"alive"` \| `"disconnected"` \| `"dead"` |
| `pools[].quota` | int | Configured weight |
| `pools[].effectiveQuota` | int | Live weight (redistributed when a pool dies) |
| `pools[].accepted` / `rejected` / `submitted` | int | Per-pool share counters |
| `pools[].poolDifficulty` | number | That pool's current difficulty |

> Pool index in this array is **0-based**; the UI labels pools 1-based (Pool 1 = index 0).

> **Legacy top-level mirror (cgminer-compat).** Alongside the `stratum` object, the info
> response also emits flat fields mirroring the *active* pool, kept for older cgminer-style
> clients — **prefer `stratum.pools[]`**: `stratumURL`, `stratumPort`, `stratumUser`,
> `stratumDifficulty`, `stratumEnonceSubscribe`. A single legacy fallback pool is exposed as
> `fallbackStratumURL` / `fallbackStratumPort` (default `3333`) / `fallbackStratumUser` /
> `fallbackStratumEnonceSubscribe`, with `usingFallback` (bool). These are superseded by the
> multipool `pools[]` array and normally empty/unused.

#### Per-board / per-chip — `boards[]`
Always **exactly one entry** (id `0`) on the Nano 3s.

```json
"boards": [
  {
    "id": 0,
    "state": "active",
    "runtimeState": "active",
    "chips": 12,
    "frequency": 300.0,
    "hashrate": 3500.0,
    "temp1": 54.0,
    "temp2": 54.0,
    "thermalZone": "normal",
    "enabled": true,
    "faultReason": null,
    "targetFreqMhz": 300.0,
    "requestedVoltageMv": 3650,
    "voltageActual": 3648,
    "chipNonces30s": [12,11,13,12,11,12,13,12,11,12,11,13],
    "chipNonceWindowSecs": 30,
    "chipTemps": [54.5,55.1,54.8,55.3,54.9,55.0,55.2,54.7,55.1,54.8,55.0,54.9],
    "chipTempWindowSecs": 5
  }
]
```

| Board field | Type | Notes |
|---|---|---|
| `state` | string | `empty` \| `initializing` \| `active` \| `failed` |
| `runtimeState` | string | `empty` \| `initializing` \| `active` \| `failed` \| `overheatShutdown` \| `recoveryPending` \| `userDisabled` |
| `faultReason` | string\|null | e.g. `OVERTEMP_SHUTDOWN`, `ASIC_INIT_FAILURE`, `WATCHDOG_NONCE_STALL` |
| `chips` | int | Chips responding (12 when healthy) |
| `frequency` / `hashrate` | number | MHz / GH/s for this board |
| `thermalZone` | string | `normal` \| `warn` \| `hot` \| `danger` |
| `chipNonces30s` | int[] | ⚠️ Misnamed — currently **cumulative accepted shares attributed per chip since session start**, NOT a 30 s window (its sum equals `sharesAccepted`). Still usable as *relative* chip health; do not rate it against time. Will be renamed/fixed in a future build. |
| `chipTemps` | number[] | Per-chip die temps (°C) |

#### History — `history` object
Pre-bucketed series for charting (also available standalone at `/api/history/data`).
| Field | Notes |
|---|---|
| `hashrate_1m` / `hashrate_10m` / `hashrate_1h` / `hashrate_1d` | GH/s series |
| `timestamps*` | ms offsets from `timestampBase` |
| `timestampBase` | epoch ms anchor |
| `temp_10m` / `fan_10m` | matching temp/fan series |

Other top-level objects: `led` (RGB state), `solar` (solar-mining telemetry),
`immersionConfig`, `thermalThresholds`, `army`. See **Appendix A** for full field lists.

---

### 2.2 `GET /api/system/asic`
Static model capabilities + the tuning envelope (use to build a tuning UI).

```bash
curl http://<miner-ip>/api/system/asic
```
```json
{
  "ASICModel": "A3197S",
  "deviceModel": "Avalon Nano 3s",
  "asicCount": 12,
  "chipsPerBoard": 12,
  "minFrequency": 100,
  "maxFrequency": 600,
  "defaultFrequency": 300,
  "absMaxFrequency": 600,
  "frequencyOptions": [100,150,200,250,300,350,400,450,500,550,600],
  "minVoltage": 3100,
  "maxVoltage": 3900,
  "defaultVoltage": 3200,
  "voltageOptions": [3200,3250,3314,3350,3400,3450,3500,3550,3600,3650,3700,3750,3800,3850,3900],
  "userPresets": [ { "name": "Efficient", "frequency": 300, "voltage": 3650 } ]
}
```
`frequencyOptions` / `voltageOptions` are the suggested UI steps; any value within
`min…max` is accepted by `PATCH /api/system`. Manage `userPresets` via the presets endpoints.

### 2.3 `GET /api/system/logs`
Last 100 lines of the daemon log.
```json
{ "logs": "[2026-06-06T20:08:27Z INFO ...] ...\n..." }
```

### 2.4 `GET /api/history/len` and `GET /api/history/data`
```json
{ "len": 742 }
```
`/api/history/data` returns the same `history` object as `/api/system/info`, standalone
for chart clients that don't want the full snapshot.

---

## 3. Control endpoints (write)

### 3.1 `PATCH /api/system`
The main control endpoint. Send only the fields you want to change. Returns
`{"ok": true}`. Changes apply immediately and persist to the config file
(survive reboot).

```bash
curl -X PATCH http://<miner-ip>/api/system \
  -H "Content-Type: application/json" \
  -d '{"frequency": 300, "coreVoltage": 3650, "fanspeed": 80}'
```

#### Writable fields
| Field | Type | Unit | Effect |
|---|---|---|---|
| `frequency` | number | MHz | Target PLL freq. Clamped **100–600** (BDOC: min 50). Triggers a managed ramp. |
| `perChipFreq` | number[] | MHz | Per-chip frequency targets, index = chip (up to `asicCount` entries). `0` = follow the global `frequency`. Non-zero clamped **100–600** (BDOC: min 50); the chip ramps to its own target while the rest of the chain holds. |
| `autoPowerOn` | bool | | Energise the hashboard automatically at boot. Default: 3s `true`, Q `false`. Reboot required (see 3.7). |
| `coreVoltage` | number | mV | ASIC Vcore. Clamped **3100–3900** (BDOC: unclamped). |
| `fanspeed` | int | % | 0–100, immediate. Sets manual fan duty. |
| `autofanspeed` | int | | `1` → thermal-auto (fans managed by temp). |
| `poolMode` | int | | `0` = failover, `1` = multipool. |
| `pools` | array | | Full pool list — **see 3.2**. |
| `hostname` | string | | Device hostname. |
| `displayBrightness` | int | | OLED brightness 0–255. |
| `led` | object | | `{ "r","g","b","brightness","override" }` (0–255). |
| `bdocMode` | bool | | Removes V/F clamps; single reset threshold (`bdocOverheatTemp`). |
| `bdocOverheatTemp` | number | °C | BDOC reset threshold (50–120). |
| `immersionMode` | bool | | Raises thermal thresholds; remaps fans to pump+radiator. |
| `immersionConfig` | object | | `{ pumpChannel, pumpSpeed, radiatorChannel, tempOffset }`. |
| `overheat_temp` | number | °C | Normal-mode danger threshold. |
| `ignoreTempSensorFault` | bool | | Keep mining if the NTC probe faults (use with care). |
| `solar` | object | | Solar integration config (see Appendix A). |
| `pidTargetTemp` | number | °C | Fan-PID target temp (thermal-auto mode). Clamped **40–90**. |
| `pidP` / `pidI` / `pidD` | number | | Fan-PID gains (thermal-auto mode). Defaults `12` / `0.4` / `0`. |
| `psuBypass` | bool | | Enable PSU-bypass mode (external DC injection vs USB-C PD). |
| `bypassVoltage` | number | V | Injected voltage for bypass mode. Clamped **26–30**. |
| `oledRotation` | int | | OLED orientation in degrees (`0`/`90`/`180`/`270`). |

> **Verify writes** by re-reading `GET /api/system/info` — `PATCH` always returns `{"ok":true}`.

### 3.2 Configuring pools (V1 + V2)
Send the **full** pool list in one `PATCH`. Up to 8 pools; mix V1 and V2 freely.

```bash
curl -X PATCH http://<miner-ip>/api/system \
  -H "Content-Type: application/json" \
  -d '{
    "poolMode": 1,
    "pools": [
      { "url": "pool.gobrrr.me",     "port": 3333, "user": "bc1q...w1", "pass": "x", "quota": 50, "protocol": "v1" },
      { "url": "stratum.braiins.com","port": 3333, "user": "acct.worker","pass": "x", "quota": 25, "protocol": "v1" },
      { "url": "stratum.braiins.com","port": 3333, "user": "",           "pass": "x", "quota": 25, "protocol": "v2",
        "sv2Key": "<pool-sv2-authority-key>" }
    ]
  }'
```

| Pool field | Type | Notes |
|---|---|---|
| `url` | string | Host **without** scheme (the firmware adds it). |
| `port` | int | Pool port. |
| `user` | string | Worker/username (pool-specific). |
| `pass` | string | Password (use `"x"` if the pool ignores it). |
| `quota` | int | Relative weight in multipool (`poolMode:1`). |
| `protocol` | string | `"v1"` (stratum+tcp) or `"v2"` (Stratum V2). |
| `sv2Key` | string | **V2 only** — the pool's SV2 authority public key (get it from your pool's docs). Stored as `sv2://host:port/<key>`. |

Notes:
- `poolMode: 0` (failover) mines the first alive pool, falling back on disconnect.
- `poolMode: 1` (multipool) splits work by `quota`; if a pool dies its quota is
  redistributed to the survivors.
- A worker that the pool doesn't recognise yields rejects (e.g. `SNotAuthorized`) —
  that's a pool-account/worker-name issue, not a miner bug.

### 3.3 `POST /api/presets` and `DELETE /api/presets/:name`
Named frequency/voltage pairs (shown in `/api/system/asic` → `userPresets`).
```bash
curl -X POST http://<miner-ip>/api/presets \
  -H "Content-Type: application/json" \
  -d '{"name":"Efficient","frequency":300,"voltage":3650}'

curl -X DELETE "http://<miner-ip>/api/presets/Efficient"
```
Returns `{"ok":true}` (or `{"ok":false,"err":"not found"}` on delete miss).

### 3.4 `PATCH /api/thermal/thresholds`
Tune the thermal ladder.
```bash
curl -X PATCH http://<miner-ip>/api/thermal/thresholds \
  -H "Content-Type: application/json" \
  -d '{
    "normal":    { "warn": 80, "hot": 85, "danger": 90 },
    "immersion": { "warn": 85, "hot": 87, "danger": 90, "offset": 15 },
    "bdoc":      { "bdocOverheat": 95 },
    "rebootCooldown": 50
  }'
```
All sub-objects optional. `bdocOverheat` clamps 50–120 °C, `rebootCooldown` 20–80 °C.

### 3.5 `POST /api/board/0/{action}`
Per-board control. Board id is **0** (the only board).
```bash
# Re-enumerate the board (200 ms reset pulse, re-inits all 12 chips)
curl -X POST http://<miner-ip>/api/board/0/reset

# Disable (hold in reset) / enable (release)
curl -X POST http://<miner-ip>/api/board/0/enable \
  -H "Content-Type: application/json" -d '{"enabled": false}'

# Per-board frequency override
curl -X POST http://<miner-ip>/api/board/0/frequency \
  -H "Content-Type: application/json" -d '{"override": true, "mhz": 300}'
```

### 3.6 `POST /api/system/power` — hashboard ON / OFF (no reboot)
Energise or de-energise the hashboard while the control board, UI, network and telemetry stay up.
This is the idle-gate power control — the miner boots idle on the Q (energise deliberately) and
straight into hashing on the 3s (`auto_power_on` default: 3s ON, Q OFF).
```bash
curl -X POST http://<miner-ip>/api/system/power -H "Content-Type: application/json" -d '{"on": true}'   # Power ON
curl -X POST http://<miner-ip>/api/system/power -H "Content-Type: application/json" -d '{"on": false}'  # Power OFF
```
- **Power ON** — energise the hashboard and start mining (~15 s). Voltage and frequency both
  restart from their safe floor and ramp back up to your configured values, exactly as they do on
  a cold boot — so expect `frequency` / `coreVoltageActual` to climb for a few seconds before
  hashrate returns to its pre-OFF level. Pools connect with a **fresh session** at Power ON
  (subscribe, difficulty, job — like a cold boot).
- **Power OFF** — stop mining and de-energise the hashboard. On the Q the rail is cut immediately
  (a safety action, independent of the mining loop). **All pool connections are closed** and stay
  closed while the board is off — expect every `stratum.pools[].connected` to read `false` until
  the next Power ON. The control board, UI, network and telemetry stay fully online — only the
  hashboard and its pool traffic go off.

### 3.7 `POST /api/system/restart` and `POST /api/system/reboot` — full control-board reset
Both endpoints do the **same** thing: a **full reboot of the control board** (a complete reset, not
just the mining process), ~60–90 s downtime.

```bash
curl -X POST http://<miner-ip>/api/system/reboot
```
Response: `{"ok":true,"msg":"rebooting (full control-board reset)"}`.

A reboot is **required** for boot-time settings such as `autoPowerOn` to take effect.

> `POST /api/system/power-cycle` exists but returns `{"ok":false,...}` and is not supported — use
> `power` (OFF/ON) for the hashboard rail, or `reboot` for the whole control board.

### 3.8 `POST /api/system/ping`
Ping a host from the miner (handy for testing pool reachability).
```bash
curl -X POST http://<miner-ip>/api/system/ping \
  -H "Content-Type: application/json" -d '{"target":"pool.gobrrr.me"}'
```
```json
{ "success": true, "rtt": 24.8 }
```
Accepts `target` or `host`; stratum/sv2 URL prefixes are stripped automatically.
`{"success": false, "rtt": 0.0}` if unreachable.

### 3.9 `POST /api/wifi/configure`
Set WiFi credentials (used from the captive setup AP when `apMode: true`).
```bash
curl -X POST http://<miner-ip>/api/wifi/configure \
  -H "Content-Type: application/json" -d '{"ssid":"MyNet","password":"MyPassword"}'
```
Validates `ssid` non-empty and `password` ≥ 8 chars, then saves and reboots.
Returns `{"ok":true,"msg":"credentials saved, rebooting"}` or `{"ok":false,"err":"..."}`.

> **Setup flow:** a freshly-flashed/un-provisioned miner broadcasts AP `TNA-Setup` at
> `192.168.4.1`. Connect, open the UI (or POST here), and it reboots onto your network.

---

## 4. cgminer-compatible endpoint

`POST /api` accepts a subset of the classic cgminer JSON API so existing
miner-monitoring tools work with minimal changes.

```bash
curl -X POST http://<miner-ip>/api -H "Content-Type: application/json" \
  -d '{"command":"summary"}'

# multiple commands with '+'
curl -X POST http://<miner-ip>/api -d '{"command":"summary+pools+devs"}'
```

Supported commands: **`version`**, **`summary`**, **`devs`**, **`pools`**, **`stats`**.
Unknown commands return a generic `STATUS: S` OK envelope. Responses follow the
cgminer `{ "STATUS":[...], "<SECTION>":[...], "id":1 }` shape, e.g.:
```json
{ "summary": [ { "STATUS":[{"STATUS":"S","Code":11,"Msg":"Summary"}],
  "SUMMARY":[{"Elapsed":459,"GHS 5s":3500.0,"GHS av":3500.0,"Accepted":172,
              "Rejected":0,"Hardware Errors":0,"Temperature":54.0}], "id":1 } ] }
```
> `MHS 5s` in `devs` is GH/s × 1000. `GHS *` fields are already GH/s.

---

## 5. Stub endpoints
Present for UI/tooling compatibility; features not active on this hardware:
| Endpoint | Response |
|---|---|
| `GET /api/army/status` | `{ "mode":0, "enabled":false, "port":2121, "connectedSoldiers":0, "totalSoldierHashrate":0.0 }` |
| `GET /api/army/soldiers` | `[]` |
| `GET /api/stratum_proxy/status` | `{ "running": false }` |
| `GET /api/stratum_proxy/clients` | `[]` |
| `GET /api/alert/info` | `{ "enabled": false }` |
| `GET /api/influx/info` | `{ "enabled": false }` |
| `GET /api/otp/status` | `{ "enabled": false }` |

---

## 6. Integration recipes

### Home Assistant
```yaml
sensor:
  - platform: rest
    name: "Nano3s Hashrate"
    resource: http://192.168.1.32/api/system/info
    value_template: "{{ (value_json.hashRate_10m / 1000) | round(2) }}"
    unit_of_measurement: "TH/s"
    scan_interval: 10
  - platform: rest
    name: "Nano3s Temp"
    resource: http://192.168.1.32/api/system/info
    value_template: "{{ value_json.temp }}"
    unit_of_measurement: "°C"
    scan_interval: 10
```

### Poll + control (Python)
```python
import requests
BASE = "http://192.168.1.32"

s = requests.get(f"{BASE}/api/system/info").json()
print(f"{s['hashRate_10m']/1000:.2f} TH/s  {s['power']:.0f}W  {s['temp']}°C  "
      f"A:{s['sharesAccepted']} R:{s['sharesRejected']}")
for p in s["stratum"]["pools"]:
    print(f"  {p['protocol']} {p['url']} q{p['quota']} {p['status']} "
          f"A:{p['accepted']} R:{p['rejected']}")

# apply a preset
requests.patch(f"{BASE}/api/system", json={"frequency": 300, "coreVoltage": 3650})
```

### curl cookbook
```bash
HR=http://MINER/api/system/info
curl -s $HR | jq '.hashRate_10m/1000'                 # TH/s
curl -s $HR | jq '.boards[0].chipTemps'               # per-chip temps
curl -s $HR | jq '.stratum.pools[] | {url,protocol,status,accepted,rejected}'
curl -s $HR | jq '{th:(.hashRate_10m/1000), w:.power, c:.temp, rpm:.fanRpm}'
```

---

## 7. Gotchas (read this)
- **Single board, 12 chips.** `boards[]` always has one entry (id 0); `asicCount` is 12.
- **Voltage is millivolts.** `coreVoltage: 3650` = 3.65 V. Range 3100–3900 (normal mode).
- **Hashrate is GH/s.** Divide by 1000 for TH/s. Use `hashRate_10m` for a stable figure.
- **Multipool hashrate is noisy short-term.** Per-pool share counts settle to the quota
  ratio over minutes, not seconds.
- **Pools array is 0-indexed; the UI shows 1-based.** Pool 1 in the UI = `pools[0]`.
- **Stratum V2 pools** need `protocol:"v2"` + `sv2Key`. SV2 accepts are reported by the
  pool in **batches**, so `accepted` ticks up in steps, not per share.
- **No SoC die-temp field.** Use `temp` (board NTC) and `boards[0].chipTemps` (per-chip).
- **Writes are fire-and-forget** → always re-GET to confirm.
- **`restart` == `reboot`** here (both full reboot). There is no daemon-only restart and
  no `power-cycle` (USB-C PD hardware).
- **AP/setup mode:** if `apMode:true`, configure WiFi via `/api/wifi/configure` first.

---

## Appendix A — Advanced / diagnostic fields

These appear in `GET /api/system/info` but most apps can ignore them. Listed for
power users and deep tooling.

**PSU / PD internals:** `psuVoltageV`, `psuCurrentA`, `psuWatts`, `psuFault`,
`psuFaultCode`, `psuStatus0`, `psuStatus1`, `minPower`, `maxPower`, `vcoreV`, `vcoreMv`.

**Control board:** `cpuLoad1m`, `cpuLoad5m`, `cpuLoad15m`, `cpuFreqMhz`,
`ramTotalMb`, `ramUsedMb`, `freeHeap`, `freeHeapInt`.

**Diagnostic temps:** `iTemp` (intake), `oTemp` (outlet, `-273` when stuck),
`boardProbeStuck`, `tMax`, `tAvg`.

**Per-board diagnostics (`boards[]`):** `nonces30s`, `rxBytes30s`, `regResponses30s`,
`dropped30s`, `connector`, `sensorFault`, `freqOverride`, `freqOverrideMhz`,
`voltageActual`, `attempt`.

**LED (`led`):** `r`, `g`, `b`, `brightness`, `override` (all 0–255 / bool).
Writable via `PATCH /api/system` `{"led":{...}}`. With `override:true` the bar shows the fixed
`r/g/b` colour. With `override:false` it shows the thermal-zone colour (green/amber/orange/red) —
and `brightness` **still** applies, so you can keep the thermal colours but dim them. The DANGER
(red) state is always full brightness regardless of `brightness` — it's a safety signal.

**Solar (`solar`):** `enabled`, `pvPower`, `batterySoc`, `batteryV`, `acLoad`,
`chargerState`, `source`. Config via `PATCH /api/system` `{"solar":{...}}`
(`protocol`, `modbusHost`, `modbusPort`, `vedirectDevice`).

**Display / UI:** `oledEnabled` (bool), `oledBrightness` (= `displayBrightness`),
`defaultTheme` (`"dark"`), `swarmColor` (UI accent, `"orange"`), `invertfanpolarity`
(`0`/`1`, fan PWM polarity).

**Network diagnostics (last `/api/system/ping`):** `pingRtt`, `pingLoss`, `recentpingloss`,
`lastpingrtt` (all numbers; `0` until a ping has run).

**History timestamp series:** `timestamps_1m`, `timestamps_1h`, `timestamps_1d` — ms-offset
arrays paired with the matching `hashRate_*` series (anchor `timestampBase`).

**Aliases / fixed-value stubs (Nano 3s):** `activeBoards` (=1), `minerModel`
(= `deviceModel`), `vrFrequency` (`0`), `hashRateTimestamp` (`0`), `proxyDifficulty`
(`10000`), `stratum_keep` (`0`), `poolDiffErr` (bool), `chain_acn1` (cgminer `stats` chip
count).

**Other:** `led*` booleans (`ledRed`, `ledGreen`), `buttonIp`, `tachWarmingUp`,
`jobInterval`, `lastResetReason`, `flipscreen`/`invertscreen`/`autoscreenoff` (display),
`army`, `immersionConfig`, `thermalThresholds`. (Fan-PID fields `pidTargetTemp`/`pidP`/
`pidI`/`pidD` and `oledRotation` are active — writable, see **3.1**.)

---

## Appendix B — Thermal behaviour (reference)

| Zone | Trigger (normal mode, default) | Action |
|---|---|---|
| Normal | < warn | — |
| Warn | ≥ 80 °C | Fans → 100 % |
| Hot | ≥ 85 °C | Fans 100 % + step frequency down |
| Danger | ≥ 90 °C | Board held in reset until re-enabled |

Hysteresis applies before exiting a zone. **Immersion mode** raises thresholds and
remaps fans to pump+radiator. **BDOC mode** disables the warn/hot ladder (single
`bdocOverheatTemp` reset) and removes V/F clamps — full manual control. Thresholds are
editable via `PATCH /api/thermal/thresholds`.
