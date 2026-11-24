# Prompt: build your own pulse meter with Claude Code

Copy everything in the box below into [Claude Code](https://claude.com/claude-code), started in an
empty folder. Attach the photos listed at the top. Fill in the `<...>` parts or delete the lines you
don't know: Claude will ask.

The prompt is written so that Claude asks you before anything irreversible (flashing the board,
overwriting firmware) and keeps your WiFi password and keys out of anything it shares.

---

```text
I want to count the LED pulses of my electricity meter with an ESP32 and an LM393
light-sensor module, and get power (W) and energy (kWh) into Home Assistant via ESPHome.
Use https://github.com/mwestermann/esphome-pulse-meter as the starting point: clone it,
read README.md and pulse-meter.yaml, and adapt the configuration to MY meter and hardware.

What I have (photos attached):
- Photo of my meter's front: <attached>  (read the model, imp/kWh constant, display, pulse LED)
- Fuse/breaker for the measured circuit: <e.g. 3 x 25 A, or photo attached>
- ESP32 board: <e.g. ESP32-WROOM-32 DevKit, 38 pins, or photo attached>
- LM393 module DO is wired to GPIO: <26>
- Serial port of the ESP32 on my computer (if known): <e.g. /dev/cu.usbserial-0001 or COM3>
- Does the board currently run other firmware I may want back (e.g. Glow)? <yes/no>

Please:
1. Identify the meter from the photo. Look up its datasheet and find the pulse constant
   (imp/kWh) and, if stated, the minimum LED period or pulse width. Tell me what you found
   and where, and what you could not find.
2. Calculate and explain the substitutions in pulse-meter.yaml:
   pulse_rate, min_period_ms, max_power_w, pulse_filter, pulse_pin, name, friendly_name.
   - max_power_w = physical maximum of the circuit (voltage x fuse current x phases).
   - min_period_ms = datasheet value, or 3,600,000 / (pulse_rate x max_kW) minus ~10 %.
   - pulse_filter must be shorter than the LED on-time (default 10ms).
   Ask me if a value is uncertain instead of guessing silently.
3. Create secrets.yaml from secrets.example.yaml with freshly generated api_encryption_key,
   ota_password and ap_password. Ask me to fill in wifi_ssid and wifi_password myself in an
   editor. Never print secrets in the chat, and make sure secrets.yaml stays git-ignored.
4. Validate with `uvx esphome config` and compile with `uvx esphome compile`.
   Fix any errors.
5. If the board runs firmware I want to keep, offer to make a full flash backup with
   esptool first, stored outside the project folder (it contains WiFi credentials).
6. Do NOT flash the board until I explicitly say yes. Then flash over USB with
   `uvx esphome run` (not `upload` alone, which does not recompile).
7. Read the device log and confirm WiFi, API and sensors. Then guide me through a bench
   test: cover the sensor (no pulses), flash a torch at it 10 times (10 pulses), and
   check "Rejected pulses" stays at 0.
8. After I mount it on the meter: help me adjust the trimpot, then calibrate with the
   set_meter_reading action using the meter's display value. Remind me to do this
   BEFORE adding Energy to the Home Assistant Energy dashboard.
9. Verify: I will send two photos of the display, at least 0.1 kWh apart (better 0.5-1 kWh),
   each with the time to the second. Stream the device log during that period, compare the
   meter's increase with the device's increase, report any constant offset, and offer
   to correct it.

Keep answers short. Explain trade-offs when you make a choice on my behalf.
```

---

## Tips

- **Photo quality matters.** Shoot the meter straight on, without glare, so the `imp/kWh` text and
  the model name are readable.
- **Timestamps.** Uploaded photos usually lose their EXIF time. Read the capture time in your
  phone's photo info and type it next to each photo.
- **No Claude Code?** The prompt also works in other AI coding assistants that can run shell
  commands. You can also follow the manual steps in the [README](README.md).
