# PaceBMS — Pace BMS to MQTT Bridge

A Python-based bridge for **Pace BMS** battery management systems. It reads live battery data from the BMS and publishes it to MQTT with **Home Assistant auto-discovery** support.

This project is for **Pace BMS protocol batteries** using the RS232/UART ASCII frame format. It is **not limited to Hubble**; Hubble AM2 is simply one tested example. The monitor auto-detects the number of packs and the number of cells reported by each pack, then publishes the matching Home Assistant MQTT entities. Additional packs are detected automatically when the batteries are correctly linked and their DIP switches / pack addresses are configured correctly.

---

## File Versions

| File | Version | Changed | Notes |
|------|---------|---------|-------|
| `bms_monitor.py` | 2.2.0 | 2026-05-16 | Direct Telegram notification engine, proper serial no-response disconnect detection, clean service-stop shutdown handling, MQTT status/error payloads, startup/shutdown/recovery events |
| `bms_notify.py` | 1.0.0 | 2026-05-16 | Notification engine for Telegram alerts, SOC thresholds, warnings, FET alerts, SOH alerts, disconnect/recovery, daily summary, and cell delta reports |
| `constants.py` | 1.0.0 | 2026-05-16 | Pace BMS protocol constants, CID2 codes, warning/protection state maps |
| `config.yaml` | 2.0.15 | 2026-05-16 | Serial mode, Telegram settings, notification toggles, retry threshold, report schedule, MQTT settings, and HA add-on schema |

---

## Features

- Reads cell voltages, temperatures, current, voltage, SOC, SOH, cycles, and capacity per pack
- Auto-detects the number of packs reported by the master BMS, provided linked packs have correct DIP switch / address settings
- Auto-detects the number of cells per pack, for example 13-cell Hubble AM2 packs or 16-cell Pace lithium packs
- Supports multiple battery packs through the master battery connection where the BMS exposes linked packs
- Publishes all data to MQTT with Home Assistant MQTT Discovery
- Publishes Home Assistant availability using `pacebms/availability`
- Sends Telegram notifications directly from Python, without needing Home Assistant automations
- Detects real BMS communication loss when the serial adapter is still present but the battery is not replying
- Sends disconnect and recovery Telegram notifications
- Sends service startup and service stopped Telegram notifications
- Publishes startup, shutdown, disconnect, and recovery events to MQTT as JSON
- Supports SOC low alerts, SOC high alerts, warning flag alerts, FET alerts, SOH alerts, daily summary reports, and cell delta reports
- Uses retained MQTT state topics by default so Home Assistant has values after restart
- Runs as a Home Assistant add-on, standalone Docker container, or direct Python script
- Supports Serial and TCP/IP connection modes
- Structured logging with configurable debug levels

---

## Supported Hardware

| BMS | Connection | Tested |
|-----|------------|--------|
| Pace BMS P16S200A | Serial USB | Yes |
| Pace BMS AM-x series / Hubble AM2 | USB-RS232 to master battery RS232 port | Yes |
| Other batteries using the Pace RS232/UART ASCII protocol | Serial USB or TCP/IP, depending on hardware | Expected, protocol-compatible |

The monitor is intended for batteries that expose the **Pace BMS RS232/UART ASCII protocol**. It is not brand-locked to Hubble. Hubble AM2 is included because it has been tested and because it commonly uses a Pace-compatible BMS.

---

## Auto Cell and Pack Detection

The monitor does **not** assume a fixed 16-cell battery and it does **not** assume a fixed number of battery packs. During each analog-data read, the BMS response includes the pack count and the cell count for each pack. The script reads those values and creates/publishes only the packs and cells reported by the BMS.

Additional packs are detected automatically when:

- The batteries are linked correctly using the manufacturer's battery link / communication ports.
- The master battery is configured as the master.
- Each slave battery has a unique address using the correct DIP switch settings.
- The master BMS reports the linked packs through the Pace protocol.

Examples:

- A 13-cell Pace-compatible pack will publish `cell_01` to `cell_13`.
- A 16-cell Pace-compatible pack will publish `cell_01` to `cell_16`.
- A correctly configured master/slave installation will publish the packs returned by the master BMS, such as `pack_1` and `pack_2`.
- If a slave pack is not shown, first check the battery link cable, DIP switch addressing, and whether the master BMS can see that slave pack.

The `zero_pad_number_cells` setting only controls the **topic name format**. It does not force the number of cells. For example, `zero_pad_number_cells: 2` gives names like `cell_01`, `cell_02`, and `cell_13`.

The `zero_pad_number_packs` setting only controls the **topic name format**. It does not force the number of packs. For example, `zero_pad_number_packs: 0` gives names like `pack_1` and `pack_2`.

---

## Multi-Pack Addressing Notes

For multi-pack systems, this monitor reads pack information reported by the master BMS. It does not manually create extra packs in software. If additional packs do not appear in Home Assistant, the usual cause is that the master BMS is not reporting them.

Check the following first:

- The packs are connected using the correct battery link / communication cable.
- The master pack is set to the correct master address.
- Each slave pack has a unique DIP switch address.
- The battery manufacturer allows multi-pack data to be queried from the master using the Pace protocol.
- The monitor is connected to the port that exposes the Pace RS232/UART ASCII protocol.

Once the master BMS reports the additional pack, the monitor will publish it automatically.


### Example DIP switch address table

Some Pace BMS manuals define the battery address using a 4-position DIP switch. The exact switch direction can vary by battery brand and enclosure, so always confirm against your own battery manual. The example table below follows the manual layout where switch **#1** is the lowest address bit.

| Address | DIP #1 | DIP #2 | DIP #3 | DIP #4 | Typical role in manual |
|---------|--------|--------|--------|--------|------------------------|
| 0 | OFF | OFF | OFF | OFF | Independent / single battery use |
| 1 | ON | OFF | OFF | OFF | Main / master pack |
| 2 | OFF | ON | OFF | OFF | Pack 1 / first slave |
| 3 | ON | ON | OFF | OFF | Pack 2 |
| 4 | OFF | OFF | ON | OFF | Pack 3 |
| 5 | ON | OFF | ON | OFF | Pack 4 |
| 6 | OFF | ON | ON | OFF | Pack 5 |
| 7 | ON | ON | ON | OFF | Pack 6 |
| 8 | OFF | OFF | OFF | ON | Pack 7 |
| 9 | ON | OFF | OFF | ON | Pack 8 |
| 10 | OFF | ON | OFF | ON | Pack 9 |
| 11 | ON | ON | OFF | ON | Pack 10 |
| 12 | OFF | OFF | ON | ON | Pack 11 |
| 13 | ON | OFF | ON | ON | Pack 12 |
| 14 | OFF | ON | ON | ON | Pack 13 |
| 15 | ON | ON | ON | ON | Pack 14 |

In plain terms: for multi-pack operation, do not leave every battery on the same DIP setting. One pack must be set as the main/master pack, and each additional pack must have its own unique address. Once the master BMS sees those packs, this software will publish them automatically.

---

## Hubble AM2 Notes

The Hubble AM2 has RS232 and RS485 communication ports. This project uses the **RS232 port** on the **master battery**.

Recommended setup:

1. Connect the USB-RS232 adapter to the **RS232 port** on the master battery.
2. Link the slave battery to the master using the normal battery link cable.
3. Set DIP switches correctly according to the battery manual. A common Pace-style setup is:
   - Single independent battery = address 0
   - Main / master battery = address 1
   - First slave battery = address 2
   - Next slave batteries = unique addresses after that
4. Read all reported packs through the master battery.

Important: the Hubble AM2 RS485 port uses Modbus RTU and is not the same as the Pace RS232 ASCII protocol used by this monitor.

---

## Requirements

- Python 3.11+
- MQTT broker, for example Mosquitto
- Home Assistant, optional but recommended
- USB-RS232 adapter for serial monitoring, where required by your Pace BMS hardware

Python dependencies:

```txt
paho-mqtt
pyserial
pyyaml
```

Telegram uses Python's built-in `urllib`, so no extra Telegram library is required.

---

## Installation

### Option A — Home Assistant Add-on

1. In Home Assistant, go to **Settings -> Add-ons -> Add-on Store**.
2. Click the three-dot menu and choose **Repositories**.
3. Add this repository:

```txt
https://github.com/saratrax13-sketch/serial_rs232_pacebms
```

4. Install the add-on.
5. Configure the add-on under the **Configuration** tab.
6. Start the add-on.

### Option B — Docker Compose

```bash
git clone https://github.com/saratrax13-sketch/serial_rs232_pacebms.git
cd serial_rs232_pacebms
```

Edit `config.yaml`, then start the container:

```bash
docker compose up -d
```

View logs:

```bash
docker compose logs -f
```

### Option C — Direct Python

```bash
git clone https://github.com/saratrax13-sketch/serial_rs232_pacebms.git
cd serial_rs232_pacebms
pip install -r requirements.txt
python3 bms_monitor.py
```

The script loads configuration from `/data/options.json` when running as a Home Assistant add-on. For local development it also checks for `pace-bms-dev/config.yaml`.

---

## Configuration

All settings are configured under the `options:` section in `config.yaml`, or through the Home Assistant add-on Configuration tab.

### Main settings

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `telegram_bot_token` | string | — | Telegram bot token from BotFather |
| `telegram_chat_id` | string | — | Telegram chat ID for notifications |
| `notify_enabled` | bool | `true` | Master switch for all Telegram notifications |
| `mqtt_host` | string | — | MQTT broker IP address or hostname |
| `mqtt_port` | int | `1883` | MQTT broker port |
| `mqtt_user` | string | — | MQTT username |
| `mqtt_password` | string | — | MQTT password |
| `mqtt_base_topic` | string | `pacebms` | Root MQTT topic |
| `mqtt_ha_discovery` | bool | `true` | Enable Home Assistant MQTT Discovery |
| `mqtt_ha_discovery_topic` | string | `homeassistant` | Home Assistant discovery prefix |
| `connection_type` | `IP` or `Serial` | `Serial` | BMS connection method |
| `bms_ip` | string | — | BMS IP address, only used in IP mode |
| `bms_port` | int | `5000` | BMS TCP port, only used in IP mode |
| `bms_serial` | string | `/dev/ttyUSB0` | Serial device path, only used in Serial mode |
| `bms_baudrate` | int | `9600` | Serial baud rate |
| `scan_interval` | int | `5` | Seconds between full BMS polling cycles |
| `debug_output` | int | `0` | `0` info only, `1` debug, `3` raw frames |
| `zero_pad_number_cells` | int | `2` | Cell topic padding, for example `cell_01` |
| `zero_pad_number_packs` | int | `0` | Pack topic padding. `0` gives `pack_1`, `pack_2` |

### Notification settings

| Option | Default | Description |
|--------|---------|-------------|
| `notify_soc_low` | `true` | Alert when SOC crosses configured low thresholds |
| `notify_soc_high` | `true` | Alert when SOC reaches high-charge threshold |
| `notify_warnings` | `true` | Alert on BMS warning flags |
| `notify_fet` | `true` | Alert when charge/discharge FET turns off unexpectedly |
| `notify_soh` | `true` | Alert when SOH drops below configured threshold |
| `notify_disconnect` | `true` | Alert when the BMS stops replying and when it recovers |
| `notify_startup` | `true` | Alert when monitor starts or stops |
| `notify_daily_summary` | `true` | Send daily kWh and worst cell summary |
| `notify_delta_report` | `true` | Send cell delta report for the configured window |
| `notify_soc_low_thresholds` | `75,50,25,10` | SOC levels that trigger low battery alerts |
| `notify_soc_high_threshold` | `98` | SOC level that triggers high-charge alert |
| `notify_soc_high_reset` | `95` | SOC must drop below this before another high-charge alert is sent |
| `notify_soh_threshold` | `95` | SOH percentage below which an alert is sent |
| `notify_retry_count` | `1` | Failed BMS communication attempts before disconnect alert. `1` means alert on first failed read |
| `notify_daily_summary_time` | `19:00` | Daily summary time |
| `notify_delta_report_time` | `10:15` | Cell delta report time |
| `notify_delta_window_start` | `00:00` | Start of delta tracking window |
| `notify_delta_window_end` | `10:00` | End of delta tracking window |

Recommended value for fast disconnect testing:

```yaml
notify_retry_count: 1
```

This is important for serial testing because the USB adapter may remain connected even when the battery-side cable is unplugged. The monitor now treats a failed BMS response as the actual disconnect condition.

---

## Finding the Serial Device Path

Use the stable `by-id` path where possible:

```bash
ls /dev/serial/by-id/
```

Example:

```yaml
bms_serial: "/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0"
```

This is better than `/dev/ttyUSB0` or `/dev/ttyUSB1`, because USB numbering can change after a reboot.

---

## MQTT Topics

All topics are published under `{mqtt_base_topic}/`. The default base topic is `pacebms/`.

### Per-pack topics

With `zero_pad_number_packs: 0`, pack topics are published as `pack_1`, `pack_2`, etc.

| Topic | Unit | Description |
|-------|------|-------------|
| `pacebms/pack_1/v_cells/cell_01` | mV | Individual cell voltage |
| `pacebms/pack_1/temps/temp_1` | °C | Temperature sensor |
| `pacebms/pack_1/v_pack` | V | Pack voltage |
| `pacebms/pack_1/i_pack` | A | Pack current. Negative means charging |
| `pacebms/pack_1/soc` | % | State of charge |
| `pacebms/pack_1/soh` | % | State of health |
| `pacebms/pack_1/i_remain_cap` | Ah | Remaining capacity |
| `pacebms/pack_1/i_full_cap` | Ah | Full charge capacity |
| `pacebms/pack_1/i_design_cap` | Ah | Design capacity |
| `pacebms/pack_1/cycles` | — | Charge cycle count |
| `pacebms/pack_1/cells_max_diff_calc` | mV | Maximum cell voltage spread |
| `pacebms/pack_1/warnings` | — | Active warning string |
| `pacebms/pack_1/balancing1` | — | Cell balancing state bits |
| `pacebms/pack_1/balancing2` | — | Cell balancing state bits |
| `pacebms/pack_1/charge_fet` | ON/OFF | Charge FET state |
| `pacebms/pack_1/discharge_fet` | ON/OFF | Discharge FET state |
| `pacebms/pack_1/prot_short_circuit` | ON/OFF | Short circuit protection active |

### Aggregate topics

| Topic | Unit | Description |
|-------|------|-------------|
| `pacebms/pack_remain_cap` | Ah | Total remaining capacity across all packs |
| `pacebms/pack_full_cap` | Ah | Total full capacity |
| `pacebms/pack_design_cap` | Ah | Total design capacity |
| `pacebms/pack_soc` | % | Overall SOC |
| `pacebms/pack_soh` | % | Overall SOH |
| `pacebms/availability` | online/offline | Bridge availability |
| `pacebms/bms_version` | — | BMS firmware version |
| `pacebms/bms_sn` | — | BMS serial number |
| `pacebms/pack_sn` | — | Battery pack serial number |

### Status and error topics

| Topic | Payload | Description |
|-------|---------|-------------|
| `pacebms/bms_status` | JSON | Startup and shutdown events |
| `pacebms/bms_error` | JSON | Disconnect and recovery events |

Example startup payload:

```json
{"status": "startup", "bms_sn": "ABC123", "bms_version": "1.23"}
```

Example shutdown payload:

```json
{"status": "shutdown", "bms_sn": "ABC123", "timestamp": 1778940000}
```

Example disconnect payload:

```json
{"status": "disconnected", "reason": "Receive failed", "retry_count": 1, "offline_time": "0s", "offline_secs": 0, "timestamp": 1778940000}
```

Example recovery payload:

```json
{"status": "recovered", "retry_count": 1, "offline_time": "10s", "offline_secs": 10, "timestamp": 1778940010}
```

---

## Telegram Notifications

Notifications are sent directly from `bms_notify.py` using the Telegram Bot API.

Home Assistant automations are no longer required for the core Telegram messages. MQTT status and error topics are still published, so Home Assistant automations can still be added if you want phone push notifications or extra dashboards.

### Telegram setup

1. Open Telegram and create a bot with BotFather.
2. Copy the bot token into `telegram_bot_token`.
3. Send `/start` to your new bot.
4. Get your chat ID from a Telegram user info bot.
5. Copy the chat ID into `telegram_chat_id`.
6. Restart the add-on or service.

Example:

```yaml
telegram_bot_token: "YOUR_BOT_TOKEN"
telegram_chat_id: "123456789"
```

### Expected Telegram messages

You should receive messages for:

- Monitor started
- Monitor stopped
- BMS disconnected
- BMS reconnected
- Low SOC threshold reached
- Battery fully charged
- BMS warning detected
- BMS warning cleared
- FET alert
- SOH degradation alert
- Daily summary
- Cell delta report

---

## Disconnect and Service-Stop Behaviour

This version improves two important failure cases.

### Battery cable disconnected

When the cable between the battery and the serial adapter is disconnected, the USB adapter may still exist as `/dev/ttyUSB1`. Older logic could therefore think the serial port was still connected.

The monitor now treats an empty serial read as a real communication failure:

```txt
BMS serial recv: no data received before timeout
```

It then publishes `pacebms/availability = offline`, sends a Telegram disconnect alert, and retries the BMS connection.

### Service stopped

The monitor now handles normal Python exit plus service termination signals. This allows the `BMS Monitor Stopped` Telegram message to be sent when the Home Assistant add-on, Docker container, or Python service is stopped cleanly.

---

## Home Assistant

When `mqtt_ha_discovery: true`, sensors are created automatically in Home Assistant.

Sensors include suitable Home Assistant metadata:

- Cell and pack voltages -> voltage
- Current -> current
- Temperatures -> temperature
- SOC/SOH -> battery percentage
- Ah capacity values -> measurement sensors without an energy device class

Binary sensors such as FETs and protection states use `ON` and `OFF` payloads.

Discovery topics are republished every hour and after reconnects.

---

## Architecture

```txt
Pace-compatible BMS battery system
    |
    +-- Master battery RS232/UART Pace protocol port
    |
USB-RS232 adapter
    |
bms_monitor.py
    |
    +-- Pace protocol request/response parser
    +-- Data model: AnalogData / PackData / PackCapacity / WarnData
    +-- MQTT publisher
    +-- Home Assistant MQTT Discovery
    +-- bms_notify.py Telegram notification engine
    |
MQTT Broker
    |
Home Assistant
```

---

## Debugging

Set `debug_output` in config:

| Level | What you see |
|-------|--------------|
| `0` | Normal startup, connection, and pack summary logs |
| `1` | Debug logs, including skipped bytes between packs |
| `3` | Raw request and response frames |

View logs:

```bash
# Home Assistant add-on
Settings -> Add-ons -> BMS Pace -> Logs

# Docker
docker compose logs -f

# Direct Python
python3 bms_monitor.py
```

---

## Recommended Test Procedure

After installing the fixed files:

1. Start the add-on or service.
2. Confirm you receive `BMS Monitor Started` in Telegram.
3. Disconnect the battery-side serial cable.
4. Confirm you receive `BMS Disconnected`.
5. Reconnect the cable.
6. Confirm you receive `BMS Reconnected`.
7. Stop the add-on or service.
8. Confirm you receive `BMS Monitor Stopped`.

For quick testing, use:

```yaml
scan_interval: 5
notify_retry_count: 1
```

---

## Troubleshooting

### Sensors show `Unknown` in Home Assistant

Delete stale retained MQTT topics using MQTT Explorer, then restart the add-on so discovery is republished.

### Only some cells are showing

The parser auto-detects pack boundaries. Set `debug_output: 1` and check the logs.

### Slave or additional packs are not showing

The monitor can only publish packs that the master BMS reports. Check the battery link cable, DIP switch addressing, and master/slave configuration. Each slave pack must have a unique address according to the battery manufacturer's DIP switch table. After correcting the DIP switches, restart the BMS/monitor if required and check the logs for the detected pack count.

### Cell topics sort incorrectly

Keep this setting:

```yaml
zero_pad_number_cells: 2
```

This gives correctly sorted names such as `cell_01`, `cell_02`, and so on. It does not force 16 cells; a 13-cell pack will still only publish cells reported by the BMS.

### Pack topics are different from older README examples

With the current recommended config:

```yaml
zero_pad_number_packs: 0
```

Topics use:

```txt
pacebms/pack_1/...
pacebms/pack_2/...
```

not:

```txt
pacebms/pack_01/...
pacebms/pack_02/...
```

### Serial not connecting

Use the stable `/dev/serial/by-id/` path if possible. Also make sure the add-on has:

```yaml
uart: true
usb: true
```

For Hubble AM2, confirm that you are connected to the **RS232 port**, not the RS485 port. For other batteries, confirm that the port you are using exposes the Pace RS232/UART ASCII protocol.

### Disconnect alert only appears after reconnecting

Set:

```yaml
notify_retry_count: 1
```

Also make sure you are using the fixed `bms_monitor.py`, which treats empty serial reads as communication failures.

### Service stop does not send Telegram

Make sure you are using the fixed `bms_monitor.py` with signal handling enabled. The service must be stopped cleanly. A forced kill may not allow any program enough time to send a final Telegram message.

### Telegram notifications not working

Check the following:

- `notify_enabled: true`
- The specific notification toggle is enabled, for example `notify_disconnect: true`
- `telegram_bot_token` is correct
- `telegram_chat_id` is correct
- You sent `/start` to the bot in Telegram
- The Home Assistant host or Docker container has internet access

### Checksum errors

Usually indicates noisy serial communication, wrong baud rate, or the wrong port. Use:

```yaml
bms_baudrate: 9600
debug_output: 3
```

### RS485 port not responding

Some batteries expose Modbus RTU on RS485 and Pace ASCII protocol on RS232/UART. This monitor is for the Pace RS232/UART ASCII protocol path. If your RS485 port is Modbus-only, use the RS232/UART Pace protocol port instead.

---

## Notes on Home Assistant Automations

Home Assistant automations are optional in this version.

The monitor already sends Telegram directly. You may still use MQTT topics such as `pacebms/bms_status` and `pacebms/bms_error` to create extra HA automations, dashboards, persistent notifications, or mobile app push notifications.

---

## Contributing

Pull requests are welcome. Please test against a live BMS where possible and include relevant log output.

---

## License

MIT

---

## Acknowledgements

Originally based on the `bmspace` project by Tertiush.
