# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an ESPHome configuration for a reTerminal device with a 7.3" e-ink display (model: epaper_spi 7.3in-spectra-e6). The device is based on an ESP32-S3 running ESP-IDF framework and integrates with Home Assistant for smart home functionality.

## Hardware Configuration

- **Board**: ESP32-S3-DevKitC-1
- **Display**: 7.3" E-Ink (Spectra E6) with red/black/blue color support
- **Sensors**:
  - SHT40 temperature/humidity sensor (I2C: SDA=GPIO19, SCL=GPIO20)
  - ADC battery voltage monitoring (GPIO1)
  - PCF8563 RTC (I2C address 0x51)
- **Buttons**: 3 physical buttons (GPIO3, GPIO5, GPIO6) for user interaction
- **Features**: Active buzzer (GPIO45), microSD card support (GPIO15/16), WiFi, Home Assistant API

## Development Commands

### Build and Deploy
```bash
# Compile the configuration (requires ESPHome installed)
esphome compile reterminal.yaml

# Upload to device via WiFi (OTA)
esphome upload reterminal.yaml --device <IP_ADDRESS>

# Upload via USB
esphome upload reterminal.yaml

# View logs
esphome logs reterminal.yaml
```

### Validation
```bash
# Validate configuration syntax
esphome config reterminal.yaml
```

## Architecture Notes

### Power Management
Deep sleep is currently **disabled** to allow OTA updates. When enabled, the device:
- Runs for 5 minutes after wake
- Sleeps for 10 minutes
- Wakes on button press (GPIO3/5/6) or timer expiry

### Display Update Strategy
The e-ink display updates every 3 minutes automatically and can be manually refreshed via Button 1. The display uses a custom layout with:
- **Badge area** (top 20%): System status header
- **Left column** (50% width): Date/time, battery status, guest WiFi QR code
- **Right column** (50% width): Service status details

### Sensor Data Flow
1. Local sensors (SHT40, battery ADC) update every 60s
2. Template sensors calculate derived values (battery %, runtime estimates)
3. Home Assistant API provides time sync via `homeassistant_time` platform
4. Hardware RTC (PCF8563) serves as fallback when WiFi is unavailable

### Button Functionality
- **Button 1 (GPIO3)**: Refresh display + reload QR code from online source
- **Button 2 (GPIO6)**: Force sensor updates
- **Button 3 (GPIO5)**: Log system info (memory, uptime, temperature)

### Color System
The display supports 3 colors defined in the `color:` section:
- `display_red`: Primary borders and accents
- `display_black`: Main text
- `display_blue`: Secondary text and highlights

### Bluetooth Proxy
Currently **disabled** due to potential ESP32-S3 bootloop issues. Can be re-enabled by uncommenting the `esp32_ble_tracker` and `bluetooth_proxy` sections.

## Important Configuration Details

### Secrets Required
Create a `secrets.yaml` file in the same directory with:
```yaml
wifi_ssid: "your_ssid"
wifi_password: "your_password"
```

### API Encryption
The Home Assistant API uses encryption. The key is embedded in the config (line 42). If rotating keys, update both the YAML and Home Assistant integration.

### Online Image
The QR code is fetched from a URL (line 79) with a security token. This image is:
- Downloaded on boot and button 1 press
- Cached locally (update_interval: never)
- Resized to 200x200 pixels
- Must be PNG format, binary type

### GPIO Strapping Warnings
Several GPIOs have strapping warnings suppressed (GPIO3, GPIO45). These pins have special boot-time functions on ESP32-S3 but are safe to use with `ignore_strapping_warning: true` once the device has booted.

## MCP Configuration for Claude

To enable Claude Code to interact with your Home Assistant instance, configure the Model Context Protocol (MCP) server in your Claude Desktop configuration file:

**Location**:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

**Configuration**:
```json
{
  "mcpServers": {
    "Home Assistant": {
      "command": "mcp-proxy",
      "args": [
        "--transport=streamablehttp",
        "--stateless",
        "http://localhost:8123/api/mcp"
      ],
      "env": {
        "API_ACCESS_TOKEN": "<your_access_token_here>"
      }
    }
  }
}
```

**Setup steps**:
1. Install `mcp-proxy` if not already installed
2. Generate a long-lived access token in Home Assistant (Settings → People → Your User → Long-Lived Access Tokens)
3. Replace `<your_access_token_here>` with your actual token
4. Update the URL if your Home Assistant runs on a different host/port
5. Restart Claude Desktop to load the configuration

Once configured, Claude Code can query Home Assistant entities, check device states, and help debug integration issues directly.

## Common Development Tasks

### Adding New Sensors
1. Add sensor definition under `sensor:` or `text_sensor:` section
2. Update display lambda in `display:` section (line 544+) to render the new data
3. Consider adding to button actions if interactive updates are needed

### Modifying Display Layout
The display rendering uses a custom lambda (line 544-652). Key functions:
- `draw_bordered_area()`: Draws red bordered boxes with optional titles
- Layout uses pixel-based positioning with constants for MARGIN and SPACING
- Text alignment: Use `TextAlign::CENTER` for centered text
- Fonts: `inter_large` (36px), `inter_medium` (24px), `inter_small` (18px)

### Changing Update Intervals
- Display: `update_interval: 3min` (line 541)
- Sensors: Individual `update_interval` parameters (typically 60s)
- Auto-boot refresh: 30s interval checks if uptime < 60s (line 514)

### Battery Voltage Calibration
If battery percentage readings are inaccurate, adjust the voltage divider multiplier at line 178:
```cpp
voltage = voltage * 2.0; // Adjust multiplier based on actual hardware
```

Also adjust the voltage-to-percentage mapping (lines 180-184) if using different battery chemistry.

## Troubleshooting

### Device Won't Boot
- Check if deep sleep is enabled (currently disabled)
- Bluetooth proxy can cause bootloops on ESP32-S3 - verify it's commented out

### Display Not Updating
- Check `busy_pin` status (GPIO13, inverted)
- Verify SPI pins are correct (CLK=GPIO7, MOSI=GPIO9, MISO=GPIO8)
- Review logs for e-ink driver errors

### QR Code Not Loading
- Verify network connectivity and URL accessibility
- Check token validity in the image URL (line 79)
- Look for "QR Code erfolgreich heruntergeladen" or error logs

### Time Not Syncing
- Primary: Check Home Assistant connection (`ha_connection` sensor)
- Fallback: Verify I2C connection to PCF8563 RTC (address 0x51)
- Check that `homeassistant_time` platform receives sync events
