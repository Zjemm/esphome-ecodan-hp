# Syncing Heat Pump Temperature to Remote Thermostat

This complete guide explains how to add, configure, and use automatic temperature synchronization from your CN105 heat pump controller to your CNRF remote thermostat using REST API.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
   - [Quick Setup](#quick-setup)
   - [Building Custom BIN](#building-custom-bin)
4. [Configuration](#configuration)
5. [Usage](#usage)
6. [Troubleshooting](#troubleshooting)
7. [Technical Details](#technical-details)
8. [FAQ](#faq)

---

## Overview

This integration connects two separate ESP devices:

1. **CN105 Heat Pump Controller (Main ESP)** 
   - Reads temperature from the Ecodan heat pump via CN105 port
   - Monitors Zone 1 and/or Zone 2 room temperatures
   - Sends temperature data via HTTP POST

2. **CNRF Remote Thermostat (Secondary ESP)**
   - Acts as virtual remote thermostat via CNRF port
   - Receives temperature data via REST API
   - Controls the heat pump based on received temperature

The `sync-to-remote-thermostat.yaml` configuration automatically POSTs room temperature to the remote thermostat in real-time.

## Prerequisites

- **CN105 Heat Pump Controller** - Already configured and running
- **CNRF Remote Thermostat** - Already configured and running
- Both ESPs on the same network
- Know the IP address of your remote thermostat ESP

## Setup Instructions

### Step 1: Add ID to http_request Component

**⚠️ CRITICAL: This step is mandatory for the sync to work!**

Open your `ecodan-esphome.yaml` and find this line:

```yaml
http_request:
  buffer_size_tx: 20248
```

Change it to:

```yaml
http_request:
  buffer_size_tx: 20248
  id: http_request_component    # ADD THIS LINE
```

**Why is this needed?**
The sync configuration uses C++ lambda functions to build dynamic URLs (with IP address, room ID, and temperature values). In ESPHome, lambda functions can only call components that have an ID. Without this ID, the code will not compile.

Alternative approaches (without lambda) would be limited to static URLs and cannot handle the dynamic configuration we need for the Web UI.

### Step 2: Include the Sync Configuration

Add the sync configuration to your `ecodan-esphome.yaml` packages section:

```yaml
packages:
  base: !include confs/base.yaml
  zone1: !include confs/zone1.yaml          # Required for z1_room_temp sensor
  zone2: !include confs/zone2.yaml          # Required for z2_room_temp sensor (if using Zone 2)
  sync_thermostat: !include confs/sync-to-remote-thermostat.yaml  # ADD THIS LINE
  # ... other packages
```

### Step 3: Compile and Flash

#### Option A: Quick Installation (Using Existing Configuration)

Compile and flash your CN105 ESP with the updated configuration:

```bash
esphome run ecodan-esphome.yaml
```

If you get a compilation error about `http_request_component`, you forgot Step 1!

#### Option B: Building Custom BIN File

See the [Building Custom BIN](#building-custom-bin) section below for creating pre-compiled firmware files.

### Step 4: Configure via Web UI

After flashing, configure the sync settings via the Web UI:

1. Open your browser and navigate to: `http://<your-esp-ip>`
2. Find the sync configuration entities:
   - **Remote Thermostat IP**: Enter the IP address of your remote thermostat ESP (e.g., `192.168.1.55`)
   - **Sync Zone**: Select which zone to synchronize (`Zone 1` or `Zone 2`)
   - **Remote Room ID**: Enter the room identifier on the remote thermostat (`0` for room_0, `1` for room_1, etc.)
   - **Sync Interval**: Set the periodic sync interval in seconds (default: 60 seconds)
3. Enable synchronization:
   - Toggle **Sync to Remote Thermostat** switch to ON

The ESP will immediately start synchronizing the temperature!

## How It Works

### Architecture

```
┌─────────────────────────────┐         HTTP POST          ┌──────────────────────────────┐
│  CN105 Heat Pump ESP        │    ────────────────────>   │  CNRF Remote Thermostat ESP  │
│  ─────────────────────       │                            │  ──────────────────────────   │
│  • Reads temperature from   │                            │  • Receives temperature via  │
│    heat pump CN105 port     │                            │    REST API                  │
│  • Zone 1: z1_room_temp     │                            │  • Sends to heat pump via    │
│  • Zone 2: z2_room_temp     │                            │    CNRF port                 │
│  • Posts to:                │                            │  • Endpoint:                 │
│    /number/room_0/set       │                            │    /number/room_0            │
└─────────────────────────────┘                            └──────────────────────────────┘
```

### Real-time Updates
- When `z1_room_temp` or `z2_room_temp` sensor value changes, it immediately POSTs the new temperature
- Uses REST API endpoint: `http://<thermostat_ip>/number/room_<id>/set?value=<temperature>`
- Temperature changes are detected instantly (within 1 second)

### Periodic Fallback Sync
- A periodic sync runs at the configured interval (default: every 60 seconds)
- Ensures temperature stays synchronized even if the on_value trigger fails
- Acts as a watchdog to maintain consistency

### Configuration via Web UI
All settings are configurable without reflashing:
- **Remote Thermostat IP**: The IP address of your CNRF thermostat ESP
- **Sync Zone**: Choose between Zone 1 or Zone 2
- **Remote Room ID**: Select which room (0-7) on the remote thermostat
- **Sync Interval**: Adjust the periodic sync frequency (10-300 seconds)
- **Enable/Disable**: Toggle synchronization on/off

### Safety Features
- **IP validation**: Sync is skipped if IP address is empty
- **State persistence**: All settings are saved across reboots
- **Default OFF**: Sync is disabled by default to prevent accidental activation
- **Error logging**: Issues are logged for troubleshooting

### Monitoring
- **Last Synced Temperature** sensor shows the most recent temperature value sent
- Detailed log messages in ESP logs show all sync activity
- Log level: INFO for successful syncs, WARN for configuration issues

## Entities Created

After flashing, you'll see these new entities in Home Assistant / Web UI:

| Entity | Type | Default | Description |
|--------|------|---------|-------------|
| `Sync to Remote Thermostat` | Switch | OFF | Enable/disable temperature synchronization |
| `Remote Thermostat IP` | Text | "" | IP address of the remote thermostat ESP |
| `Remote Room ID` | Text | "0" | Room identifier on remote thermostat (0-7) |
| `Sync Zone` | Select | "Zone 2" | Choose which zone temperature to sync |
| `Sync Interval (seconds)` | Number | 60 | Periodic sync interval in seconds |
| `Last Synced Temperature` | Sensor | - | Last temperature value successfully sent |

## Troubleshooting

### Compilation Error: "http_request_component not found"

**Cause**: You forgot to add the ID to the http_request component.

**Solution**: 
1. Open `ecodan-esphome.yaml`
2. Find `http_request:` section
3. Add `id: http_request_component` line
4. Recompile

### No Temperature Updates on Remote Thermostat

**1. Verify sync is enabled**
- Check that "Sync to Remote Thermostat" switch is ON
- Default state is OFF for safety

**2. Check IP address**
```bash
ping <remote_thermostat_ip>
```
- Verify the IP is correct and reachable
- Both ESPs must be on the same network

**3. Verify room identifier**
- Ensure "Remote Room ID" matches your remote thermostat configuration
- Remote thermostat supports room_0 through room_7

**4. Check ESP logs**
Enable logging and look for sync messages:
```
[sync_thermostat:INFO] Zone 2 temperature changed to 21.5°C, updating remote thermostat at http://192.168.1.55/number/room_0/set?value=21.5
```

**5. Test REST API manually**
Verify the remote thermostat API is working:
```bash
# Test POST
curl -X POST "http://<thermostat_ip>/number/room_0/set?value=22.5" -d ""

# Test GET
curl -X GET "http://<thermostat_ip>/number/room_0"
```

Expected response:
```json
{"id":"number-room_0","value":"22.5","state":"22.5"}
```

### Temperature Not Reading from Heat Pump

**Cause**: Missing zone configuration or no thermostat connected.

**Solution**:
- Ensure `zone1.yaml` and/or `zone2.yaml` are included in your configuration
- Check that sensors `z1_room_temp` or `z2_room_temp` have valid values (not "unavailable")
- Verify a thermostat is connected to the heat pump that reports room temperature
- Some heat pump configurations require IN1 or remote thermostat to be enabled

### HTTP Request Errors in Logs

**Cause**: Network issues or firewall blocking.

**Solution**:
- Ensure both ESPs are on the same network/VLAN
- Check router firewall rules
- Verify no AP isolation is enabled on WiFi
- Try increasing the `timeout` in http_request configuration

### Remote Thermostat Receives Wrong Temperature

**Cause**: Wrong zone selected.

**Solution**:
- Verify "Sync Zone" selector shows the correct zone
- Zone 1 uses `z1_room_temp` sensor
- Zone 2 uses `z2_room_temp` sensor

### Sync Stops Working After Reboot

**Cause**: Settings might not be persisting or sync is disabled.

**Solution**:
- All settings have `restore_value: true` and should persist
- Check if "Sync to Remote Thermostat" switch is still ON after reboot
- Review ESP logs for any errors during startup

## Advanced Configuration

### Using Multiple Remote Thermostats

If you have multiple CNRF remote thermostats, you can sync to all of them:

**Option 1: Create additional sync configurations**
1. Copy `confs/sync-to-remote-thermostat.yaml` to `confs/sync-to-remote-thermostat-2.yaml`
2. Change all IDs to be unique (e.g., `sync_to_remote_enabled_2`, `remote_thermostat_ip_setting_2`)
3. Include both in your packages

**Option 2: Use different room IDs on the same thermostat**
- The remote thermostat supports up to 8 rooms (0-7)
- Configure multiple room IDs via "Remote Room ID" setting
- Each room can receive different zone temperatures

### Adjusting Sync Behavior

**Faster updates** (more network traffic):
- Decrease "Sync Interval" to 10-30 seconds
- Real-time on_value triggers still fire immediately

**Slower updates** (less network traffic):
- Increase "Sync Interval" to 120-300 seconds
- Suitable for stable temperature environments

**Disable periodic sync** (on_value only):
- Set "Sync Interval" to maximum (300 seconds)
- Only real-time temperature changes trigger updates

### Network Optimization

**Static IP for remote thermostat**:
Configure a static IP in your router's DHCP settings to prevent IP changes.

**DNS hostname support**:
If your network supports mDNS, you can use hostname instead of IP:
```
ecodan-thermostat.local
```
(Enter this in "Remote Thermostat IP" field)

## Technical Details

### REST API Reference

The CNRF remote thermostat exposes these REST endpoints:

**Get current temperature:**
```bash
curl -X GET "http://<thermostat_ip>/number/room_0"
```

Response:
```json
{"id":"number-room_0","value":"21.5","state":"21.5"}
```

**Set temperature:**
```bash
curl -X POST "http://<thermostat_ip>/number/room_0/set?value=22.5" -d ""
```

**Available room endpoints:**
- `/number/room_0` through `/number/room_7`
- Each room is independently configurable on the remote thermostat

### Implementation Details

**Why lambda functions are used:**

The sync configuration uses C++ lambda functions instead of pure YAML for several reasons:

1. **Dynamic URL building**: URLs must be constructed with user-configurable values (IP, room ID, temperature)
2. **Conditional logic**: Checks for empty IP, zone selection, and sensor availability
3. **String manipulation**: Formatting temperature values and building HTTP requests
4. **State management**: Tracking last sync time for interval-based syncing

YAML actions alone cannot provide this level of flexibility and dynamic behavior.

**Why http_request needs an ID:**

In ESPHome:
- **YAML actions** can reference components without IDs (e.g., `http_request.post`)
- **Lambda functions** (C++ code) require IDs to reference components (e.g., `id(http_request_component)`)

Since this configuration uses lambdas for flexibility, an ID is mandatory.

### Performance Considerations

- **Network traffic**: Each sync sends ~100 bytes HTTP POST request
- **CPU usage**: Minimal, lambda execution takes <1ms
- **Memory usage**: ~2KB for configuration and state storage
- **WiFi impact**: Negligible, even with 10-second intervals

### Security Considerations

- **No authentication**: REST API has no authentication by default
- **Local network only**: Should only be used on trusted local networks
- **No encryption**: HTTP is used (not HTTPS)
- **Consider**: Use VLANs or firewall rules to isolate IoT devices

---

## Building Custom BIN

If you want to create a pre-compiled firmware file (BIN) that includes this sync feature, follow these instructions.

### Prerequisites

- ESPHome installed: `pip3 install esphome`
- This repository cloned or downloaded
- Basic terminal/command line knowledge

### Quick Build Instructions

For a complete firmware with **English labels** and **2 zones** and **sync feature**:

#### Step 1: Create Build Configuration

Create a file named `ecodan-en-2zones-sync.yaml` in the main project folder:

```yaml
substitutions:
  name: "ecodan-heatpump"
  friendly_name: "Ecodan Heatpump"

esphome:
  name: ${name}
  friendly_name: ${friendly_name}
  project:
    name: "Zjemm.esphome-ecodan-hp"
    version: "1.1.0"

esp32: !include confs/esp32s3.yaml  # or esp32.yaml for regular ESP32

# CRITICAL: http_request must have an ID for sync to work!
http_request:
  buffer_size_tx: 20248
  id: http_request_component  # Required for sync functionality

ota: !include confs/ota.yaml
wifi: !include confs/wifi.yaml

packages:
  labels: !include confs/ecodan-labels-en.yaml  # English labels
  base: !include confs/base.yaml
  zone1: !include confs/zone1.yaml              # Zone 1 support
  zone2: !include confs/zone2.yaml              # Zone 2 support
  sync_thermostat: !include confs/sync-to-remote-thermostat.yaml  # Sync feature
  auto_adaptive: !include confs/auto-adaptive.yaml
  short_cycle: !include confs/short-cycle.yaml
  energy: !include confs/energy.yaml
  request_codes: !include confs/request-codes.yaml
```

#### Step 2: Compile

```bash
cd /path/to/esphome-ecodan-hp-main
esphome compile ecodan-en-2zones-sync.yaml
```

**Expected build time:**
- First build: 10-15 minutes (downloads dependencies)
- Subsequent builds: 2-3 minutes

#### Step 3: Locate BIN File

After successful compilation, find your BIN file:

```bash
# ESP32-S3:
.esphome/build/ecodan-heatpump/.pioenvs/ecodan-heatpump/firmware.bin

# Copy and rename:
cp .esphome/build/ecodan-heatpump/.pioenvs/ecodan-heatpump/firmware.bin \
   ecodan-hp-en-2zones-sync-esp32s3.bin
```

**BIN file size:** Approximately 1.5-2.0 MB

### Flashing the BIN File

#### First-Time Flash (USB Required)

Use ESPHome web flasher or esptool:

```bash
# Using ESPHome
esphome run ecodan-en-2zones-sync.yaml

# Or using esptool
pip3 install esptool
esptool.py --port /dev/ttyUSB0 write_flash 0x0 ecodan-hp-en-2zones-sync-esp32s3.bin
```

#### Over-The-Air (OTA) Updates

After first flash, use OTA:

```bash
esphome upload ecodan-en-2zones-sync.yaml --device <esp-ip>
```

Or upload BIN via Home Assistant or ESPHome Dashboard.

### Hardware Variants

#### For Regular ESP32

Change in your YAML:
```yaml
esp32: !include confs/esp32.yaml  # Instead of esp32s3.yaml
```

#### For M5Stack Atom S3 with LED

```yaml
esp32: !include confs/esp32s3-led.yaml
```

#### For Proxy Configurations

```yaml
# HeiShaMon proxy:
esp32: !include confs/esp32s3-heishamon.yaml

# Direct proxy:
esp32: !include confs/esp32s3-proxy.yaml
```

### What's Included in This BIN

| Feature | Included |
|---------|----------|
| Language | English |
| Zone 1 | ✅ Yes |
| Zone 2 | ✅ Yes |
| Sync Feature | ✅ Yes |
| Auto-Adaptive | ✅ Yes |
| Short Cycle Protection | ✅ Yes |
| Energy Monitoring | ✅ Yes |
| Service Codes | ✅ Yes |
| Web UI | ✅ Yes |
| OTA Updates | ✅ Yes |

### Build Troubleshooting

**Error: `http_request_component` not found**
- Cause: Missing ID in http_request configuration
- Fix: Verify `id: http_request_component` is present

**Error: Could not find substitution**
- Cause: Missing label file
- Fix: Verify `confs/ecodan-labels-en.yaml` exists

**Error: Sensor `z1_room_temp` not found**
- Cause: Zone package not included
- Fix: Verify `zone1: !include confs/zone1.yaml` is present

**Build runs out of memory**
- Solution: Add `--no-logs` flag
  ```bash
  esphome compile ecodan-en-2zones-sync.yaml --no-logs
  ```

**Build takes very long**
- First build: Normal (downloads all dependencies)
- Subsequent builds should be faster
- Use `--no-logs` to reduce memory usage

### Building Multiple Language Variants

To create BIN files for different languages, duplicate your build file and change the labels line:

```yaml
# English
packages:
  labels: !include confs/ecodan-labels-en.yaml

# Dutch
packages:
  labels: !include confs/ecodan-labels-nl.yaml

# German
packages:
  labels: !include confs/ecodan-labels-de.yaml
```

Available languages: `en`, `nl`, `de`, `fr`, `it`, `es`, `fi`, `no`, `sv`, `da`, `pl`

---

## Related Documentation

- [Remote Thermostat REST API Documentation](https://github.com/gekkekoe/esphome-ecodan-remote-thermostat/blob/main/docs/update-from-rest.md)
- [Main Project README](../README.md)
- [Hardware Setup Guide](hardware.md)
- [Build from Source Guide](build-from-source.md)
- [Remote Thermostat Home Assistant Integration](https://github.com/gekkekoe/esphome-ecodan-remote-thermostat/blob/main/docs/update-from-ha.md)
- [CN105 Heat Pump Documentation](https://github.com/Zjemm/esphome-ecodan-hp)
- [ESPHome HTTP Request Component](https://esphome.io/components/http_request.html)

## FAQ

**Q: Can I sync to Home Assistant instead of a remote thermostat?**
A: Yes! You can use Home Assistant automations to read the temperature sensors and send them to the remote thermostat. However, this configuration provides direct ESP-to-ESP communication without requiring Home Assistant.

**Q: What happens if the remote thermostat is offline?**
A: The HTTP POST will timeout (default 3 seconds) and an error will be logged. The sync will retry at the next interval.

**Q: Can I sync both zones to different remote thermostats?**
A: Yes! Create two instances of the sync configuration with different IDs and configure each for a different zone and thermostat IP.

**Q: Does this work with proxy mode?**
A: Yes, as long as the CN105 ESP can read the room temperature sensors. Proxy mode only affects the heat pump communication, not the temperature reading.

**Q: Will this affect my heat pump warranty?**
A: This solution only reads temperature data and sends it to another ESP. It does not modify heat pump operation or protocols.
