# eyocean-ha

ESPHome-based Home Assistant integration for the **EYOCEAN LED Desk Lamp** (ASIN: B0BNDLND3Y) via 433.92 MHz RF control.

The EYOCEAN desk lamp uses a 433.92 MHz ASK/OOK remote with the **EV1527** protocol (24-bit fixed codes, no rolling code). This project replaces the physical remote with an ESP32 + CC1101 transceiver, giving full Home Assistant control.

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

> **Important**: CC1101 runs on 3.3V only. Do NOT connect to 5V.

## Protocol Details

| Parameter       | Value                                         |
| --------------- | --------------------------------------------- |
| Frequency       | 433.92 MHz                                    |
| Modulation      | ASK/OOK                                       |
| Protocol        | EV1527-based (32-bit)                         |
| Frame structure | 20-bit address + 4-bit command + 8-bit suffix |
| Security        | Fixed code (no rolling)                       |
| FCC ID          | 2BD2C-ZZ02DY01007                             |

### Decoded Remote Codes

The EYOCEAN remote uses a **32-bit** frame (not standard 24-bit EV1527). The extra 8-bit suffix is required — the lamp ignores 24-bit frames.

| Button            | Full Code (32-bit) | Bytes 0-2  | Suffix |
| ----------------- | ------------------ | ---------- | ------ |
| Power On/Off      | `0xBAD1511F`       | `0xBAD151` | `0x1F` |
| Brightness        | `0xBAD15B15`       | `0xBAD15B` | `0x15` |
| Color Temperature | `0xBAD15A14`       | `0xBAD15A` | `0x14` |
| Night Light       | `0xBAD1410F`       | `0xBAD141` | `0x0F` |
| Reading Mode      | `0xBAD14907`       | `0xBAD149` | `0x07` |

> **Note**: The address portion is unique to each remote. You'll need to capture your own codes if they differ — see [Capturing Your Own Codes](#capturing-your-own-codes). Standard 24-bit EV1527 decoders (Flipper Zero, rtl_433) will only show the first 24 bits — you must capture raw data and decode all 32 bits.

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

Each EYOCEAN remote has a unique address burned into the chip. The protocol uses **32-bit frames** — standard EV1527 decoders only show 24 bits, so you must capture raw data. To capture your codes:

1. **Option A — Flipper Zero**: Sub-GHz → Read → press each button → save `.sub` files
2. **Option B — ESP32+CC1101**: Flash with `remote_receiver` + `dump: all`, press buttons, read logs
3. **Option C — RTL-SDR**: Use `rtl_433` to decode EV1527 frames

Update the codes in `esphome/eyocean-rf.yaml` with your values.

## How It Works

```
┌──────────────┐    433.92 MHz     ┌──────────────┐
│  ESP32+CC1101 │ ──── RF ────→   │  EYOCEAN Lamp │
│  (ESPHome)   │                   │  (receiver)   │
└──────┬───────┘                   └──────────────┘
       │ WiFi
       ▼
┌──────────────┐
│ Home Assistant│
│  (dashboard) │
└──────────────┘
```

The ESP32 connects to your WiFi and exposes entities to Home Assistant. When you toggle a switch in HA, ESPHome transmits the corresponding EV1527 code via the CC1101 at 433.92 MHz — exactly as the original remote would.

## Contributing

Found different button codes on your EYOCEAN remote? Open a PR with your codes and model number. The address varies per remote but data nibbles should be consistent across all EYOCEAN lamps using the same receiver IC.

## License

MIT
