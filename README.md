# Sensorbox-ESPHome
# Sensorbox V2 — ESPHome Configuration Notes

This repository contains modified ESPHome configurations for the **3D Printer Emission Sensor Array (Sensorbox V2)** by Thomas Sanladerer ([@MadeWithLayers](https://www.printables.com/@MadeWithLayers)).

> **Original project:** [https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2](https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2)
>
> Full credit to Thomas for the outstanding hardware design, PCB layout, sensor selection and original firmware concept. This is an exceptional project and well worth building. Go give it a like on Printables!

---

## Files

| File | Description |
|------|-------------|
| `airquality01.yaml` | Non-LVGL version using ESPHome's native display lambda renderer |
| `airquality01_lvgl.yaml` | LVGL version using arc gauge widgets |

---

## Before You Flash

Before compiling either config, you will need to supply your own credentials. There are two ways to do this:

**Option A — Use a `secrets.yaml` file (recommended)**
Add the following entries to your ESPHome `secrets.yaml` file:

```yaml
wifi_ssid: "your_wifi_ssid"
wifi_password: "your_wifi_password"
api_encryption_key: "your_api_encryption_key"
ota_password: "your_ota_password"
```

Then update the config files to reference them:

```yaml
api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password
```

**Option B — Enter credentials directly in the yaml**
Replace the placeholder values in the config file directly:

```yaml
api:
  encryption:
    key: "your_api_encryption_key_here"

ota:
  - platform: esphome
    password: "your_ota_password_here"
```

A new API encryption key can be generated from the ESPHome dashboard when creating a new device, or via the ESPHome CLI with `esphome generate-api-key`.

> **Note:** Do not commit your actual credentials to a public repository. If using Option B, ensure your yaml files are in `.gitignore` or that credentials have been removed before pushing.

---

## Non-LVGL Version (`airquality01.yaml`)

### Overview

This version uses ESPHome's native drawing API via a `lambda:` block in the `display:` component. All sensor readings are drawn as text with colour-coded values, divided into sections by horizontal lines.

### Changes from Original

#### Display Driver Migration
The original config used `platform: ili9xxx` which has been deprecated in favour of `platform: mipi_spi`. The display section has been updated accordingly:

```yaml
display:
  - platform: mipi_spi      # was: ili9xxx
    model: ILI9341
    data_rate: 40MHz         # explicit SPI rate — default was 10MHz
    transform:
      swap_xy: false         # all three transform options now required
      mirror_x: false
      mirror_y: true
    dimensions: 240x320      # inline format required by mipi_spi
```

**Key differences from `ili9xxx`:**
- All three `transform` options (`swap_xy`, `mirror_x`, `mirror_y`) must be explicitly declared — omitting any one causes a validation error
- `dimensions` must use the inline `WxH` format rather than a nested `height`/`width` block
- `data_rate` defaults to 10MHz in `mipi_spi` vs 40MHz in `ili9xxx` — explicitly setting 40MHz restores previous performance

#### Display Lambda Refactor
The original display lambda was functional but contained repeated code blocks. The refactored version introduces:

- A **`threshold_color` helper lambda** replacing 9 identical 4-line colour cascade blocks (one per sensor). Thresholds are now expressed as a single line per sensor
- A **`draw_section` helper lambda** replacing 3 identical section-drawing blocks (title, unit, value loop). Sections are now drawn with a single function call
- All sensor state variables are read once at the top of the frame rather than inline
- Unused variables (`h`, `col`, `title`, `unit`, `names[]`, `values[]`, `colors[]`, `count`) removed

#### BMP280 QNH Pressure Correction
The BMP280 reports absolute station pressure. A sea-level correction filter has been added for an installation altitude of **120m**:

```yaml
pressure:
  name: "BMP280 Pressure (QNH)"
  filters:
    - lambda: return x / pow(1.0 - (120.0 / 44330.0), 5.255);
```

Adjust `120.0` to match your actual altitude in metres. The corrected value is directly comparable to Bureau of Meteorology QNH readings and weather station output.

#### Logger Level
Changed from `VERY_VERBOSE` to `DEBUG` for normal operation. `VERY_VERBOSE` generates significant log noise and has a minor performance impact on a busy device like this one.

#### SGP30 eCO2 Display
The SGP30 eCO2 estimate has been removed from the display (though it continues to be reported to Home Assistant). The SCD40 provides a real NDIR CO2 measurement — the SGP30 eCO2 is a correlation-based estimate derived from VOC levels and is not reliable as a standalone CO2 reading.

---

## LVGL Version (`airquality01_lvgl.yaml`)

### Overview

This version replaces the display lambda entirely with [LVGL (Light and Versatile Graphics Library)](https://esphome.io/components/lvgl/) widgets. Rather than redrawing the full screen on a timer, LVGL only redraws widgets that have changed — triggered directly by each sensor's `on_value:` event.

The display shows arc/gauge style indicators for each sensor reading, colour-coded using the same thresholds as the non-LVGL version.

### Layout

```
┌─────────────────────────┐
│  24.5°C    51.2%        │  ← temp / humidity / datetime
├─────────────────────────┤
│  PARTICLES  µg/m³       │
│   [PM1]  [PM2.5] [PM10] │  ← arc gauges, auto-centred
├─────────────────────────┤
│  CO2  ppm               │
│      [SCD40] [ENS*]     │  ← ENS160 hidden until connected
├─────────────────────────┤
│  VOC  ppb               │
│  [SGP30] [ZE08] [ENS*]  │  ← ENS160 hidden until connected
└─────────────────────────┘
```

*Hidden until sensor is physically connected*

### Key Features

#### Sensor-Driven Updates
Every sensor widget updates only when its sensor fires an `on_value:` event. No polling, no full redraws. This means:
- Arc value updates
- Label text updates
- Arc colour changes (white → yellow → orange → red → purple based on thresholds)

All happen instantly when new data arrives, independently of each other.

#### Hidden-Until-Connected Widgets
All sensor sections and individual arcs start with `hidden: true`. They reveal themselves automatically on first `on_value:` trigger. This means:

- **Currently disconnected sensors** (ENS160, SGP4x, SHT40) produce no visible empty space
- **Adding a new sensor** requires no config changes — plug it in and it appears on the display
- **Removing a sensor** causes its arc to simply remain hidden on next boot

The sensors currently confirmed connected on this hardware:

| Sensor | Interface | Status |
|--------|-----------|--------|
| AHT20 | I2C main (0x38) | ✅ Connected |
| BMP280 | I2C main (0x77) | ✅ Connected |
| SCD40 | I2C main (0x62) | ✅ Connected |
| SGP30 | I2C main (0x58) | ✅ Connected |
| PMSX003 | UART (GPIO7/8) | ✅ Connected |
| ZE08-CH2O | UART (GPIO9/10) | ✅ Connected |
| SHT40 | I2C main (0x44) | ❌ Not present |
| ENS160 + AHT20 | I2C secondary | ❌ Not present |
| SGP4x | I2C main (0x59) | ❌ Not present |

#### Auto-Centring Arc Layout
Each section uses an LVGL **flex row layout** for the arc container:

```yaml
layout:
  type: flex
  flex_flow: ROW
  flex_align_main: CENTER
  flex_align_cross: CENTER
```

Since LVGL excludes hidden widgets from flex layout calculations, visible arcs are automatically centred regardless of how many are showing. A single arc sits in the middle; two arcs are evenly spaced as a centred pair; three fill the width.

#### Datetime Display
The datetime label is updated via the `time` component's `on_time_sync` and `on_time` triggers rather than being redrawn every frame:

```yaml
time:
  - platform: homeassistant
    id: esptime
    on_time_sync:
      then:
        - lvgl.label.update:
            id: lbl_datetime
            ...
    on_time:
      - seconds: 0
        minutes: /1
        then:
          - lvgl.label.update:
              id: lbl_datetime
              ...
```

#### Display Configuration
As with the non-LVGL version, the display requires `auto_clear_enabled: false` and `update_interval: never` — LVGL manages all rendering directly:

```yaml
display:
  - platform: mipi_spi
    id: main_display
    auto_clear_enabled: false
    update_interval: never
    ...
```

### Colour Thresholds

All sensors use the same colour scale:

| Colour | Meaning |
|--------|---------|
| White | Normal / Good |
| Yellow | Caution |
| Orange | Warning |
| Red | Danger |
| Purple | Severe |

Specific thresholds per sensor match WHO and industry guidelines for indoor air quality.

---

## Hardware Notes

### mipi_spi Transform Requirements
Unlike `ili9xxx`, the `mipi_spi` driver requires **all three** transform options to be explicitly set. Missing any one will cause a validation error at compile time.

### SPI Data Rate
The default data rate for `mipi_spi` is 10MHz. This config explicitly sets `data_rate: 40MHz` which is reliable on the ESP32-S2 with PSRAM and noticeably improves display refresh performance given the volume of content being rendered.

### PSRAM
The ESP32-S2 Saola used in this build has 2MB quad-mode PSRAM. This must be explicitly declared:

```yaml
psram:
  mode: quad
```

Without this, the display buffer allocation will fail silently and produce a blank screen.

---

## Credits

**Original project design, PCB, enclosure and firmware concept:**
Thomas Sanladerer ([@MadeWithLayers](https://www.printables.com/@MadeWithLayers))
[https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2](https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2)

This is a truly excellent project. The PCB design in particular — breaking out all ESP32-S2 GPIO pins, supporting multiple interchangeable sensor configurations and fitting everything into a compact 3D-printed enclosure — makes it one of the most practical and well-executed open hardware air quality monitors available. Highly recommended.
