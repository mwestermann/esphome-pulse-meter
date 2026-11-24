# esphome-pulse-meter

Count the flashing LED on your electricity meter with an ESP32 and a cheap LM393 light sensor,
and get **power (W)** and **energy (kWh)** in Home Assistant, including the Energy dashboard.

A small, readable ESPHome configuration you own and tune to *your* meter. It is a
do-it-yourself alternative to [Home Assistant Glow](https://github.com/klaasnicolaas/home-assistant-glow).

> **Made with AI, built on Glow.** The configuration, this README and [PROMPT.md](PROMPT.md)
> were written together with [Claude Code](https://claude.com/claude-code) (Anthropic's AI coding
> assistant), with a human testing every step on real hardware. The idea, the hardware approach and
> much of the know-how come from the Home Assistant Glow project by Klaas Schoute and its contributors.
> No Glow code is copied, but this project would not exist without it. Thank you.

---

## Why you might use this instead of Glow

**Glow is the better choice for most people.** It is mature, well documented, has a web UI,
firmware updates, a large community and a maintainer. If Glow works well on your meter, keep it.

Consider this project if Glow gives you trouble on your setup, or you prefer a small config you can
read in five minutes. These issues came up when replacing Glow 4.2.3 on an ESP32 with an
LM393 module. Your setup may differ, and some of them can also be fixed by customising Glow:

| Problem seen | What this config does |
|--------------|-----------------------|
| **Double counting.** Every LED flash was counted ~2× (contact-bounce-like edges ~1.5 ms apart). Glow's default filter only rejects pulses under 13 µs. | `pulse_meter` with a 10 ms filter in `PULSE` mode: a pulse only counts if the signal is stable ≥ 10 ms. |
| **Comparator chatter.** With the trimpot near its threshold the LM393 oscillated at ~200 pulses/s. | Same filter, plus a *hold-off*: pulses closer than the meter's shortest possible interval are rejected and counted in a diagnostic sensor. |
| **Ghost megawatts at boot** from a floating input pin. | Internal pulldown on the pin; power values above the circuit's physical maximum are discarded. |
| **Optical feedback.** The pulse LED on the ESP board blinked on every pulse and the sensor saw it. | No pulse LED at all. |
| **Polluted total.** Noise ended up in Total Energy permanently. | The total only grows by accepted pulses, is stored in flash, and can be set to the meter's own reading. |
| **Loop blocking.** The firmware update check (`http_request`) blocked the main loop for ~1 s. | No update check, no web server. Updates via ESPHome OTA. |

What you give up compared to Glow: the web UI, the ready-made firmware, automatic update
notifications, and a maintained project with support. This is a personal project shared as-is.

---

## How it works

```
meter LED ──light──▶ LM393 photodiode ─DO─▶ ESP32 GPIO26 (pulldown)
                                              │
                                pulse_meter (10 ms PULSE filter)
                                              │
                          hold-off: reject pulses < min_period_ms
                                              │
                 pulses_total (uint64, saved to flash) ──▶ Energy (kWh)
                 time between pulses ──────────────────────▶ Power (W)
                                              │
                                Home Assistant (native API, encrypted)
```

At 1000 imp/kWh, every flash is 1 Wh. Power is calculated from the time between flashes.

---

## What you need

| Part | Notes | Approx. price |
|------|-------|---------------|
| ESP32 dev board (ESP32-WROOM-32, "DevKit") | 30- or 38-pin both work. **Count the pins** before buying accessories. | €5–10 |
| LM393 light-sensor module with **photodiode** | 4 pins: VCC, GND, DO, AO, plus a blue trimpot. Often sold as "LM393 photosensitive sensor module" or "MH-Sensor-Series". | €1–3 |
| 3 female–female jumper wires | Dupont. Longer if the ESP32 sits away from the meter. | €1 |
| **Good USB power supply** (5 V, ≥ 1 A) and a data-capable USB cable | A weak supply makes the ESP32 reboot in a loop and drop off WiFi. This happened during testing. | €5–10 |
| Black electrical tape, Blu-Tack or double-sided foam tape | For mounting and shielding stray light. | – |
| Optional: screw-terminal adapter for ESP32 | **Must match your board's pin count and row spacing.** A 30-pin adapter does not fit a 38-pin board. | €3–5 |

Tools: a computer with USB, and [uv](https://docs.astral.sh/uv/) to run ESPHome (`uvx esphome`).
Alternatively, use the ESPHome add-on in Home Assistant.

### Safety

- **Never open the meter or touch anything behind its cover.** This project only *looks* at the LED.
- Utility meters are sealed and owned by the grid operator. Don't damage seals, and use tape or adhesive that comes off cleanly.
- Don't cover the meter's display or buttons more than necessary.

---

## Assembly

### Wiring

| LM393 module | ESP32 |
|--------------|-------|
| VCC | **3V3** (not 5V/VIN: DO must stay at 3.3 V) |
| GND | GND |
| DO  | GPIO26 |
| AO  | not connected |

GPIO26 is a safe, non-strapping pin. Any other free input GPIO works too: change `pulse_pin`.
On 38-pin boards, 3V3, GND and GPIO26 are on the same row as `5V`/`V5`.
Check the printed labels on your board.

**Write down your wire colours.** Cheap cables don't follow conventions; black can be VCC.

### Mounting

1. Find the meter's **pulse LED**. It is usually red and marked with the constant, e.g. `1000 imp/kWh`.
2. Place the module's photodiode directly over the LED.
3. Shield it from room light and from the ESP32's own power LED, for example with black tape around the sensor.
4. Adjust while the meter is flashing (some load running). Then fix the module with tape or Blu-Tack.
   Don't put hot glue *on* the photodiode, because it scatters light.

### Trimpot adjustment

The module has two LEDs: power, and DO. The DO LED shows what the ESP32 sees.

1. With the meter LED off, turn the trimpot until the DO LED is **off**.
2. Check that the DO LED flashes **once per meter flash**.
3. Turn a bit further *away* from the switching point. The LM393 has no hysteresis, so near
   the threshold it chatters, and the "Rejected pulses" sensor will climb.

---

## Build the firmware

### Option A: let Claude Code build it for you

Open [PROMPT.md](PROMPT.md), copy the prompt into [Claude Code](https://claude.com/claude-code) in
a folder of your choice, and attach:

- **A clear photo of your meter's front.** The constant (`imp/kWh`), model name, display and
  pulse LED must be readable.
- **A photo of the fuse or breaker for the circuit** (e.g. `3 × 25 A`), or just tell it.
- Your ESP32 board (photo or model) and which GPIO you wired DO to.

The prompt makes Claude look up your meter's datasheet, calculate the right limits, fill in
the configuration, compile it, and walk you through flashing and calibration. It is told to ask
before it flashes anything.

### Option B: do it yourself

1. Copy `secrets.example.yaml` to `secrets.yaml` and fill it in. Generate keys with the commands
   in the file. `secrets.yaml` is git-ignored.
2. Edit the `substitutions:` at the top of [pulse-meter.yaml](pulse-meter.yaml):

   | Substitution | How to choose it |
   |--------------|------------------|
   | `pulse_rate` | The `imp/kWh` printed on the meter. |
   | `min_period_ms` | From the meter datasheet ("min. period"), or `3,600,000 / (pulse_rate × max_kW)` minus ~10 %. |
   | `max_power_w` | Physical maximum of the circuit: voltage × fuse current × phases. |
   | `pulse_filter` | `10ms`. Lower to `5ms` only if the device counts *fewer* pulses than the meter. |
   | `pulse_pin` | The GPIO you wired DO to. |

3. Compile and flash over USB the first time:

   ```bash
   uvx esphome run pulse-meter.yaml
   ```

   Later updates go over WiFi (OTA):

   ```bash
   uvx esphome run pulse-meter.yaml --device pulse-meter.local
   ```

   Use `run` (or `compile` before `upload`): **`upload` alone does not recompile**, so changes
   to `secrets.yaml` would not be included.

### Add it to Home Assistant

Home Assistant discovers the device under **Settings → Devices & services**. Enter the
`api_encryption_key` from `secrets.yaml`.

| Entity | Unit | Notes |
|--------|------|-------|
| Power | W | Drops to 0 after 2 min without a pulse. |
| Energy | kWh | `total_increasing`. Use it in the Energy dashboard **after calibration**. |
| Rejected pulses | count | Diagnostic. Should stay at or near 0. |
| WiFi signal, Uptime | | Diagnostic. |

---

## Calibration and verification

### 1. Set the meter reading (before using the Energy dashboard)

Read the kWh value on the meter's display **and note the exact time**. Ideally do it while the meter is
idle, so the display doesn't move. Then in Home Assistant: **Developer tools → Actions**:

```yaml
action: esphome.pulse_meter_set_meter_reading
data:
  kwh: 12345.67
```

The action name follows your device name (`name: pulse-meter` → `esphome.pulse_meter_...`).
The value is written to flash immediately.

Do this **before** adding Energy to the Energy dashboard. Otherwise Home Assistant books the jump
from 0 to your meter reading as consumption. To fix a jump later, use **Developer tools → Statistics**.

### 2. Verify against the meter's display

1. Make sure the meter is under load and the module's DO LED flashes along with the meter LED.
2. Take a photo of the display and note the time to the second. Chat apps and AI uploads often strip
   the photo's EXIF timestamp, so read the time from your phone's photo info.
3. Wait until the display has moved at least **0.1 kWh** (quick check) or **0.5–1 kWh** (precise check).
4. Take a second photo and note its time.
5. Compare the display's increase with the increase in Home Assistant's Energy history over the same
   interval.

How to read the result:

- **The increase matches** (within the display's resolution, often 0.01 kWh): counting works.
- **A constant offset** (Home Assistant always X kWh off) means the calibration was done with a stale reading.
  Call the action again with a fresh reading.
- **Home Assistant counts less**: lower `pulse_filter` to `5ms`, check the sensor alignment.
- **Home Assistant counts more**: light leaks or the trimpot is too close to its threshold. "Rejected pulses" will often be climbing.

If you use Claude Code, send it both photos with their times. It can read the device log and do the
comparison for you.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Device keeps rebooting or vanishes from WiFi | Weak USB power supply or cable. Use a proper ≥ 1 A supply. |
| "Last reset too quick" in the log | Same as above (brown-out resets). |
| WiFi connect fails right after flashing | Firmware was built before `secrets.yaml` was filled in. Use `esphome run`. |
| Power shows 0 W although the meter flashes | Sensor misaligned, or trimpot set so DO never switches. Watch the DO LED. |
| Counts too many, "Rejected pulses" climbing | Trimpot near threshold, or stray light. Shield the sensor. |
| Counts too few | LED on-time shorter than `pulse_filter`. Try `5ms`. |
| Total lower after a power cut | Expected: up to 5 min of pulses since the last flash write can be lost. Recalibrate if needed. |

Logs over WiFi:

```bash
uvx esphome logs pulse-meter.yaml --device pulse-meter.local
```

---

## Limitations

- Counts every flash as consumption. On meters where the same LED also flashes for export
  (e.g. with solar panels), import and export cannot be told apart.
- Accuracy depends on the meter: one flash = 1/`pulse_rate` kWh. Low loads give slow power updates
  (at 1000 imp/kWh and 100 W, one flash every 36 s).
- Up to 5 minutes of pulses can be lost on a power cut (flash-wear trade-off; see `preferences`).
- ESP32 only, as configured. An ESP8266 needs a different `esphome` platform section.

## Credits

- [Home Assistant Glow](https://github.com/klaasnicolaas/home-assistant-glow) by Klaas Schoute and
  contributors: the original project and inspiration.
- [ESPHome](https://esphome.io) and its `pulse_meter` component.
- Written with [Claude Code](https://claude.com/claude-code) and tested on a Carlo Gavazzi EM340
  (1000 imp/kWh) sub-meter.

## License

[MIT](LICENSE)
