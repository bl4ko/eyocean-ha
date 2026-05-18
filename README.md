# eyocean-ha

ESPHome-based Home Assistant integration for the **EYOCEAN LED Desk Lamp** (ASIN: B0BNDLND3Y) via 433.92 MHz RF control.

The EYOCEAN desk lamp uses a 433.92 MHz ASK/OOK remote with an **EV1527-based** protocol (32-bit fixed codes — extended from standard 24-bit EV1527, no rolling code). This project replaces the physical remote with an ESP32 + CC1101 transceiver, giving full Home Assistant control.

## Hardware

| Component                          | Purpose         |
| ---------------------------------- | --------------- |
| ESP32-S3 DevKit                    | Microcontroller |
| CC1101 433MHz module (SMA antenna) | RF transceiver  |
| Dupont wires                       | Wiring          |
| USB-C cable + power adapter        | Power           |

### Wiring (ESP32-S3)

```
ESP32-S3          CC1101
─────────         ──────
GPIO9  (CLK)  →   SCK
GPIO10 (MOSI) →   MOSI
GPIO11 (MISO) →   MISO
GPIO12 (CS)   →   CSN
GPIO13 (TX)   →   GDO0
GPIO14 (RX)   →   GDO2
3.3V          →   VCC
GND           →   GND
```

> [!CAUTION]
> CC1101 runs on 3.3V only. Do NOT connect to 5V.

## Protocol Details

| Parameter       | Value                                                     |
| --------------- | --------------------------------------------------------- |
| Frequency       | 433.92 MHz                                                |
| Modulation      | ASK/OOK                                                   |
| Protocol        | EV1527-based (32-bit)                                     |
| Frame structure | 16-bit fixed prefix + 8-bit button code + 8-bit XOR check |
| Security        | Fixed code (no rolling)                                   |
| FCC ID          | 2BD2C-ZZ02DY01007                                         |

### Decoded Remote Codes

The EYOCEAN remote uses a **32-bit** frame (not standard 24-bit EV1527). All 32 bits are required — the lamp ignores truncated 24-bit frames.

**Frame structure** (decoded from one remote):

| Byte | Content      | Value (this remote)                    |
| ---- | ------------ | -------------------------------------- |
| 0-1  | Fixed prefix | `0xBAD1` (constant across all buttons) |
| 2    | Button code  | varies per button                      |
| 3    | XOR check    | `byte2 XOR 0x4E`                       |

Byte 3 equals `byte2 XOR 0x4E` for all five buttons — this is verified from the data. Whether the prefix is a per-remote address or a constant shared across all EYOCEAN units is unknown (only one remote was tested). The `0x4E` XOR key may be derived from the prefix or may be independent.

| Button            | Full Code    | Command |
| ----------------- | ------------ | ------- |
| Power On/Off      | `0xBAD1511F` | `0x51`  |
| Brightness        | `0xBAD15B15` | `0x5B`  |
| Color Temperature | `0xBAD15A14` | `0x5A`  |
| Night Light       | `0xBAD1410F` | `0x41`  |
| Reading Mode      | `0xBAD14907` | `0x49`  |

> [!WARNING]
> These codes were captured from a single remote. It is unknown whether all EYOCEAN remotes ship with the same codes or each has a unique address — the lamp has no pairing/learn button, which suggests codes may be shared across units. If the codes above don't work with your lamp, you'll need to capture your own — see [Capturing Your Own Codes](#capturing-your-own-codes). Standard 24-bit EV1527 decoders (Flipper Zero, rtl_433) only show the first 24 bits — you must capture raw data and decode all 32 bits.

## Installation

### 1. Flash ESPHome

```bash
pip install esphome
esphome run esphome/eyocean-rf.yaml
```

### 2. Add to Home Assistant

The device auto-discovers via ESPHome integration. Go to **Settings → Devices & Services → ESPHome** and adopt the device.

### 3. Place Near Lamp

The ESP32+CC1101 needs RF line-of-sight to the lamp's receiver. Place within ~5m for reliable operation.

## State Management

The lamp is **receive-only** — it has no back-channel to report its state.

## Capturing Your Own Codes

If the codes above don't work, you'll need to capture your own. The protocol uses **32-bit frames** — standard EV1527 decoders only show 24 bits, so you must capture raw data. To capture your codes:

1. **Option A — Flipper Zero**: Sub-GHz → Read → press each button → save `.sub` files
2. **Option B — ESP32+CC1101**: Flash with `remote_receiver` + `dump: all`, press buttons, read logs
3. **Option C — RTL-SDR**: Use `rtl_433` in raw mode — standard EV1527 decoding only shows 24 bits

Update the codes in `esphome/eyocean-rf.yaml` with your values.

## How It Works

```
┌───────────────┐    433.92 MHz    ┌───────────────┐
│ ESP32+CC1101  │ ──── RF ────→    │ EYOCEAN Lamp  │
│  (ESPHome)    │                  │  (receiver)   │
└──────┬────────┘                  └───────────────┘
       │ WiFi
       ▼
┌────────────────┐
│ Home Assistant │
│  (dashboard)   │
└────────────────┘
```

The ESP32 connects to your WiFi and exposes entities to Home Assistant. When you toggle a switch in HA, ESPHome transmits the corresponding EV1527 code via the CC1101 at 433.92 MHz — exactly as the original remote would.

## Contributing

Found different codes on your EYOCEAN lamp? Open a PR with your codes and model number — this would help confirm whether all units share the same codes or each remote has a unique address.

## License

MIT
