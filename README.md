# env-monitor-pcb
A PCB project creating an environmental monitor on a PCB to track temperature, humidity, CO2 levels and barometric pressure.

## Status:
 - In progress... - Schematic design phase

## Goals and brief:
 - The goal of this project is to create a small portable monitor that can assess vital envionmental values in its surrounding area with high accuracy, so that people can use it to determine whether the values in an area make it safe to be in, good for their health, general conviences and interests. The monitor should be small enough to be carried within a backpack without taking much space, so that it is portable and should be able to be placed within a room or area, both outdoors and indoors.

## How does it work?:
 - An ESP32 will act as a central microcontroller with all the sensors connected to 2 GPIO's per component, using I2C communication for clock and data. The ESP32 will then process the data and display values for the user on the E-ink display, which will update every 5 minutes via WiFi. A LiPo battery will keep the device powered, with a long battery life, it will be able to be charged via USB-C, using two 5.1kΩ resistors to negotiate 5V from the charger.

## Block diagram:
 - In progress...

## Target specifications and features:
 - Communication type between components and microcontroller: I2C
 - PPM CO2 Measurements: 400-5000 ppm
 - Temperature measurements: -20 to 65 Celcius
 - Pressure measurements: 300 hPa to 1300 hPa
 - Sampling rate: Measurements taken every 5 minutes (for low power purposes)
 - Screen: 2.13" E-paper display
 - Battery life: 2000mAh 3.7V LiPo battery - Around 38 hours with all components + WiFi transfer active, at a 5-minute sampling rate (USB-C charging)

## Intended components (with cost):
 - Microcontroller: ESP32 - £6
 - Battery: Adafruit 2000mAh LiPo - £4
 - Battery charger: MCP73831 - £0.70
 - USB-C connector: Female USB-C - £1
 - CO2, humidity and temperature sensor: SCD41 - £9
 - Pressure sensor: BMP388 - £6
 - Display: Waveshare 2.13" e-ink display - £5
 - Resistors, capacitors, buttons etc (small components): Estimated below £4
 - Manufacturing: JLCPCB - ~£3-5

## Repository structure:
\`\`\`
/docs - datasheets, design notes, power budget and photos
/hardware - KiCAD schematic and PCB layout files
/gerbers - fabrication outputs
/firmware - ESP32 firmware source
\`\`\`

## Design Documentation
- [Component selection rationale](docs/component-selection.md)
- [Power budget calculation](docs/power-budget.md)
- [Schematic](docs/schematic-v1.pdf)
- [Bring-up log](docs/bring-up-log.md)

## Progress Log
- [x] Project scoping and component selection
- [ ] Schematic design
- [ ] PCB layout
- [ ] Design review
- [ ] Manufacturing order
- [ ] Assembly
- [ ] Firmware development
- [ ] Testing and validation

## Author
Bulakon Collins — BEng Electrical Engineering, University of Nottingham  
[LinkedIn](https://www.linkedin.com/in/bulakon-collins-701033253/) | 
[GitHub]()

