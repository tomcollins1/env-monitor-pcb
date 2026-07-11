# Component Selection Rationale

> **Project:** env-monitor-pcb — Portable Environmental Monitor
> **Author:** Bulakon Collins · **Date:** July 2026 · **Version:** 1.0

This document details the component selection process for the environmental monitor PCB. For each major component, a set of candidates was evaluated against the project's key requirements: I2C communication compatibility, low power consumption, small form factor, cost, and availability. The selected component and the reasoning for rejecting alternatives are documented for each category.

---

## Table of Contents

- [1. Microcontroller](#1-microcontroller)
- [2. CO₂, Temperature & Humidity Sensor](#2-co-temperature--humidity-sensor)
- [3. Barometric Pressure Sensor](#3-barometric-pressure-sensor)
- [4. LiPo Battery Charger IC](#4-lipo-battery-charger-ic)
- [5. Battery](#5-battery)
- [6. USB-C Connector](#6-usb-c-connector)
- [7. Summary](#7-summary)

---

## 1. Microcontroller

### ✅ Selected: ESP32 Module — ~£6

The ESP32 was selected as the central microcontroller because it natively satisfies all four of the project's core processing requirements without the need for external modules:

| Requirement | ESP32 Capability |
|---|---|
| I2C | For communication with the SCD41 and BMP390 sensors |
| SPI | For driving the Waveshare 2.13" e-ink display |
| Wi-Fi (802.11 b/g/n) | For publishing sensor data to a remote dashboard |
| Dual-core processing | To handle sensor acquisition and Wi-Fi communication simultaneously without blocking |

A key advantage for this battery-powered application is the ESP32's deep sleep current of approximately **10µA**. At a 5-minute sampling interval, the microcontroller spends the vast majority of its time in deep sleep, making this low idle current critical to achieving the target battery life of ~38 hours. A microcontroller without a comparable sleep mode would draw 15–20mA continuously, reducing battery life to under 5 hours for the same workload.

The ESP32 is used in the module form factor (ESP32-WROOM-32), meaning the antenna, RF matching network, crystal, and flash memory are all contained within the module's metal shield. This removes the need to design RF-sensitive circuitry, which is a specialist discipline requiring impedance-matched trace routing and RF simulation tools beyond the scope of this project.

The ESP32 is supported by both the Arduino framework and Espressif's native ESP-IDF, with mature, actively maintained libraries available for every component selected in this project. It is also widely deployed in commercial IoT products and industrial sensor nodes, making familiarity with the platform a transferable industry skill.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| Raspberry Pi Pico W (RP2040) | Cheap (~£6), dual-core, Wi-Fi, Python/C++ support | Wi-Fi stack less mature than ESP32; deep sleep less well implemented; thinner library support for SCD41 and e-ink displays |
| STM32F103C8 + ESP8266 Wi-Fi module | Common in industry, bare-metal programming experience, lower unit cost (~£3) | No native Wi-Fi requires a separate ESP8266 module communicating via UART AT commands, adding board complexity, a second firmware stack, and an additional failure point. Wi-Fi operations become multi-step command exchanges rather than direct API calls |
| Nordic nRF52840 | Excellent BLE, ultra-low power consumption | No native Wi-Fi; would require a phone gateway to publish data, adding unnecessary complexity to the data pipeline for this application |

</details>

---

## 2. CO₂, Temperature & Humidity Sensor

### ✅ Selected: Sensirion SCD41 — ~£9

The SCD41 was selected as the primary environmental sensor. It uses **photoacoustic NDIR (Non-Dispersive Infrared)** technology to measure CO₂ concentration — this method detects actual CO₂ molecules via infrared absorption, producing a true CO₂ reading rather than an estimated equivalent derived from VOC (Volatile Organic Compound) proxies, as used in cheaper sensor alternatives. This distinction is critical for a device described as a high-accuracy monitor.

In addition to CO₂ (400–5000 ppm range, matching the project specification), the SCD41 also measures **temperature (-10 to +60°C) and relative humidity (0–100% RH)** in a single package. This eliminates the need for a dedicated temperature and humidity sensor, reducing component count, board space, and BOM cost.

The most significant advantage of the SCD41 over its predecessor is its **single-shot measurement mode**: the sensor takes one measurement on demand and immediately returns to a low-power idle state, drawing only 0.4mA between measurements. This is directly aligned with the 5-minute sampling interval and has a material impact on battery life compared to sensors that must run continuously.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| Sensirion SCD40 | Same NDIR technology, slightly cheaper | Lacks single-shot measurement mode; must run periodic measurement cycle continuously, drawing significantly more average current over a 5-minute interval |
| MH-Z19C | Accurate NDIR measurement, ~£15 | Communicates via UART, not I2C — breaks the shared I2C bus architecture and requires additional GPIO pins; physically larger |
| CCS811 / ENS160 | Very cheap (~£3–5), tiny package | Estimates eCO₂ from VOC sensing rather than measuring CO₂ directly; readings are less accurate and drift over time — not appropriate for a high-accuracy application |
| Sensirion SCD30 | High accuracy NDIR, well established | Larger physical size, higher power consumption, and higher cost than the SCD41 with no meaningful accuracy advantage for this application |

</details>

---

## 3. Barometric Pressure Sensor

### ✅ Selected: Bosch BMP390 — ~£2

The BMP390 is Bosch's current recommended barometric pressure and temperature sensor. It offers ±0.5 hPa absolute accuracy across a range of 300–1250 hPa, fully covering the project's specification of 300–1300 hPa. It supports both I2C and SPI, making it fully compatible with the shared I2C bus, and draws only **3.2µA in low power mode** — negligible relative to the rest of the system.

Since the SCD41 already provides temperature and humidity readings, the BMP390 contributes only the barometric pressure channel, with no redundant data. Its 2.0×2.0mm LGA package is compact enough to fit comfortably on the board without affecting the overall footprint. The sensor is well supported by the Adafruit BMP3XX Arduino library, which is actively maintained and provides a straightforward API.

The BMP390 is the direct successor to the BMP388 and is Bosch's currently recommended part, offering measurably better noise performance (±0.09 Pa RMS vs ±0.12 Pa RMS) and lower power consumption at the same price point.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| BMP388 | Same interface, slightly cheaper, pin-compatible | Predecessor to BMP390; higher noise floor (±0.12 Pa RMS vs ±0.09 Pa RMS), higher power in normal mode, no longer the recommended Bosch part |
| BME280 | Measures pressure, temperature, and humidity in one package | Lower pressure accuracy than BMP390; temperature and humidity channels are redundant given the SCD41 already provides these with higher accuracy |
| BME680 | Adds VOC gas sensing on top of pressure, temp, humidity | VOC channel produces estimated rather than measured air quality values; higher cost and power draw with no meaningful advantage over the SCD41 + BMP390 combination |
| LPS22HB (ST Microelectronics) | Very small footprint, I2C compatible | Lower absolute accuracy than BMP390; thinner library ecosystem; no advantage justifies switching from the Bosch part |

</details>

---

## 4. LiPo Battery Charger IC

### ✅ Selected: Microchip MCP73831 — ~£0.70

The MCP73831 is a linear single-cell LiPo charger IC in a SOT-23-5 package. It implements the full CC/CV (Constant Current / Constant Voltage) charging profile required to safely charge a single-cell LiPo battery. The charge current is set by a single external programming resistor on the PROG pin, calculated as:

$$R_{PROG} = \frac{1000V}{I_{CHARGE}}$$

For a target charge current of 500mA (appropriate for the 2000mAh battery at a 0.25C rate), this gives R_PROG = 2kΩ, resulting in a charge time of approximately 4 hours from empty. Beyond the programming resistor, only input and output decoupling capacitors are required, minimising board space and external component count.

The MCP73831 includes **thermal shutdown protection**, which disconnects the charge current if the IC temperature exceeds a safe threshold, protecting both the IC and the battery during fault conditions. It is well documented with a comprehensive Microchip datasheet and application notes, and is readily available from Mouser, Farnell, and LCSC.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| TP4056 | Most widely used hobbyist LiPo charger, very cheap, comparable functionality | Less comprehensive documentation than MCP73831; slightly inferior thermal performance; no meaningful advantage over MCP73831 for this application |
| BQ24079 (Texas Instruments) | Built-in power path management — device can run from USB input while simultaneously charging the battery, producing more polished product behaviour | Significantly more complex to design with; requires additional external components and careful PCB layout around the switching node; beyond the scope of this project's complexity target, but a strong candidate for a revision 2 design |

</details>

---

## 5. Battery

### ✅ Selected: Adafruit 2000mAh LiPo, 3.7V — ~£8

The 2000mAh single-cell LiPo was selected as the optimal balance between battery life and physical size. Based on the power budget calculation (see [`power-budget.md`](power-budget.md)), the average system current draw at a 5-minute sampling interval with Wi-Fi active is approximately **52mA**, giving an estimated runtime of:

$$\frac{2000\text{mAh}}{52\text{mA}} \approx 38 \text{ hours}$$

The Adafruit cell uses a standard **JST-PH 2mm connector**, which is directly compatible with the MCP73831 reference circuit. At **40×50×9mm**, it fits alongside or beneath the PCB within the target board outline without significantly increasing the device's overall footprint. The Adafruit LiPo cells are well-documented with known discharge characteristics, include built-in protection circuitry, and are a commonly used reference cell for battery life calculations in hobbyist and low-volume commercial designs.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| 1000mAh Adafruit LiPo | Significantly thinner and lighter (~5mm thick), ~19 hours runtime | Halves the battery life; only appropriate if deep sleep mode is implemented to compensate, which adds firmware complexity |
| 3000mAh Adafruit LiPo | ~57 hours runtime, same connector | Noticeably larger and heavier; marginal runtime benefit over 2000mAh does not justify the increased form factor for a portable device |
| 18650 Li-ion Cell | Higher energy density, widely available, robust | Cylindrical form factor requires a mechanical holder rather than a PCB connector, adding height and complicating enclosure design; better suited to a purpose-built enclosure than a flat PCB assembly |

</details>

---

## 6. USB-C Connector

### ✅ Selected: Female USB-C Connector — ~£1

USB-C was selected as the charging interface because it is the current universal standard for device charging, meaning any modern USB-C cable and charger will be compatible with the device. Two **5.1kΩ pull-down resistors on the CC1 and CC2 pins** are used to negotiate a 5V/0.9A power contract from any USB-C source, providing the input voltage required by the MCP73831 charger IC.

USB-C is mechanically reversible, tolerates a high number of insertion cycles, and its use signals a considered, modern design to both end users and employers reviewing the project.

<details>
<summary><b>Alternatives Considered</b></summary>

<br>

| Component | Key Strengths | Reason Not Selected |
|---|---|---|
| Micro-USB | Cheaper, easier to hand solder, still functional | Legacy connector; standardised on USB-C across the industry since 2022; using Micro-USB in a new 2025 design is considered poor practice |
| Barrel Jack (5.5/2.1mm) | Mechanically robust, tolerates higher currents, common in lab equipment | Not appropriate for a portable device; requires a specific wall adapter rather than a universal cable; poor user experience for a consumer-facing product |

</details>

---

## 7. Summary

All components communicate over I2C (sensors) or SPI (display), share a common 3.3V supply rail, and are supported by mature Arduino/ESP-IDF libraries. The component set was selected to minimise external component count, maximise power efficiency at the target 5-minute sampling interval, and remain within a total BOM cost of ~£40.

| Component | Selected Part | Unit Cost |
|---|---|---|
| Microcontroller | ESP32-WROOM-32 Module | ~£6 |
| CO₂, Temp & Humidity Sensor | Sensirion SCD41 | ~£9 |
| Pressure Sensor | Bosch BMP390 | ~£2 |
| Battery Charger IC | Microchip MCP73831 | ~£0.70 |
| Battery | Adafruit 2000mAh LiPo | ~£8 |
| USB-C Connector | Female USB-C | ~£1 |
| Display | Waveshare 2.13" E-ink | ~£5 |
| Passives, connectors, buttons | Various | ~£4 |
| PCB Manufacture (×5) | JLCPCB | ~£4 |
| **Total** | | **~£39.70** |
