# Power Budget

> **Project:** env-monitor-pcb — Portable Environmental Monitor
> **Author:** Bulakon Collins · **Date:** July 2026 · **Version:** 1.0

This document calculates the average current consumption of the environmental monitor at a 5-minute sampling interval, and derives estimated battery runtime from that figure. All values are sourced from manufacturer datasheets or calculated from first principles. Figures should be updated once firmware is running and actual measurements can be taken.

---

## Table of Contents

- [1. Methodology](#1-methodology)
- [2. System Assumptions](#2-system-assumptions)
- [3. Per-Component Analysis](#3-per-component-analysis)
  - [3.1 ESP32-WROOM-32](#31-esp32-wroom-32)
  - [3.2 Sensirion SCD41](#32-sensirion-scd41)
  - [3.3 Bosch BMP390](#33-bosch-bmp390)
  - [3.4 Waveshare 2.13" E-ink Display](#34-waveshare-213-e-ink-display)
  - [3.5 MCP73831 Battery Charger IC](#35-mcp73831-battery-charger-ic)
  - [3.6 AP2112K LDO Voltage Regulator](#36-ap2112k-ldo-voltage-regulator)
  - [3.7 I2C Pull-up Resistors](#37-i2c-pull-up-resistors)
- [4. Total Current Summary](#4-total-current-summary)
- [5. Runtime Estimates](#5-runtime-estimates)
- [6. Notes and Observations](#6-notes-and-observations)
- [7. Datasheet References](#7-datasheet-references)

---

## 1. Methodology

Most components in this system cycle between an **active state** (sampling, displaying and transmitting data) and a **sleep state** (waiting for the next cycle). The average current draw of each component is calculated using a duty-cycle weighted average:

$$I_{avg} = \frac{(I_{active} \times t_{active}) + (I_{sleep} \times t_{sleep})}{t_{cycle}}$$

Where:
- `I_active` = current draw during active operation (mA) — from datasheet
- `t_active` = duration of active state per cycle (seconds)
- `I_sleep` = current draw during sleep/idle (mA) — from datasheet
- `t_sleep` = duration of sleep state per cycle (seconds)
- `t_cycle` = total cycle duration = `t_active + t_sleep` (seconds)

For components that are always on (LDO regulator, pull-up resistors), the average current equals their fixed draw with no duty cycling.

Battery runtime is then calculated as:

$$Runtime_{hours} = \frac{Capacity_{mAh}}{I_{total\_avg\_mA}}$$

---

## 2. System Assumptions

| Parameter | Value | Basis |
|---|---|---|
| Sampling interval (cycle duration) | 300 seconds (5 minutes) | Project specification |
| Battery capacity | 2000 mAh | Adafruit 2000mAh LiPo |
| Supply voltage | 3.3V (regulated) | AP2112K LDO output |
| ESP32 active window per cycle | ~7 seconds | Estimated: sensor read + display update + Wi-Fi publish |
| SCD41 measurement duration | ~5 seconds | Sensirion SCD4x Datasheet, Section 3 |
| BMP390 forced mode duration | ~2 ms | Bosch BMP390 Datasheet, Section 3.3 |
| E-ink full refresh duration | ~2 seconds | Waveshare 2.13" e-Paper V4 Specification, Section 6.2 |
| I2C pull-up resistance | 4.7kΩ | Standard value for 3.3V I2C bus |
| Wi-Fi active during wake window | Yes — MQTT publish each cycle | Firmware design |

> ⚠️ **Note:** The ESP32 active window of 7 seconds is an estimate. This figure should be measured with a current probe or power monitor (e.g. Nordic PPK2 or Joulescope) once firmware is running, and this document updated accordingly.

---

## 3. Per-Component Analysis

### 3.1 ESP32-WROOM-32

The ESP32 spends most of the cycle in deep sleep, waking once per 5-minute interval to read sensors, update the display, publish over Wi-Fi, and return to sleep.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Deep sleep current (chip, RTC on) | < 5µA | ESP32 Datasheet v5.2, Table 4-2 |
| Active + Wi-Fi TX average | ~80mA | ESP32 Datasheet v5.2, Table 4-2 |

> **Important:** The datasheet deep sleep figure of <5µA describes the bare chip on Espressif's test fixture. On a custom PCB with an LDO regulator, a conservative target of **10µA** is used here. This is achievable on a custom PCB with correct LDO selection. A dev board would draw 5–15mA in deep sleep due to onboard USB-to-serial and indicator LEDs — this budget assumes the custom PCB design, not a dev board.

**Calculation:**

$$I_{avg} = \frac{(80\text{mA} \times 7\text{s}) + (0.01\text{mA} \times 293\text{s})}{300\text{s}}$$

$$I_{avg} = \frac{560 + 2.93}{300} = \frac{562.93}{300} \approx \mathbf{1.88\text{ mA}}$$

---

### 3.2 Sensirion SCD41

The SCD41 is operated in **single-shot measurement mode** (SCD41-only feature). It takes one measurement on demand and returns to idle, drawing only 0.4mA between measurements. This mode is optimised for a 5-minute measurement interval per the Sensirion datasheet.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Single-shot measurement current | ~18mA | SCD4x Datasheet v1.7, Section 3 (Electrical Characteristics) |
| Single-shot measurement duration | ~5 seconds | SCD4x Datasheet v1.7, Section 3 |
| Idle current | ~0.4mA | SCD4x Datasheet v1.7, Section 3 |

**Calculation:**

$$I_{avg} = \frac{(18\text{mA} \times 5\text{s}) + (0.4\text{mA} \times 295\text{s})}{300\text{s}}$$

$$I_{avg} = \frac{90 + 118}{300} = \frac{208}{300} \approx \mathbf{0.69\text{ mA}}$$

---

### 3.3 Bosch BMP390

The BMP390 is operated in **forced mode** — a single pressure measurement is taken on demand, after which the sensor automatically returns to sleep mode. At a 5-minute interval, the active window is only ~2ms, making the BMP390's contribution to average current essentially negligible.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Forced mode current | ~700µA (0.7mA) | BMP390 Datasheet BST-BMP390-DS002, Section 4 |
| Forced mode duration | ~2ms (0.002s) | BMP390 Datasheet BST-BMP390-DS002, Section 3.3 |
| Sleep mode current | ~0.2µA (0.0002mA) | BMP390 Datasheet BST-BMP390-DS002, Section 4 |

> The BMP390 datasheet also states a typical power consumption of **3.2µA at 1Hz** in normal mode. In forced mode at 1/300Hz (once every 5 minutes), the average is far lower.

**Calculation:**

$$I_{avg} = \frac{(0.7\text{mA} \times 0.002\text{s}) + (0.0002\text{mA} \times 299.998\text{s})}{300\text{s}}$$

$$I_{avg} = \frac{0.0014 + 0.06}{300} = \frac{0.0614}{300} \approx \mathbf{0.05\text{ mA}}$$

> A rounded value of **0.05mA** is used here to account for I2C bus activity and measurement settling time. This is a conservative overestimate.

---

### 3.4 Waveshare 2.13" E-ink Display

E-ink is the correct display technology for this application because it **draws power only during a refresh** and holds the image at zero current draw indefinitely. For a device updating every 5 minutes, the display is static for ~298 out of every 300 seconds.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Refresh current | ~20mA (typical) | Waveshare 2.13" e-Paper V4 Specification, Section 6.2 |
| Full refresh duration | ~2 seconds | Waveshare 2.13" e-Paper V4 Specification, Section 8 (Optical Spec: T_update ~3s max) |
| Static current (image held) | 0mA | E-ink bistable display — no power required to maintain image |

> ⚠️ **Note:** Waveshare state that refresh power consumption data is experimental and actual values will have errors depending on driver board and usage. The 20mA figure is a practical average from the specification. Verify against Section 6.2 of the V4 datasheet PDF directly.

**Calculation:**

$$I_{avg} = \frac{(20\text{mA} \times 2\text{s}) + (0\text{mA} \times 298\text{s})}{300\text{s}}$$

$$I_{avg} = \frac{40 + 0}{300} = \frac{40}{300} \approx \mathbf{0.13\text{ mA}}$$

---

### 3.5 MCP73831 Battery Charger IC

When USB is not connected, the MCP73831 enters a low-power shutdown state. Its standby current is negligible.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Shutdown/standby current | ~1µA (0.001mA) | MCP73831 Datasheet DS20001984H, Section 1 (Electrical Characteristics, I_DD) |

**Average current (fixed, no duty cycling):**

$$I_{avg} = \mathbf{0.001\text{ mA}}$$

---

### 3.6 AP2112K LDO Voltage Regulator

The LDO regulator is always active, consuming a quiescent current to maintain its output voltage regardless of the load.

**Datasheet values:**
| Parameter | Value | Source |
|---|---|---|
| Quiescent current | 55µA (0.055mA) | AP2112K Datasheet, Section 6 (Electrical Characteristics) |

**Average current (fixed, no duty cycling):**

$$I_{avg} = \mathbf{0.055\text{ mA}}$$

---

### 3.7 I2C Pull-up Resistors

Two 4.7kΩ pull-up resistors on the SDA and SCL lines keep the I2C bus high between transmissions. This is a **continuous** current draw calculated from Ohm's Law — not a datasheet value.

**Calculation:**

Each resistor draws:

$$I = \frac{V}{R} = \frac{3.3\text{V}}{4700\text{ Ω}} = 0.000702\text{ A} = 0.70\text{ mA}$$

Two resistors:

$$I_{total} = 0.70\text{ mA} \times 2 = \mathbf{1.40\text{ mA}}$$

> **Design note:** The I2C pull-up resistors are the **dominant fixed overhead** in deep sleep mode at 1.40mA continuous. Increasing the pull-up resistance to **10kΩ** would reduce this to 0.66mA (saving 0.74mA), extending deep sleep runtime by approximately 15%. This is worth considering for revision 2 if maximum battery life is a priority. Higher resistance pull-ups are acceptable as long as bus capacitance and cable length are kept short, which they will be on this PCB.

**Average current (fixed, continuous):**

$$I_{avg} = \mathbf{1.40\text{ mA}}$$

---

## 4. Total Current Summary

| Component | Active Current | Active Duration | Sleep/Idle Current | Sleep Duration | **Average Current** |
|---|---|---|---|---|---|
| ESP32-WROOM-32 | 80mA | 7s | 0.01mA | 293s | **1.88mA** |
| Sensirion SCD41 | 18mA | 5s | 0.4mA | 295s | **0.69mA** |
| Bosch BMP390 | 0.7mA | 0.002s | 0.0002mA | 299.998s | **0.05mA** |
| Waveshare 2.13" E-ink | 20mA | 2s | 0mA | 298s | **0.13mA** |
| MCP73831 (standby) | — | continuous | 0.001mA | — | **0.001mA** |
| AP2112K LDO | — | continuous | 0.055mA | — | **0.055mA** |
| I2C pull-ups (×2, 4.7kΩ) | — | continuous | 1.40mA | — | **1.40mA** |
| | | | | **Total** | **4.22mA** |

---

## 5. Runtime Estimates

### Scenario A — Deep Sleep Mode (5-minute sampling interval)

This is the intended operating mode. The ESP32 sleeps between measurements.

$$Runtime = \frac{2000\text{ mAh}}{4.22\text{ mA}} \approx \mathbf{474\text{ hours}} \approx \mathbf{19.7\text{ days}}$$

### Scenario B — All Components Continuously Active (Worst Case)

This scenario assumes no sleep modes — ESP32 always on with Wi-Fi active, SCD41 in continuous measurement mode, display refreshing constantly. This is not the intended mode but is useful as a worst-case reference.

| Component | Continuous Current |
|---|---|
| ESP32 (Wi-Fi active, no sleep) | 80mA |
| SCD41 (continuous measurement) | 18mA |
| BMP390 (normal mode, 1Hz) | 0.7mA |
| E-ink (continuous refresh) | 20mA |
| MCP73831 standby | 0.001mA |
| AP2112K LDO | 0.055mA |
| I2C pull-ups | 1.40mA |
| **Total** | **~120mA** |

$$Runtime = \frac{2000\text{ mAh}}{120\text{ mA}} \approx \mathbf{16.7\text{ hours}}$$

### Runtime Comparison

| Scenario | Total Current | Estimated Runtime | Notes |
|---|---|---|---|
| Deep sleep, 5-min interval | ~4.22mA | **~474 hours (~19.7 days)** | Intended operating mode |
| All on continuously (worst case) | ~120mA | **~16.7 hours** | Reference only |

### Realistic Adjustment

LiPo cells typically deliver **80–90% of rated capacity** in practice due to voltage sag and protection circuit cutoff. Applying this factor:

| Scenario | Optimistic (90%) | Conservative (80%) |
|---|---|---|
| Deep sleep | ~427 hours (~17.8 days) | ~379 hours (~15.8 days) |
| Worst case | ~15 hours | ~13.4 hours |

---

## 6. Notes and Observations

**I2C pull-up resistors are the dominant fixed overhead.** At 1.40mA continuous, they account for 33% of the total average current in deep sleep mode. This is a known trade-off — lower resistance gives better signal integrity at higher bus speeds, but higher resistance reduces current draw. At the low I2C speeds used for sensor polling, 10kΩ pull-ups would work equally well and reduce this contribution to 0.66mA.

**The 7-second active window is an estimate.** The actual value depends on Wi-Fi association time, MQTT handshake time, sensor read time, and display refresh time. Wi-Fi association from cold start is typically 1–3 seconds on a known network. The full wake window is unlikely to exceed 10 seconds in normal operation. Once firmware is written, measure this with a current probe and update the table.

**SCD41 single-shot mode is optimised for 5-minute intervals.** Per the Sensirion datasheet, the ASC (Automatic Self-Calibration) algorithm is optimised for single-shot measurements performed every 5 minutes. Using a different interval will affect calibration algorithm performance and should be accounted for in firmware configuration.

**The BMP390 contribution is negligible.** At 0.05mA average, the pressure sensor has almost no impact on battery life. Its forced mode is well suited to low sampling rate applications like this one.

**Deep sleep is dramatically more efficient than continuous operation.** The 28× difference in runtime between the two scenarios (474h vs 16.7h) demonstrates why sleep mode implementation in firmware is critical. The component selection (SCD41 single-shot, BMP390 forced mode, e-ink display) was specifically chosen to support this architecture.

---

## 7. Datasheet References

| Component | Parameter | Value Used | Document | Section | Link |
|---|---|---|---|---|---|
| ESP32-WROOM-32 | Deep sleep current | <5µA chip; ~10µA on custom PCB | ESP32 Series Datasheet v5.2 | Table 4-2 | [espressif.com](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf) |
| ESP32-WROOM-32 | Active + Wi-Fi TX average | ~80mA | ESP32 Series Datasheet v5.2 | Table 4-2 | [espressif.com](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf) |
| ESP32-WROOM-32 | Sleep modes documentation | — | Espressif IDF Docs | Sleep Modes | [docs.espressif.com](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/sleep_modes.html) |
| Sensirion SCD41 | Single-shot current | ~18mA | SCD4x Datasheet v1.7 | Section 3 | [sensirion.com](https://sensirion.com/media/documents/48C4B7FB/67FE0194/CD_DS_SCD4x_Datasheet_D1.pdf) |
| Sensirion SCD41 | Idle current | ~0.4mA | SCD4x Datasheet v1.7 | Section 3 | [sensirion.com](https://sensirion.com/media/documents/48C4B7FB/67FE0194/CD_DS_SCD4x_Datasheet_D1.pdf) |
| Sensirion SCD41 | Single-shot duration | ~5 seconds | SCD4x Datasheet v1.7 | Section 3 | [sensirion.com](https://sensirion.com/media/documents/48C4B7FB/67FE0194/CD_DS_SCD4x_Datasheet_D1.pdf) |
| Bosch BMP390 | Forced mode current | ~700µA | BMP390 Datasheet BST-BMP390-DS002 | Section 4 | [bosch-sensortec.com](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp390-ds002.pdf) |
| Bosch BMP390 | Sleep mode current | ~0.2µA | BMP390 Datasheet BST-BMP390-DS002 | Section 4 | [bosch-sensortec.com](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp390-ds002.pdf) |
| Bosch BMP390 | Low power consumption spec | 3.2µA @1Hz (normal mode) | BMP390 Datasheet BST-BMP390-DS002 | Section 1 | [bosch-sensortec.com](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp390-ds002.pdf) |
| Waveshare 2.13" e-ink | Refresh current | ~20mA | 2.13inch e-Paper V4 Specification | Section 6.2 | [waveshare.com](https://files.waveshare.com/upload/4/4e/2.13inch_e-Paper_V4_Specification.pdf) |
| Waveshare 2.13" e-ink | Static current | 0mA | 2.13inch e-Paper V4 Specification | Section 6.2 | [waveshare.com](https://files.waveshare.com/upload/4/4e/2.13inch_e-Paper_V4_Specification.pdf) |
| Waveshare 2.13" e-ink | Refresh duration | ~2 seconds | 2.13inch e-Paper V4 Specification | Section 8 | [waveshare.com](https://files.waveshare.com/upload/4/4e/2.13inch_e-Paper_V4_Specification.pdf) |
| Waveshare 2.13" e-ink | Product page & wiki | — | Waveshare Wiki | — | [waveshare.com](https://www.waveshare.com/wiki/2.13inch_e-Paper_HAT_Manual) |
| MCP73831 | Shutdown current (I_DD) | ~1µA | MCP73831 Datasheet DS20001984H | Section 1 | [microchip.com](https://ww1.microchip.com/downloads/en/DeviceDoc/MCP73831-Family-Data-Sheet-DS20001984H.pdf) |
| AP2112K | Quiescent current | 55µA | AP2112K Datasheet | Section 6 | [diodes.com](https://www.diodes.com/assets/Datasheets/AP2112.pdf) |
| I2C pull-up resistors | Current draw | Calculated (Ohm's Law) | I = V/R = 3.3V / 4700Ω × 2 | — | — |

> **Verification note:** The SCD41 and BMP390 electrical characteristics tables render as images in their respective PDFs rather than extractable text. All current values listed above should be verified directly by opening each datasheet PDF and locating the relevant electrical characteristics table. Section numbers are accurate as of the datasheet versions linked above.
