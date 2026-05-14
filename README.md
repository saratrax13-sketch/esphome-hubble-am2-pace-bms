# ESPHome Hubble AM2 PACE BMS Monitor

ESPHome Hubble AM2 PACE BMS Monitor is an ESP32 RS485/Modbus configuration for monitoring Hubble AM2 battery packs in Home Assistant.

It is built for Hubble AM2 batteries that use a PACE-based BMS and exposes detailed battery information through ESPHome, MQTT, and Home Assistant.

The project supports a Master/Slave two-pack setup, automatic valid 13-cell detection, per-pack diagnostics, combined battery-bank calculations, projected runtime, projected charging time, and fault visibility without hiding stale or failed data.

---

## Suggested GitHub Repository

Recommended repository name:

```text
esphome-hubble-am2-pace-bms
```

Recommended GitHub description:

```text
ESPHome monitor for Hubble AM2 PACE RS485/Modbus batteries with Master/Slave support, auto cell detection, MQTT/Home Assistant integration, and combined battery-bank sensors.
```

---

## What This Project Does

This project allows one ESP32 to monitor two Hubble AM2 battery packs over RS485:

```text
Pack 1 / Master = Modbus address 0x01
Pack 2 / Slave  = Modbus address 0x02
```

The ESP32 reads data from both packs and publishes the values to Home Assistant.

It provides:

```text
Master battery voltage, current, power, SOC and SOH
Slave battery voltage, current, power, SOC and SOH
Master and Slave remaining/full/design capacity
Master and Slave cell voltages
Master and Slave cell balancing status
Master and Slave warnings, protections, faults and status
Master and Slave temperatures
Combined battery-bank totals
Combined cell-health values
Projected runtime while discharging
Projected charging time while charging
```

The project is monitoring-only. It does not write settings to the BMS.

---

## Key Features

- ESP32 + RS485 monitoring
- Hubble AM2 PACE BMS support
- Master/Slave battery support
- One ESP32 reads both batteries on a shared RS485 bus
- Automatic valid cell-count detection
- Correctly detects Hubble AM2 as 13 cells per pack
- Reads up to 16 possible cells but ignores invalid/unused cells
- Per-pack cell voltage monitoring
- Per-pack cell balancing monitoring
- Per-pack warning, protection, fault and status decoding
- Combined battery-bank calculations
- Projected runtime in minutes
- Projected charging time in minutes
- MQTT/Home Assistant support
- Timeout handling to avoid stale data being shown as live data
- Clear comments inside the ESPHome YAML for maintenance and troubleshooting

---

## Hardware Required

- ESP32 development board
- RS485-to-TTL board/module
- Hubble AM2 battery pack
- Second Hubble AM2 pack if using Master/Slave
- RJ45 breakout cable or suitable battery communication cable
- Twisted pair cable for RS485 A/B where possible
- Common ground wire if required by your RS485 board/battery setup

---

## ESP32 to RS485 Board Wiring

The ESP32 connects to the RS485 board using UART TX/RX and a flow-control pin.

Example wiring used in this project:

| ESP32 Pin | RS485 Board Pin | Purpose |
|---|---|---|
| GPIO16 | RX / RO | ESP32 receives data from RS485 board |
| GPIO17 | TX / DI | ESP32 sends data to RS485 board |
| GPIO18 | DE / RE | RS485 transmit/receive direction control |
| GND | GND | Common ground |
| 3.3V or 5V | VCC | Power for RS485 board, depending on board requirements |

> Important: Check your RS485 board voltage requirements. Some RS485 modules are 3.3V compatible, while others expect 5V. Do not power a 3.3V-only board from 5V unless it is rated for it.

Typical ESPHome UART configuration:

```yaml
uart:
  - id: uart_0
    tx_pin: GPIO17
    rx_pin: GPIO16
    baud_rate: 9600
    stop_bits: 1
    parity: NONE
```

Typical Modbus configuration:

```yaml
modbus:
  - id: modbus0
    uart_id: uart_0
    flow_control_pin: GPIO18
    send_wait_time: 500ms
```

---

## RS485 Board to Battery Wiring

The RS485 side of the RS485 board connects to the Hubble AM2 battery RS485 communication port.

Typical RS485 signal naming:

| RS485 Board | Battery/BMS |
|---|---|
| A / D+ / 485+ | A / D+ / 485+ |
| B / D- / 485- | B / D- / 485- |
| GND / COM | GND / COM, if required |

Different manufacturers sometimes label A/B differently. If the BMS does not communicate, and all other settings are correct, try swapping A and B.

---

## Master/Slave RS485 Bus Wiring

For one ESP32 to read both the Hubble AM2 Master and Slave battery, both batteries must be on the same RS485 bus.

Use a daisy-chain/bus layout:

```text
ESP32 RS485 Board
   A / D+  ───── Master A / D+  ───── Slave A / D+
   B / D-  ───── Master B / D-  ───── Slave B / D-
   GND     ───── Master GND     ───── Slave GND, if required
```

Visual layout:

```text
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ ESP32 RS485  │       │ Hubble AM2   │       │ Hubble AM2   │
│ Board        │       │ Master       │       │ Slave        │
│              │       │              │       │              │
│ A / D+  ─────┼───────┤ A / D+  ─────┼───────┤ A / D+       │
│ B / D-  ─────┼───────┤ B / D-  ─────┼───────┤ B / D-       │
│ GND     ─────┼───────┤ GND     ─────┼───────┤ GND          │
└──────────────┘       └──────────────┘       └──────────────┘
```

Avoid a star layout:

```text
                 ┌──── Master
ESP32 RS485 ─────┤
                 └──── Slave
```

RS485 works best as a bus/daisy-chain.

---

## Modbus Addressing

The project assumes:

```text
Hubble AM2 Master = 0x01
Hubble AM2 Slave  = 0x02
```

Example ESPHome Modbus controller section:

```yaml
modbus_controller:
  # Hubble AM2 Master
  - id: bms_master
    address: 0x01
    modbus_id: modbus0
    setup_priority: -10
    command_throttle: 500ms
    update_interval: 30s
    offline_skip_updates: 5

  # Hubble AM2 Slave
  - id: bms_slave
    address: 0x02
    modbus_id: modbus0
    setup_priority: -10
    command_throttle: 750ms
    update_interval: 45s
    offline_skip_updates: 5
```

If the Slave battery does not respond, check that it is physically connected to the same RS485 bus and configured with a unique Modbus address.

---

## RJ45 RS485 Pinout Documentation

Many Hubble AM2 / PACE-based batteries expose communication through RJ45-style ports.

> Important: RJ45 does not automatically mean Ethernet. On battery BMS systems, an RJ45 socket may be used for RS485, CAN, battery-to-battery linking, or inverter communication.

Always confirm the Hubble AM2 / PACE BMS pinout against the battery documentation before wiring.

---

## Common RJ45 Numbering

When looking at an RJ45 plug with the clip facing away from you and the copper contacts facing up, pin 1 is usually on the left:

```text
Copper contacts facing up
Clip facing away

Pin:  1  2  3  4  5  6  7  8
      |  |  |  |  |  |  |  |
     [------------------------]
```

Another view:

```text
RJ45 plug front view, contacts up:

  1  2  3  4  5  6  7  8
┌────────────────────────────┐
│ ▓  ▓  ▓  ▓  ▓  ▓  ▓  ▓     │
└────────────────────────────┘
```

Always confirm orientation before crimping or probing.

---

## Common PACE / Pylon-Style RS485 RJ45 Pinout

A commonly used RS485 pinout on many PACE/Pylon-style lithium batteries is:

| RJ45 Pin | Signal | Notes |
|---|---|---|
| Pin 1 | RS485-B / 485B / D- | Inverting RS485 line |
| Pin 2 | RS485-A / 485A / D+ | Non-inverting RS485 line |
| Pin 3 | NC / Reserved | May vary by battery |
| Pin 4 | CAN-H | Used for CAN, not RS485 |
| Pin 5 | CAN-L | Used for CAN, not RS485 |
| Pin 6 | NC / Reserved | May vary by battery |
| Pin 7 | GND / COM | Communication ground on some models |
| Pin 8 | GND / COM | Communication ground on some models |

> Warning: This is a common pattern, not a universal rule. Some batteries swap A/B, use different ground pins, or use separate RJ45 ports for inverter CAN and RS485 monitoring.

---

## RS485 Connection from RJ45 to RS485 Board

If your Hubble AM2 / PACE BMS uses the common pinout above, the connection to the RS485 board would normally be:

| Battery RJ45 Pin | Battery Signal | RS485 Board |
|---|---|---|
| Pin 1 | RS485-B / D- | B / D- / 485- |
| Pin 2 | RS485-A / D+ | A / D+ / 485+ |
| Pin 7 or 8 | GND / COM | GND, if required |

Example:

```text
Battery RJ45 Pin 2  RS485-A / D+  ───── RS485 board A / D+ / 485+
Battery RJ45 Pin 1  RS485-B / D-  ───── RS485 board B / D- / 485-
Battery RJ45 Pin 7  GND / COM     ───── RS485 board GND, if required
```

If there is no communication but everything else is correct, try swapping A and B:

```text
Battery A / D+  -> RS485 board B / D-
Battery B / D-  -> RS485 board A / D+
```

Some manufacturers label A/B opposite to the RS485 board manufacturer.

---

## T568B RJ45 Wire Colours

If the RJ45 cable uses standard T568B wiring, the wire colours are usually:

| RJ45 Pin | T568B Colour |
|---|---|
| Pin 1 | White/Orange |
| Pin 2 | Orange |
| Pin 3 | White/Green |
| Pin 4 | Blue |
| Pin 5 | White/Blue |
| Pin 6 | Green |
| Pin 7 | White/Brown |
| Pin 8 | Brown |

Using the common PACE/Pylon-style RS485 pinout:

| Function | RJ45 Pin | T568B Colour |
|---|---|---|
| RS485-B / D- | Pin 1 | White/Orange |
| RS485-A / D+ | Pin 2 | Orange |
| GND / COM | Pin 7 or 8 | White/Brown or Brown |

> Always test continuity with a multimeter. Do not rely only on cable colour, especially with pre-made or non-standard cables.

---

## CAN Port vs RS485 Port

Many Hubble/PACE-style batteries have multiple RJ45 communication ports, for example:

```text
CAN / Inverter communication port
RS485 / BMS monitoring port
Link / battery-to-battery port
```

This ESPHome project uses RS485/Modbus.

Do not connect the ESP32 RS485 board to CAN-H or CAN-L.

If the battery has a dedicated RS485 port, use that port.

If the battery uses the same physical RJ45 connector for multiple communication types, confirm which pins are RS485 A/B before connecting.

---

## RJ45 Safety Checklist

Before powering the ESP32/RS485 monitor:

```text
Confirm the battery RJ45 pinout from the manual
Confirm whether the port is RS485, not CAN-only
Confirm RJ45 pin 1 orientation
Confirm A/B wiring with a continuity tester
Confirm GND/COM is connected only if required
Confirm no battery power pins are connected to the RS485 board
Confirm only one ESP32/device is polling the RS485 bus
```

Incorrect RJ45 wiring can prevent communication and may damage hardware if voltage pins are connected incorrectly.

---

## Auto Cell-Count Detection

The configuration reads up to 16 possible cell-voltage registers per pack.

Only realistic lithium cell voltages are counted as valid cells.

Typical valid range:

```text
2.0V to 5.0V
```

For Hubble AM2 packs, the expected result is:

```text
Master detected cells = 13
Slave detected cells  = 13
Combined detected cells = 26
```

Example behaviour:

```text
Cell 1-13 = valid voltage
Cell 14-16 = unavailable / NaN
Detected cell count = 13
```

The unused cells are not included in:

```text
Minimum cell voltage
Maximum cell voltage
Average cell voltage
Delta cell voltage
Detected cell count
Combined battery calculations
```

---

## Combined Battery Calculations

For Master/Slave batteries connected in parallel, combined values are calculated as follows:

| Combined Value | Calculation |
|---|---|
| Total Current | Master current + Slave current |
| Total Power | Master power + Slave power |
| Total Remaining Capacity | Master remaining Ah + Slave remaining Ah |
| Total Full Capacity | Master full Ah + Slave full Ah |
| Total Design Capacity | Master design Ah + Slave design Ah |
| Average Pack Voltage | Average of valid Master/Slave voltages |
| Combined SOC | Weighted by remaining capacity and full capacity |
| Combined SOH | Based on full capacity versus design capacity |
| Total Detected Cells | Master detected cells + Slave detected cells |
| Lowest Cell Voltage | Lowest valid cell across both packs |
| Highest Cell Voltage | Highest valid cell across both packs |
| Worst Delta Cell Voltage | Highest pack delta |

> Do not add pack voltages together when packs are connected in parallel.

---

## Projected Runtime and Charging Time

The project includes projected runtime and charging-time sensors.

Runtime while discharging:

```text
Remaining capacity Ah ÷ discharge current A = runtime
```

Charging time while charging:

```text
(Full capacity Ah - remaining capacity Ah) ÷ charge current A = time to full
```

The projected time sensors publish in minutes to avoid Home Assistant displaying sub-1-hour values as `0 h`.

Example:

```text
Projected Runtime: 45 min
Projected Charging Time: 38 min
```

The sensors intentionally return unavailable/NaN when not applicable:

```text
Runtime while charging = unavailable
Charging time while discharging = unavailable
Charging time when battery is already full = 0 min
Runtime when current is near zero = unavailable
```

This avoids misleading values such as extremely high runtime estimates when current is close to zero.

---

## Data Trust and Timeout Behaviour

The configuration is designed not to hide communication failures by holding stale values forever.

The recommended behaviour is:

```text
Good fresh value received = publish value
Invalid/impossible value = reject value
No fresh live value within timeout = unavailable
```

This helps ensure that Home Assistant cards show actual live data rather than old values that make a failed link look healthy.

---

## MQTT and Home Assistant

Suggested MQTT identity for this project:

```yaml
mqtt:
  topic_prefix: hubble_am2/combined
```

Suggested ESPHome identity:

```yaml
substitutions:
  name: hubble-am2-combined
  friendly_name: "Hubble AM2 Combined"
  device_description: "Hubble AM2 Master/Slave PACE RS485 battery monitor"
```

Example Home Assistant entities:

```text
sensor.hubble_am2_combined_master_state_of_charge
sensor.hubble_am2_combined_slave_state_of_charge
sensor.hubble_am2_combined_combined_state_of_charge
sensor.hubble_am2_combined_combined_total_current
sensor.hubble_am2_combined_combined_projected_runtime
sensor.hubble_am2_combined_combined_projected_charging_time
```

---

## Home Assistant Dashboard Card

The repository can include a `card.yaml` file for a Lovelace dashboard card.

Recommended card sections:

```text
Combined overview
Projected runtime / charging time
Combined cell health
Master pack summary
Slave pack summary
Master/Slave cell health
Master/Slave temperatures
Master/Slave BMS status
Master/Slave cell voltages
Master/Slave cell balancing
Diagnostics / identity
```

---

## Troubleshooting

### Master reads, but Slave does not

Check:

```text
Slave is connected to the same RS485 bus
Slave has a unique Modbus address
Slave address matches the YAML address
RS485 A/B wiring is correct
Only one ESP32 is polling the bus
```

### All readings are unavailable

Check:

```text
ESP32 UART pins
RS485 board power
RS485 DE/RE flow-control pin
RS485 A/B polarity
Battery RS485 port selection
Modbus baud rate
Battery communication address
```

### Partial responses or Modbus timeouts

Check:

```text
RS485 wiring quality
Cable length
Termination resistor
Multiple devices polling the same bus
Update intervals too aggressive
MQTT overload from too many sensors
```

### Cell 14-16 show unavailable

This is normal for Hubble AM2 13S packs.

The sensors remain available in the YAML for compatibility with 16S PACE BMS layouts, but they are ignored in calculations when invalid.

### Projected charging time shows 0 min

This is normal when the battery is already full.

Example:

```text
Full capacity = 90.30 Ah
Remaining capacity = 90.30 Ah
Capacity needed = 0 Ah
Charging time = 0 min
```

### Projected runtime shows unavailable

This is normal when the battery is not discharging or when current is too close to zero.

### Warnings show Cell Overvoltage / Battery Full

This can happen when the battery is full and individual cells are near the upper protection threshold. Check the BMS status, cell voltages, inverter charge settings, and manufacturer limits.

---

## Safety Notes

This project is for monitoring only.

It should not write to the BMS or change battery settings.

Always follow battery manufacturer wiring guidelines, RS485 pinout documentation, and electrical safety practices.

---

## License

Suggested open-source license:

```text
MIT License
```

---

## Disclaimer

This project is provided for monitoring and educational purposes. Use it at your own risk. Battery systems can be hazardous. Incorrect wiring or configuration may damage equipment or create unsafe conditions.
