# 🧂 ESPHome — Water Softener Salt Level Monitor

Monitors the salt level of a water softener using a Time-of-Flight distance sensor and integrates seamlessly with Home Assistant. Displays the fill level as a percentage and triggers a configurable alarm before the tank runs empty.

---

## Hardware

| Component | Model | Docs |
|---|---|---|
| Microcontroller | M5Stack AtomS3 Lite (ESP32-S3FN8) | [Documentation](https://docs.m5stack.com/en/core/AtomS3%20Lite) |
| Distance Sensor | M5Stack Unit ToF (VL53L0X) | [Documentation](https://docs.m5stack.com/en/unit/TOF) |

### AtomS3 Lite — Specs

- SoC: ESP32-S3FN8, 8 MB Flash
- Integrated Wi-Fi, built-in 3D antenna
- Form factor: 24 × 24 × 9.5 mm
- Sensor connection: HY2.0-4P (PORT.A)
- USB Type-C for flashing & serial communication

### Unit ToF (VL53L0X) — Specs

- Technology: Time-of-Flight (940 nm laser)
- Measurement range: 3 cm – 200 cm (standard), up to 200 cm in long-range mode
- Accuracy: ±3%, resolution 1 mm
- Protocol: I2C, address `0x29`
- Connection: HY2.0-4P (SDA → G2, SCL → G1)

### Wiring

```
AtomS3 Lite PORT.A  →  Unit ToF
───────────────────────────────
GND  (black)        →  GND
5V   (red)          →  5V
G2   (yellow, SDA)  →  SDA
G1   (white, SCL)   →  SCL
```

---

## How It Works

The VL53L0X sensor continuously measures the distance from the sensor's underside to the salt surface. Since the sensor is mounted at the top of the tank, **larger distance = less salt**.

The fill level percentage is calculated from the measured distance:

```
Fill Level (%) = 100 × (Empty Distance − Measured Value) / (Empty Distance − Full Distance)
```

The alarm (`Salt Empty Error`) triggers once the fill level drops below a configurable threshold — well before the tank is physically empty.

---

## Alarm Logic

```
100% ──────────────────────────── FULL (Full Distance, e.g. 60 mm)
      │
      │   Normal operation
      │
 30% ──── RESET THRESHOLD (Threshold + Hysteresis)
      │        ↑ Alarm clears here (after refilling)
      │
 20% ──── ALARM THRESHOLD (Salt Warning Threshold)
      │        ↓ Salt Empty Error → PROBLEM
      │
  0% ──────────────────────────── EMPTY (Empty Distance, e.g. 530 mm)
```

### States

| Condition | Action |
|---|---|
| No alarm active **AND** fill level ≤ Threshold | `Salt Empty Error` → **Problem** |
| Alarm active **AND** fill level ≥ Threshold + Hysteresis | `Salt Empty Error` → **OK** |

The hysteresis prevents flapping: the reset point always sits above the alarm threshold — regardless of the configured values.

---

## Notable Defaults

- **WPA3** is enforced for Wi-Fi authentication (`min_auth_mode: WPA3`). If your router does not support WPA3, change this to `WPA2` in the YAML.
- **Bluetooth Proxy** is enabled by default, turning the device into a BT signal repeater for Home Assistant. If not needed, comment out the `esp32_ble_tracker` and `bluetooth_proxy` sections in the YAML.

---

## Configuration in Home Assistant

All parameters are configurable directly in HA as `number` entities:

| Entity | Unit | Default | Description |
|---|---|---|---|
| `Full Distance` | mm | 60 | Distance when tank is full (sensor close to salt) |
| `Empty Distance` | mm | 530 | Distance when tank is empty |
| `Salt Warning Threshold` | % | 20 | Fill level at which the alarm triggers |
| `Salt Warning Reset Hysteresis` | % | 10 | Additional buffer before alarm clears |

**Calibration:** Use the `Calibrate Full` and `Calibrate Empty` buttons to store the current sensor reading as the respective reference value.

> **Note on empty calibration:** The VL53L0X does not measure accurately directly off a water surface. It is recommended to set the `Calibrate Empty` reference point at the level where the salt just barely protrudes above the water — not at the bare water surface.

### Entities Overview

| Entity | Type | Description |
|---|---|---|
| `VL53L0x Distance` | Sensor (mm) | Raw distance from ToF sensor |
| `Fill Level` | Sensor (%) | Calculated fill level |
| `Salt Empty Error` | Binary Sensor | Problem alarm |
| `ESP32 Temperature` | Sensor (°C) | Internal chip temperature |
| `WiFi Signal` | Sensor (dBm) | Wi-Fi signal strength |
| `Uptime` | Sensor (s) | Time since last restart |

---

## Installation

### Prerequisites

- [ESPHome Add-on](https://esphome.io/guides/getting_started_hassio) installed in Home Assistant
- `secrets.yaml` with the entries listed below

### secrets.yaml

Create a `secrets.yaml` file in the same directory as the YAML config and fill in your own values:

```yaml
# Example — replace all values with your own

wifi_ssid: "MyHomeNetwork"
wifi_password: "MyWifiPassword123"
fallback_password: "FallbackAP-Password456"
ota_password: "MyOTAPassword789"
esp32-watersoftener_api_key: "aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890abcd="
```

> The API key is generated automatically by ESPHome when adding a new device (Base64 string, 32 bytes). Never commit `secrets.yaml` to version control.

### Flashing

The recommended way is via the **ESPHome Add-on in Home Assistant**:

- **First flash:** Connect the AtomS3 Lite via USB to the Home Assistant host, then use the ESPHome dashboard to flash.
- **All subsequent updates:** Flashed wirelessly over Wi-Fi (OTA) — no USB cable needed.

> **AtomS3 Lite download mode (USB only):** Hold the reset button for ~2 seconds until the internal green LED lights up, then release. The device is now ready for flashing.

CLI flashing is also possible if you have ESPHome installed locally:

```bash
esphome run esp32-watersoftener.yaml
```

---

## Documentation & Links

- [ESPHome VL53L0X Component](https://esphome.io/components/sensor/vl53l0x.html)
- [ESPHome Template Sensor](https://esphome.io/components/sensor/template.html)
- [ESPHome Bluetooth Proxy](https://esphome.io/components/bluetooth_proxy.html)
- [ESPHome Add-on for Home Assistant](https://esphome.io/guides/getting_started_hassio)
- [M5Stack AtomS3 Lite Docs](https://docs.m5stack.com/en/core/AtomS3%20Lite)
- [M5Stack Unit ToF Docs](https://docs.m5stack.com/en/unit/TOF)
- [VL53L0X Datasheet](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/docs/datasheet/hat/VL53L0X_en.pdf)
- [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/)

---

## License

MIT
