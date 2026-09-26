[← Back to the project README](../README.md)

# Adding a new device

The end-to-end path for getting an unsupported purifier or humidifier running
ESPHome. Most devices in this repo followed exactly these steps.

You do **not** have to do all of it. Getting as far as a good UART dump is the
hard part — the protocol decoding and component work can be picked up from
there by someone else. Say so in the issue.

> ⚠️ **Unplug the unit before opening it.** Part of the control board on most of
> these devices sits on mains.

---

## 1. Open the unit and look at the PCB

The question to answer first: **is there an ESP32 (or similar Wi-Fi module) that
talks to a separate MCU over UART?**

That two-chip split is what this whole repo relies on. The Wi-Fi module handles
cloud and app; a separate microcontroller runs the fan, sensors and display, and
the two talk over a serial link. If the device is built that way, it can almost
certainly be converted.

What to look for:

- **A Wi-Fi module** — a shielded can or a small daughterboard, often on one
  edge of the board. Note the marking (`ESP32-WROOM`, `ESP32-SOLO-1`,
  `MXCHIP EMC6069`, …). Photograph it.
- **A larger MCU** — usually an unmarked or obscure QFP near the centre.
- **Test pads or a header** between them. Manufacturers nearly always leave
  programming/debug pads, and they are the easiest probe points.

If there is only a single chip doing everything, or no obvious Wi-Fi module,
this approach will not work and it becomes a custom-hardware project instead —
still welcome, just a different shape (see the 🔴 *Custom HW* rows in the README
table).

Record what you find: board photos, chip markings, and any FCC ID on the module.

### Check the PCB before you open anything

Anything sold in the US with a radio in it has an FCC filing, and those filings
usually include **internal photos** — the manufacturer's own pictures of the
bare board. That is often enough to answer the ESP32-plus-MCU question, spot the
Wi-Fi module and see where the test pads are, *before* you buy a device or take
one apart.

Search [fccid.io](https://fccid.io) for the ID printed on the device's rating
label, or browse by manufacturer:

| Grantee | Maker | Covers |
|---------|-------|--------|
| [`2ARBY`](https://fccid.io/2ARBY) | Arovast Corporation | **Levoit** / VeSync — purifiers *and* humidifiers |
| [`P53`](https://fccid.io/P53) | MXCHIP | The Wi-Fi modules Philips uses, e.g. [`P53-EMC6069`](https://fccid.io/P53-EMC6069) |

Known Levoit filings, as examples of the naming:

| Device | FCC ID |
|--------|--------|
| Core 200S | [`2ARBY-CORE-200S`](https://fccid.io/2ARBY-CORE-200S) |
| Core 300S | [`2ARBY-CORE-300S`](https://fccid.io/2ARBY-CORE-300S) |
| Core 600S | [`2ARBY-CORE600S`](https://fccid.io/2ARBY-CORE600S) |
| Sprout | [`2ARBY-B381S`](https://fccid.io/2ARBY-B381S) |
| Dual 200S humidifier | [`2ARBY-DUAL200S`](https://fccid.io/2ARBY-DUAL200S) |
| OasisMist LV450S / LV600S | [`2ARBY-LV450S`](https://fccid.io/2ARBY-LV450S) · [`2ARBY-LV600S`](https://fccid.io/2ARBY-LV600S) |

Note the inconsistent hyphenation — `CORE-300S` but `CORE600S` — so don't guess
an ID from the model name; use the [grantee index](https://fccid.io/2ARBY).

Two caveats. Internal photos are sometimes held under **short-term
confidentiality** and only appear months after the grant date, and schematics
and block diagrams are almost always permanently confidential — the Philips
AC2889 filing is an example. And the photographed revision may not match the
unit in your hands. Treat them as reconnaissance, not ground truth.

## 2. Identify the UART pins and capture dumps

Find the two lines between the Wi-Fi module and the MCU, then record them in
both directions while you drive the device.

- Series resistors clustered at the module's pads are the usual probe points.
- Probe **read-only** first, with the stock module still fitted and running —
  the point is to record its conversation with the MCU.
- Capture *both* directions. One line alone is not enough to reconstruct a
  protocol.

Full method, including baud rate, the Logic 2 setup and the frame decoder:
**[Capturing and decoding a UART dump](./capturing-uart.md)**.

Capture one action per dump where you can, and label them. A file named
`filter_reset.txt` containing exactly one filter reset is worth far more than a
long recording of everything. That is how the Philips 900 series and the Core
200S filter counter were both worked out.

Useful dumps to take:

| Dump | Why |
|------|-----|
| Power-up / handshake | Shows how the link is established |
| Power on, then off | The most basic command pair |
| Each fan speed and mode | Maps the mode enum |
| Every toggle the app offers | One capture each: display, child lock, timer, … |
| Anything device-specific | Filter reset, sensor toggles, lights |

## 3. Decode the protocol and build firmware support

This is where the dumps turn into a component: frame format, checksum,
datapoint map, then the entity wiring.

**This step can be handed off** — open an issue with your dumps and the device
details and it can be picked up. If you want to do it yourself, the existing
components are the reference: [`levoit`](../components/levoit/README.md) and
[`philips`](../components/philips/README.md) both document their protocols, and
the `devices/*/README.md` files show how a decode was written up.

Include with the dumps:

- The **MCU firmware version**, if the app shows one
- A list of everything the device can do — speeds, modes, sensors, lights,
  timers
- What the app displays, and its values at capture time. *"The app shows 100%
  filter life right now"* is what identified the Core 200S filter byte.

## 4. Park the original Wi-Fi module

Once your own ESP can drive the MCU, the stock module has to stop talking, or
both will drive the same line.

The usual method is to **pull the original module's `EN` (enable) pin to GND**,
which holds it in reset while leaving it soldered in place. Nothing is removed
and the change is reversible.

- On the Levoit boards this is the stock ESP32's `EN` pin.
- On the Philips 900 series it is a pad on the MXCHIP module, held low with a
  plain wire — see
  [that wiring guide](../devices/philips-900-series/README.md#parking-the-stock-module)
  for how the pull-up value was measured and why it matters.

Check for an existing pull-up before choosing a resistor: against a 10k pull-up,
a 10k pull-down only reaches half the rail and may not hold reset at all.

Until the module is parked, keep your ESP's TX disconnected and treat the setup
as receive-only.

## 5. Add your own ESP32

Wire the new board in:

| Connection | Notes |
|-----------|-------|
| `5V` / `3V3` and `GND` | Tap the board's own supply if it has a convenient header |
| ESP `RX` ← MCU `TX` | The line carrying status frames |
| ESP `TX` → MCU `RX` | The line the stock module transmitted on |
| Original module `EN` → `GND` | Step 4 |

Watch the logic level — measure before connecting a GPIO directly. And avoid
strapping pins: on an ESP32-C3, `GPIO8`/`GPIO9` are strapping pins and `GPIO9`
is usually the BOOT button, so a line held low there at reset drops the board
into download mode instead of running your firmware.

Then flash a config using the component, and watch the log for the link coming
up.

## 6. Contribute it back

Open a pull request with:

- Board photos and chip markings
- The labelled UART dumps
- A `devices/<model>/` folder: example YAML, wiring notes, and a README
- Component changes, if you made them

See [CONTRIBUTING.md](../CONTRIBUTING.md) for the mechanics, and any existing
`devices/` folder for the shape to follow.

Partial contributions are genuinely useful — photos and dumps alone are enough
to get a device started.
