# nordicflow — Application Modules

> This document describes the software modules and threads that run on the nordicflow watch. The project is in active exploration. Ideas and specifications here are preliminary and will evolve.

## Overview

nordicflow is an ultra-low-energy open-source watch built for **out-of-grid** scenarios. It is based on the **Nordic nRC45L15** tag with an additional **hat** that expands I/O and radio capabilities. The hat provides the physical layer for the modules described below.

## Design Principles

- **Energy efficiency first** — all modules are designed around sub-milliwatt average power budgets.
- **Decoupled & event-driven** — modules communicate through a lightweight message bus (Zephyr IPC). No thread blocks on another.
- **Degrade gracefully** — when energy is scarce, lower-priority modules yield to critical ones (timekeeping, SOS, mesh heartbeat).
- **Off-grid first** — modules assume no cellular or Wi-Fi infrastructure. Everything works peer-to-peer or via mesh.

## Candidate Modules

### 1. Walkie-Talkie Module (WTM)

Push-to-talk voice communication over the hat's radio.

- **Radio** — narrowband VHF/UHF or LoRa-based voice codec
- **Codec** — ultra-low-bitrate vocoder (e.g. MELP, Codec2 at 700–2400 bps)
- **Interaction** — PTT button triggers TX; audio plays through a small speaker or bone-conduction transducer
- **Power budget** — TX burst only; RX idle with wake-on-radio
- **Status** — exploring

### 2. Morse Code Module (MCM)

Auto-keyes Morse code encoder/decoder.

- **Encoder** — convert typed text or canned messages to Morse sidetone or blinking LED
- **Decoder** — listen for incoming Morse (via microphone or radio) and translate to text
- **Auto-repeat** — periodically beacon a configurable message (APRS-style)
- **Low-power trick** — decode can sleep between dits; wake on carrier detect
- **Status** — exploring

### 3. Mesh Module (MESHM)

Meshtastic-compatible Bluetooth / LoRa mesh node.

- **Protocol** — Meshtastic BLE + LoRa PHY (or compatible)
- **Relay** — forward messages for other nodes in range, extending mesh reach
- **Messaging** — send/receive short text, GPS coordinates, or small data payloads
- **Position sharing** — periodic GPS beacon to mesh (if GPS on hat)
- **Power budget** — RX listen interval configurable (1 s – 5 min); TX only on send/relay
- **Status** — exploring

### 4. SOS / Safety Module (SOSM)

Autonomous emergency beacon that runs independently of other modules.

- **Trigger** — long-press dedicated button, or accelerometer fall detection
- **Action** — broadcast highly-repetitive SOS on multiple radios (BLE advertisement, LoRa, Morse sidetone)
- **Critical priority** — preempts other modules and runs from a reserved energy budget
- **Fallback** — keeps a small backup battery or supercap, so SOS works even when main cell is depleted
- **Status** — planned

### 5. Power Manager Module (PMM)

Supervisory module that allocates energy among modules.

- **Input** — battery voltage, solar harvest (if hat includes solar), module energy requests
- **Policy** — configurable profiles (e.g. "expedition" = conserve, "basecamp" = full mesh)
- **Action** — suspend/resume threads, adjust duty cycles, trigger low-power deep-sleep via Zephyr PM subsystem
- **Status** — planned

### 6. Timekeeping Module (TKM)

The core watch function — keep accurate time with minimal draw.

- **RTC** — use the nRC45L15's internal RTC + external 32 kHz crystal
- **Discipline** — occasional GPS or radio time-sync when available
- **Always-on** — lowest-power module; must run even when main MCU is in deep sleep via RTC wakeup
- **Status** — core (always present)

## Inter-Module Communication

Modules communicate via Zephyr IPC primitives (message queues, mailboxes, or `k_msgq`).

- Messages are typed and carry a small payload (≤ 256 bytes).
- Modules (threads) can block on or poll for specific message types.
- The IPC itself has negligible idle current (sub-µA).

## Energy Budget (Preliminary)

| State      | Current (typical) | Note                |
| ---------- | ----------------- | ------------------- |
| Deep sleep | ~1 µA             | nRC45L15 lowest     |
| RTC only   | ~3 µA             | TKM active          |
| Mesh RX    | ~5–15 mA          | MESHM listening     |
| Mesh TX    | ~20–120 mA        | depends on TX power |
| Audio RX   | ~10–30 mA         | WTM idle + mic bias |
| Audio TX   | ~50–200 mA        | WTM + PA            |
| SOS beacon | ~10–50 mA         | SOSM active         |

Final numbers will depend on hat radio choice and duty-cycling strategy.

## Exploratory Ideas (Not Yet Modelled)

- **Weather station** — read environmental sensors (barometer, temp, humidity) from hat and broadcast over mesh.
- **GPS breadcrumb trail** — log position at configurable intervals; share over mesh or BLE.
- **E-ink field display** — external e-ink panel (on hat) showing map, messages, or Morse output.

## Related Hardware

- **Base** — Nordic nRC45L15 tag (ultra-low-power BLE/RF)
- **Hat** — add-on board providing:
    - Optional LoRa transceiver (e.g. SX1262)
    - Optional audio amp + mic
    - Optional GPS (e.g. u-blox CAM-M8Q)
    - Optional solar charger (e.g. BQ25570)
    - Breakout pins for future experiments
