[← Back to the project README](../README.md)

# Cloud-free without ESPHome — Philips / Versuni over local CoAP

Moved out of the root README. This covers running **stock** Philips/Versuni
firmware locally over CoAP — no ESPHome, no hardware changes. For the models
this repo converts to ESPHome instead, see
[`components/philips`](../components/philips/README.md).

Not every Wi-Fi purifier has an ESP32 behind the antenna. **Philips / Versuni** models use an **MXCHIP** Wi-Fi module — ARM Cortex-M silicon (STM32 / Cypress class) running MiCO OS, not Espressif — which typically shows up on the network as hostname `mxchip…` with MAC prefixes such as `B0:F8:93` (Shanghai MXCHIP Information Technology). ESPHome and Tasmota cannot be flashed onto it by any standard means.

The consolation prize: the stock firmware speaks **encrypted CoAP on UDP port 5683 on your LAN** (software signature `AWS_Philips_AIR@…`), so these devices can be driven entirely locally — no cloud, no app — **without opening the case**. That is not "free" in the sense the rest of this repo is: vendor firmware stays on the device, and Philips can change or lock it down with an OTA update. Hence its own section right below the table above, rather than a row in it.

**Integration:** **[ruaan-deysel/ha-philips-airpurifier](https://github.com/ruaan-deysel/ha-philips-airpurifier)** — HACS custom component, auto-discovery via MAC/hostname plus manual IP. A maintained continuation of [kongo09/philips-airpurifier-coap](https://github.com/kongo09/philips-airpurifier-coap), built on [@rgerganov's reverse engineering](https://xakcop.com/post/ctrl-air-purifier/). Requires Home Assistant 2026.4.0+.

> ⚠️ **Caveats, straight from that project:**
> - The connection can work initially and go unresponsive over time — power-cycle the purifier and/or restart HA. Auto-reconnect exists but doesn't always succeed. This is a device-firmware limitation, not an integration bug.
> - **Some newer firmware versions disable local CoAP entirely.** If you're buying a device specifically for this, make sure you can return it.
> - Philips' newer cloud API (Google Home / Alexa) is not available for local integrations.

### Supported models

Extracted from the integration's `FanModel` enum and `device_models.py` — **63 firmware entries / 57 distinct model codes**. The five AC0850 combo variants appear twice because they ship two different firmware personalities (`AWS_Philips_AIR` vs `AWS_Philips_AIR_Combo`).

**Air purifiers** — 47 model codes across 22 series:

| Series | Variants | Class |
|--------|----------|-------|
| AC0650 | AC0650/10 | Compact |
| AC0850 | /11, /20, /31, /41, /70, /81, /85 | Compact |
| AC0950 | AC0950, AC0951 | Compact |
| AC1214 | AC1214 | Compact |
| AC1715 | AC1715 | Compact |
| AC2210 | AC2210, AC2221 | Mid-range |
| AC2729 | AC2729 | Mid-range |
| AC2889 | AC2889 | Mid-range |
| AC2936 | AC2936, AC2939, AC2958, AC2959 | Mid-range |
| AC3033 | AC3033, AC3036, AC3039 | Advanced |
| AC3055 | AC3055, AC3059 | Advanced |
| AC3210 | AC3210, AC3220, AC3221 | Advanced |
| AC3259 | AC3259 | Advanced |
| AC3420 | AC3420, AC3421 | Advanced |
| AC3737 | AC3737 | Advanced |
| AC3829 | AC3829, AC3836 | Advanced |
| AC3854 | AC3854/50, AC3854/51 | Advanced |
| AC3858 | AC3858/50, AC3858/51, AC3858/83, AC3858/86 | Advanced |
| AC4220 | AC4220, AC4221, AC4236 | Premium |
| AC4550 | AC4550, AC4558 | Premium |
| AC5659 | AC5659, AC5660 | Premium |

**2-in-1 purifier + humidifier combos:**

| Series | Variants | Notes |
|--------|----------|-------|
| AC0850 Combo | /11C, /20C, /31C, /41C, /70C | Same hardware as above, `AWS_Philips_AIR_Combo` firmware |
| AMF765 | AMF765 | Oscillation via angle number entity + `fan.oscillate` |
| AMF870 | AMF870 | As above |

**Humidifiers & fans:**

| Model | Type |
|-------|------|
| CX3120, CX3550 | Compact humidifier |
| CX5120 | Advanced humidifier |
| HU1509, HU1510 | Compact humidifier |
| HU4209/00 | Humidifier (in code, not yet in the upstream README table) |
| HU5710 | Premium humidifier |
| CX7550/01 | Oscillating fan — needs the Philips Air app once for Wi-Fi onboarding, then fully local |

`AC2210` / `AC2221` and `HU4209/00` are present in the integration's code but missing from its own README tables — treat them as supported-but-undocumented.

### 🔗 Overlap with this repo

**AC0650** shows up on both sides: the CoAP integration talks to its stock MXCHIP module, while [our Philips / MUJI 600-series component](../components/philips/README.md) replaces that module with an ESP32 and speaks the internal `FE FF` UART protocol instead. If you want an air-quality sensor on an AC0650, the ESP route is the one that gets you there — the stock unit has no AQ sensor at all.

### 🔬 Open question — can the MXCHIP models run ESPHome after all?

> 🔎 **Update — an AC0950 has now been opened.** The module in the Series 900 is an **MXCHIP `EMC6069-P`** (Wi-Fi + BLE, FCC ID [`P53-EMC6069`](https://fccid.io/P53-EMC6069)) — *not* an EMW3080, so the LibreTiny route below does not automatically apply. **The protocol is now decoded and an AC0951 is running ESPHome** via the
> [`philips`](../components/philips/README.md) component — teardown photos, the full
> protocol write-up, captures and a wiring guide: [**devices/philips-900-series**](../devices/philips-900-series).

**Unverified — nobody has opened one of these up.** No MXCHIP→ESP32 swap on a Philips purifier is documented anywhere: the community thread only ever establishes the vendor from MAC prefixes (`b0:f8:93`, `04:78:63`, one report of `e8:c1:d7`), and every Philips project out there is software-only over CoAP. There is no teardown, no UART capture, no replacement attempt to build on.

But the interesting lead isn't adding an ESP32 — it's that **the MXCHIP may be able to run ESPHome itself**:

* The common MXCHIP part, the **EMW3080** (marketed as **MX1290**), is a **relabeled Realtek RTL8710BN** — Ameba-Z, Cortex-M4F.
* `rtl8710b` is a **[LibreTiny](https://docs.libretiny.eu/) target**, and LibreTiny has been [part of ESPHome since 2023.9.0](https://esphome.io/components/libretiny/).
* So on an EMW3080-family module the move is not 🔵 *Add ESP* but a straight **reflash of the existing module** — no extra hardware, no reset pin to hold down.

There is a working precedent on exactly this silicon: **[hn/ginlong-solis](https://github.com/hn/ginlong-solis#replacing-the-main-application)** flashes ESPHome onto the EMW3080-E in a Solis S3 Wi-Fi stick. The stock AliOS-Things image is dumped with **[ltchiptool](https://github.com/libretiny-eu/ltchiptool)**, the module is put into UART boot mode by **pulling TX low during boot** (jumper wires, no soldering), and ESPHome is then written over the stock bootloader and app, with OTA updates from then on. That project also found 8 MB of flash where the datasheets claim 2 MB.

**What has to be verified on an actual board first:**

1. **Which MXCHIP part is in there.** This is the whole question, and it needs the marking read off the can. EMW3080 / MX1290 → RTL8710BN → LibreTiny works. **EMC3080** is a Cortex-M33 and **EMW3060 / EMW3162** are STM32 + Broadcom — neither is a LibreTiny target.
2. **How the module attaches** — UART, SPI or SDIO to the purifier's MCU, and whether the MXCHIP runs the CoAP stack itself. ESPHome on the module still has to speak whatever the main MCU expects, so this protocol needs decoding either way.
3. **A logic-analyzer capture of both UART directions** during app interaction, to see whether it's a Levoit/Philips-style framed binary protocol.
4. **A full stock-firmware dump before anything is written** — as in the Solis project, this is the only way back.

**Free reconnaissance:** the AC2889 FCC filing ([2AICSAC2889](https://fccid.io/2AICSAC2889)) keeps its schematics and block diagram under long-term confidentiality, but the **AC5659 filing ([2ANX9-AC5659](https://fccid.io/2ANX9-AC5659/Internal-Photos/internal-Rev1-3693431)) has public internal photos** — same protocol family, and the cheapest way to identify the module without opening anything.

If you have one of these open on the bench, a photo of the Wi-Fi module and a UART dump in [Discussions](https://github.com/tuct/esphome-projects/discussions) would be very welcome — see [Capturing a UART Dump](#capturing-a-uart-dump) below.
