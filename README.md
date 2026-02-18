# McClintock Panel × ESP32

Driving a vintage O.B. McClintock Co. bank vault alarm panel with an ESP32 and ESPHome, turning 1940s-era security instruments into live Home Assistant gauges.

![McClintock Panel](docs/panel-photo.jpg)

## What Is This?

The O.B. McClintock Co. of Minneapolis manufactured bank vault protection systems from the 1920s through the 1950s. This panel contains three instruments:

| Gauge | Original Purpose | Coil | Drive Method |
|-------|-----------------|------|-------------|
| **Milliamp Meter** | Monitor protection circuit current | ~4mA full scale | DAC + 820Ω resistor |
| **4-Terminal Gauge** | Sound wave / vibration detection | 3kΩ | DAC direct (no resistor) |
| **5-Terminal Gauge** | Balanced circuit monitoring | 36Ω | DRV8871 H-bridge |

The panel also includes knife switches and a fuse holder — purely decorative in this project but they complete the look.

## Terminal Reference

### 5-Terminal Gauge (36Ω coil)

| Terminal | Function | Connection |
|----------|----------|-----------|
| **+** | Coil positive | → DRV8871 OUT1 |
| **−** | Coil negative | → DRV8871 OUT2 |
| **res** | Coil tap (~770Ω from +) | Not used |
| **m.c.** | Moving contact (alarm switch) | Optional: GPIO input |
| **f.c.** | Fixed contact (alarm switch) | Optional: GND |

Measured resistances: + to − = 36Ω, res to + = 770Ω, res to − = 810Ω. The m.c. and f.c. terminals are open (alarm switch contacts).

### 4-Terminal Gauge (3kΩ coil)

| Terminal | Function | Connection |
|----------|----------|-----------|
| **Coil +** | Coil positive | → GPIO26 (DAC2) |
| **Coil −** | Coil negative | → GND |
| **Fixed contact** | Alarm switch (stationary) | Optional: GND |
| **Moving contact** | Alarm switch (on needle) | Optional: GPIO input |

Measured resistance: Coil + to Coil − = 3kΩ.

## Pin Assignment

| GPIO | Function | Target |
|------|----------|--------|
| GPIO25 (DAC1) | Analog out → 820Ω resistor | Milliamp Meter |
| GPIO26 (DAC2) | Analog out direct | 4-Terminal Gauge (3kΩ) |
| GPIO16 (PWM) | DRV8871 IN1 | 5-Terminal Gauge (direction A) |
| GPIO17 (PWM) | DRV8871 IN2 | 5-Terminal Gauge (direction B) |

Power: 3.3V → DRV8871 VM (or 5V VIN if more drive is needed), GND → common ground.

## Hardware Required

| Qty | Component | Notes |
|-----|-----------|-------|
| 1 | ESP32-WROOM-32 dev board | Must be original ESP32 (has DAC). **Not** S2/S3/C3. |
| 1 | DRV8871 breakout board | Adafruit #3190 or generic. ~$4–8. |
| 1 | 820Ω resistor, ¼W | For milliamp meter current limiting. |
| ~6 | Hookup wires, 22 AWG | Solid core for gauge terminals. |
| 1 | USB cable + 5V adapter | Powers the ESP32. |

**Total cost:** ~$12–15 in parts.

## Wiring

Open `docs/wiring-and-config.html` in a browser for interactive wiring diagrams, or see the summary:

```
ESP32 GPIO25 ──→ 820Ω ──→ Milliamp Meter + ──→ Meter − ──→ GND
ESP32 GPIO26 ──────────→ 4-Terminal Coil + ──→ Coil − ──→ GND
ESP32 GPIO16 ──→ DRV8871 IN1 ──→ OUT1 ──→ 5-Terminal +
ESP32 GPIO17 ──→ DRV8871 IN2 ──→ OUT2 ──→ 5-Terminal −
ESP32 3V3    ──→ DRV8871 VM
ESP32 GND    ──→ DRV8871 GND ──→ Common ground
```

## Setup

### 1. Install ESPHome

Either as a Home Assistant add-on (Settings → Add-ons → ESPHome) or standalone:

```bash
pip install esphome
```

### 2. Configure Secrets

```bash
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your WiFi credentials, API key, etc.
```

### 3. Flash

First flash via USB:

```bash
esphome run mcclintock-panel.yaml
```

Subsequent updates go over-the-air automatically.

### 4. Calibrate

**This is critical — these are 80+ year old instruments with irreplaceable coils.**

1. The config ships with `max_power: 0.10` on the DRV8871 outputs (10% duty cycle cap)
2. In Home Assistant, slowly increase the "5-Terminal Gauge" slider and watch the needle
3. If it barely moves at 100, increase `max_power` to 0.20, re-flash, repeat
4. If it slams to the stop at low values, decrease `max_power`
5. The DAC-driven gauges (milliamp meter + 4-terminal) are inherently current-limited and safe from the start
6. Lock in your final `max_power` values — they act as permanent safety limits

## Home Assistant

After flashing, the device exposes three number entities:

- **Milliamp Gauge** — slider 0–100%
- **4-Terminal Gauge** — slider 0–100%
- **5-Terminal Gauge** — slider -100 to +100 (bidirectional)

### Mapping Real Sensors

See the commented `sensor:` block in `mcclintock-panel.yaml` for examples mapping:

- Home power usage → Milliamp Meter
- Outdoor temperature → 4-Terminal Gauge
- Indoor temp deviation from Ecobee setpoint → 5-Terminal Gauge (centered at setpoint, deflects hot/cold)

### Bonus: Alarm Contacts

The m.c./f.c. terminals on both gauges are physical switch contacts that close when the needle deflects far enough. Wire them to GPIO inputs for a real electromechanical alarm sensor in HA:

```yaml
binary_sensor:
  - platform: gpio
    name: "Vault Alarm Trip"
    pin:
      number: GPIO32
      mode: INPUT_PULLUP
      inverted: true
```

## Background

The O.B. McClintock Co. was a Minneapolis-based manufacturer of bank vault and safe protection systems. Their "Sound Wave Relay" detected vibrations from drilling or cutting on vault walls. The "Balanced Relay" monitored a protective wire grid embedded in vault walls — if a wire was cut or shorted, the circuit went out of balance and triggered the alarm. The milliamp meter let technicians verify the system was operating correctly.

This particular panel was found at an antique store for $50.

## License

MIT
