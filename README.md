# 8311 HA Bridge

Connects to the BFW Solutions WAS-110 XGS-PON ONU and integrates fiber optic statistics into Home Assistant using MQTT Auto Discovery.

This script monitors optical power levels, temperature, voltage, and link status from the WAS-110 via SSH, and publishes corresponding sensor entities to Home Assistant via your local MQTT broker.

## Features

* **XGS-PON/GPON Monitoring:** Real-time monitoring of fiber optic metrics from WAS-110 ONU
* **Multiple Metrics:** RX/TX optical power (dBm), temperature (C), voltage (V), laser bias (mA), link status
* **Home Assistant Auto Discovery:** Automatically creates and updates sensor entities via MQTT discovery
* **Organized HA Device:** Creates a dedicated device `8311 ONU ({SERIAL})` with all sensors grouped together
* **SSH-Based Monitoring:** Uses native SSH to execute monitoring commands on the WAS-110
* **Data Enrichment:**
    * Parses I2C EEPROM data for optical metrics (EEPROM50/51)
    * Extracts temperature, voltage, and current readings
    * Monitors PON link status with state codes
    * CPU temperature monitoring
* **MQTT Compatibility:** Sanitizes device IDs and topics for MQTT/HA compatibility

## Requirements

* **Python 3:** Version 3.9 or higher (tested with 3.12)
* **Python Libraries:** `paho-mqtt>=2.0.0` (see `requirements.txt`)
* **WAS-110 Device:** BFW Solutions WAS-110 XGS-PON ONU with [8311 community firmware](https://github.com/up-n-atom/8311)
* **SSH Access:** SSH client and access to WAS-110 (default: root@192.168.11.1)
* **MQTT Broker:** An MQTT broker accessible on your network
    * *Recommended:* The [Mosquitto broker Home Assistant Add-on](https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md)
* **Home Assistant MQTT Integration:**
    * The [MQTT Integration](https://www.home-assistant.io/integrations/mqtt/) must be installed and configured
    * MQTT Discovery must be enabled (default: `homeassistant` discovery prefix)

## Installation & Setup

### Option 1: Python Virtual Environment

1. **Clone/Download:** Obtain the project files and place them in a dedicated directory
2. **Create Virtual Environment:**
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate
    ```
3. **Install Dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
4. **Configure:** Edit variables at top of `8311-ha-bridge.py` or use environment variables (see `.env.example`)
5. **Run:**
    ```bash
    python3 8311-ha-bridge.py
    ```

### Option 2: Docker

1. **Build:**
    ```bash
    docker build -t 8311-ha-bridge .
    ```
2. **Run:**
    ```bash
    docker run -d --name 8311-ha-bridge \
      -e WAS_110_HOST=192.168.11.1 \
      -e HA_MQTT_BROKER=homeassistant.local \
      8311-ha-bridge
    ```

## Configuration

Edit these variables in `8311-ha-bridge.py` or set as environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `WAS_110_HOST` | IP address of your WAS-110 ONU | `192.168.11.1` |
| `WAS_110_USER` | SSH username | `root` |
| `WAS_110_PASS` | SSH password | `""` (empty) |
| `HA_MQTT_BROKER` | MQTT broker hostname/IP | `homeassistant.local` |
| `HA_MQTT_PORT` | MQTT broker port | `1883` |
| `HA_MQTT_USER` | MQTT username | `None` |
| `HA_MQTT_PASS` | MQTT password | `None` |
| `POLL_INTERVAL_SECONDS` | Polling interval | `60` |
| `DEBUG_MODE` | Enable verbose logging | `False` |
| `TEST_MODE` | Run single test cycle | `False` |

## Home Assistant Integration

Once the script is running and connected to your HA MQTT broker:

1. Navigate to **Settings -> Devices & Services -> MQTT**
2. You should see a **new device** discovered: `8311 ONU ({SERIAL_NUMBER})`
3. Click into this device to see the associated sensor entities
4. Add these sensors to your Lovelace dashboards!

See `examples/home_assistant_dashboard.yaml` for a sample dashboard configuration.

## Available Sensors

### Optical Performance
* **RX Power (dBm)** - Receiver optical power
* **RX Power (mW)** - Receiver optical power in milliwatts
* **TX Power (dBm)** - Transmitter optical power
* **TX Power (mW)** - Transmitter optical power in milliwatts
* **Voltage** - Module VCC voltage
* **TX Bias Current** - Laser bias current in mA

### Temperature
* **Optic Temperature** - Optical module temperature
* **CPU0 Temperature** - CPU thermal zone 0
* **CPU1 Temperature** - CPU thermal zone 1

### Network Status
* **PON Link Status** - PON link up/down with state details
* **SSH Connection Status** - Bridge connectivity to ONU
* **Ethernet Speed** - Negotiated ethernet speed

### Device Information
* **Vendor Name** - Device manufacturer
* **Part Number** - Device part number
* **Hardware Revision** - Hardware revision
* **PON Mode** - XGS-PON or GPON mode
* **Firmware Bank** - Active firmware bank (A/B)
* **Bridge Uptime** - Bridge runtime with statistics

## Troubleshooting

* **Script doesn't start:** Check Python version (`python3 --version`), ensure dependencies installed (`pip list`)
* **Cannot connect to WAS-110:** Verify SSH access: `ssh root@192.168.11.1` - Check credentials and firewall rules
* **No Connection to HA MQTT:** Verify MQTT broker settings. Check HA MQTT broker logs
* **Missing sensors in HA:** Ensure MQTT Discovery is enabled. Check MQTT Explorer for discovery messages
* **SSH timeouts:** The WAS-110 has aggressive rate limiting. The script combines commands to avoid this, but the device may need a few seconds between connection attempts
* **Device not responding:** The WAS-110 may enter a low-power state. Accessing the web UI at https://192.168.11.1 may "wake" it

## Technical Details

### Data Sources

The script reads data from multiple sources on the WAS-110:

1. **EEPROM51** (`/sys/class/pon_mbox/pon_mbox0/device/eeprom51`) - Real-time optical diagnostics
2. **EEPROM50** (`/sys/class/pon_mbox/pon_mbox0/device/eeprom50`) - Static device information
3. **sysfs** - CPU temperatures, ethernet speed
4. **8311 shell functions** - PON status (`pon psg`), firmware bank (`active_fwbank`)
5. **UCI config** - PON mode detection

### SSH Implementation

The script uses the native `ssh` command via Python's `subprocess` module rather than paramiko. This approach was chosen after debugging revealed authentication compatibility issues between paramiko and the WAS-110's Dropbear SSH server.

Commands are combined into single SSH sessions using delimiters to avoid the device's aggressive SSH rate limiting.

## Project Status

**Version:** 1.0.0
**Status:** Production Ready

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Acknowledgements

* **8311 Community:** For the excellent WAS-110 firmware at [github.com/up-n-atom/8311](https://github.com/up-n-atom/8311)
* **PON dot WIKI:** For comprehensive documentation at [pon.wiki](https://pon.wiki/)
* **Home Assistant Project:** For the amazing home automation platform
* **Eclipse Paho MQTT Python Client Library:** (`paho-mqtt`)

## Resources

* [8311 Community GitHub](https://github.com/up-n-atom/8311)
* [WAS-110 Documentation - PON Wiki](https://pon.wiki/xgs-pon/ont/bfw-solutions/was-110/)
* [8311 Community Discord](https://discord.pon.wiki)
* [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
