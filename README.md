# Leapmotor to Home Assistant Integration

This project provides a MacroDroid-based integration to sync Leapmotor vehicle data to Home Assistant. Since Leapmotor doesn't provide an official API, this solution uses screen scraping via MacroDroid on Android to extract vehicle information and push it to Home Assistant via the REST API.

## Overview

The integration consists of three main components:

1. **MacroDroid Macro** - Runs on your Android device to scrape the Leapmotor app and send data to Home Assistant
2. **Home Assistant Sensors** - Template sensors that extract attributes from the main sensor
3. **Home Assistant Dashboard** - A pre-configured Lovelace dashboard to display vehicle information

## Features

The integration tracks the following vehicle data:

- Vehicle state (parked, driving, etc.)
- Battery percentage
- Lock state (locked/unlocked)
- Remaining range (km)
- Interior temperature
- Exterior temperature

## Leapmotor B10 Support

A B10-specific macro (`Leapmotor_to_HomeAssistant_B10.macro`) and sensor configuration (`sensor_b10.yaml`) are available with extended functionality.

### Additional Data Points

Compared to the C10 integration, the B10 macro additionally tracks:

* Odometer (total km driven)
* Charge limit (%)
* Charge current (A)
* Charge power (kW)
* Charge remaining time
* Charge status (A = not connected, B = connected/not charging, C = charging)

### Charge Status Logic

The B10 macro distinguishes three charging states and sends them to a separate sensor (`sensor.leapmotor_b10_charge_status`):

* **A** – Not connected
* **B** – Connected, not charging
* **C** – Actively charging (charge current, power and remaining time are read from the app)

Charge limit is always read and transmitted regardless of charge status.

The A/B/C status is designed to be compatible with [evcc](https://evcc.io/) for smart charging control.

> **Note:** Charge remaining time (`charge_remaining`) is not yet fully implemented — it is transmitted as `0` for status A and B. Live readout during active charging (status C) is pending real-world testing.

### Trigger & Update Frequency

The B10 macro runs on two triggers:

* Screen turns off
* Every 5 minutes (regular interval)

The minimum interval between runs is ~5 minutes (constraint: 285 seconds).

### Setup

Follow the same installation steps as for the C10, with these differences:

* Import `Leapmotor_to_HomeAssistant_B10.macro` instead
* Configure the local variables as described in Step 2.2:
  * **`ha_url`** (Secure): Your Home Assistant URL
  * **`ha_token`** (Secure): Your Long-Lived Access Token
  * **`ha_sensor_name`**: Set to `leapmotor_b10_1`
* Add the contents of `sensor_b10.yaml` to your template sensor configuration
* The main sensor will be `sensor.leapmotor_b10_1`
* The charge status sensor will be `sensor.leapmotor_b10_charge_status`

## Prerequisites

### Required Software

- **Android Device** with [MacroDroid](https://play.google.com/store/apps/details?id=com.arlosoft.macrodroid) installed
- **Leapmotor App** installed and logged in on the same Android device
- **Home Assistant** instance (local or cloud accessible)

### Required Home Assistant Add-ons/Integrations

- [Custom Layout Card](https://github.com/thomasloven/lovelace-layout-card) (HACS)
- [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom) (HACS)
- [Mini Graph Card](https://github.com/kalkih/mini-graph-card) (HACS)
- [card-mod](https://github.com/thomasloven/lovelace-card-mod) (HACS)

## Installation

### Step 1: Set Up Home Assistant

#### 1.1 Create Template Sensors

Add the contents of `sensor.yaml` to your Home Assistant configuration.

**Option A: If using `configuration.yaml` directly:**

```yaml
# Add to configuration.yaml
template: !include sensor.yaml
```

Then place `sensor.yaml` in your Home Assistant config directory.

**Option B: If you already have template sensors:**

Copy the sensor definitions from `sensor.yaml` and merge them into your existing template configuration.

#### 1.2 Create a Long-Lived Access Token

1. In Home Assistant, click your profile (bottom left)
2. Scroll to "Long-Lived Access Tokens"
3. Click "Create Token"
4. Name it "MacroDroid Leapmotor"
5. Copy the token and save it securely (you'll need it for MacroDroid setup)

#### 1.3 Restart Home Assistant

After adding the sensor configuration, restart Home Assistant to load the new sensors.

### Step 2: Configure MacroDroid

#### 2.1 Import the Macro

1. Download `Leapmotor_to_HomeAssistant.macro` to your Android device
2. Open MacroDroid
3. Tap the menu (three dots) → "Import"
4. Select the downloaded `.macro` file

#### 2.2 Configure Variables

After importing, you need to configure three local variables:

1. Tap the macro to edit it
2. Tap "Variables" (at the bottom)
3. Configure the following **local variables**:

   - **`ha_url`** (Secure): Your Home Assistant URL

     - Examples:
       - Local: `http://homeassistant.local:8123`
       - Nabu Casa: `https://abcdefgh.ui.nabu.casa`
       - Custom domain: `https://home.yourdomain.com`
     - ⚠️ **Do not** include trailing slash

   - **`ha_token`** (Secure): The Long-Lived Access Token from Step 1.2

   - **`ha_sensor_name`** (not secure): The sensor entity ID suffix
     - Default: `leapmotor_c10_1`
     - This creates: `sensor.leapmotor_c10_1`
     - Change if you have multiple vehicles or prefer a different name

#### 2.3 Test the Macro

1. Make sure your device is unlocked and the Leapmotor app is logged in
2. In MacroDroid, tap the macro
3. Tap "Test Actions" (not "Test Macro")
4. Observe the macro running:

   - It will open the Leapmotor app
   - Wait for the home screen to load
   - Navigate to the energy/battery details
   - Scrape the data
   - Send it to Home Assistant
   - Return to your previous app

5. Check Home Assistant Developer Tools → States to verify `sensor.leapmotor_c10_1` was created

### Step 3: Add Dashboard

#### 3.1 Create a New Dashboard View

1. In Home Assistant, go to your dashboard
2. Click the three dots → "Edit Dashboard"
3. Click "Add View"
4. Switch to "YAML mode"
5. Paste the contents of `dashboard.yaml`
6. Save

#### 3.2 Adjust Sensor Names (if needed)

If you changed `ha_sensor_name` in Step 2.2, you'll need to update the dashboard:

- Replace all instances of `sensor.leapmotor_c10_1` with your custom sensor name
- Replace all instances of derived sensors accordingly:
  - `sensor.leapmotor_c10_battery_1`
  - `sensor.leapmotor_c10_range_1`
  - `sensor.leapmotor_c10_car_temperature_1`
  - `sensor.leapmotor_c10_air_temperature_1`
  - `sensor.leapmotor_c10_lock_state_1`

## How It Works

### MacroDroid Automation

**Trigger:**

- Screen turns off (to avoid interrupting active use)

**Constraints:**

- Device is unlocked
- Macro hasn't run in the last 30 minutes

**Actions:**

1. Saves the current foreground app
2. Launches the Leapmotor app
3. Waits for the "energy" screen to load (loops up to 20 times)
4. Reads screen contents and extracts:
   - Air temperature
   - Car temperature
   - Lock state
   - Remaining range
   - Car state
5. Clicks on the battery indicator to view battery percentage
6. Waits for battery screen to load (loops up to 20 times)
7. Extracts battery percentage
8. Constructs JSON payload with all data
9. Sends POST request to Home Assistant REST API
10. Kills the Leapmotor app
11. Returns to the original app

### Data Flow

```
Leapmotor App (Android)
         ↓ (screen scraping)
   MacroDroid Macro
         ↓ (HTTP POST)
Home Assistant REST API
         ↓ (creates/updates)
sensor.leapmotor_c10_1
         ↓ (attributes extracted)
Template Sensors (battery, range, temps, lock state)
         ↓ (displayed on)
   Lovelace Dashboard
```

## Data Structure

The main sensor (`sensor.leapmotor_c10_1`) stores data in this format:

```json
{
  "state": "parked",
  "attributes": {
    "battery_percent": 85,
    "air_temp": 12,
    "car_temp": 18,
    "lock_state": "locked",
    "remaining_range": 340,
    "friendly_name": "Leapmotor C10",
    "icon": "mdi:car-electric"
  }
}
```

## Troubleshooting

### Macro doesn't run automatically

- Check that the device is unlocked when the screen turns off
- Verify the "Last Run Time" constraint (30 minutes) hasn't blocked it
- Check MacroDroid logs: Menu → Log → Macro Log

### "Connection refused" or HTTP errors

- Verify `ha_url` is correct and accessible from your Android device
- Test by opening the URL in a browser on the same device
- Check that the access token is valid in Home Assistant
- Ensure Home Assistant is not behind a firewall blocking your device

### Sensors show "unknown" or "unavailable"

- Check that the macro has run at least once successfully
- Verify sensor names in `sensor.yaml` match the main sensor ID
- Check Home Assistant logs for template errors
- Ensure the main sensor exists: Developer Tools → States

### Macro fails to scrape data

- The Leapmotor app may have updated its UI elements
- Check the macro logs to see where it fails
- You may need to update the UI element IDs in the macro
- Ensure the Leapmotor app is logged in and working

### Dashboard cards not loading

- Install all required HACS integrations (see Prerequisites)
- Clear browser cache
- Check browser console for errors

## Customization

### Update Frequency

The macro runs when:

- Screen turns off
- Device is unlocked
- At least 30 minutes have passed since last run

To change the frequency:

1. Edit the macro
2. Find "Constraint: Last Run Time"
3. Adjust the time period

### Add More Sensors

If you want to track additional data from the Leapmotor app:

1. Edit the macro
2. Add screen scraping actions for the new data points
3. Update the JSON payload in the "Set Variable: ha_request_body" action
4. Add corresponding template sensors in `sensor.yaml`
5. Update the dashboard to display the new sensors

### Dashboard Styling

The dashboard uses [card-mod](https://github.com/thomasloven/lovelace-card-mod) for styling. Adjust:

- Card widths (default: 162px)
- Colors (in `icon_color` templates)
- Thresholds (battery warnings, range warnings, etc.)

## Security Considerations

### Sensitive Data

The macro handles sensitive information:

- **Home Assistant access token** - Stored securely in MacroDroid
- **Home Assistant URL** - Stored securely in MacroDroid

**Important:**

- Both are marked as "secure" variables in MacroDroid
- They are encrypted in the MacroDroid database
- The exported `.macro` file contains empty placeholders only
- Never share your configured macro file publicly
- Regenerate your access token if compromised

### Network Security

- Use HTTPS for your Home Assistant URL when possible
- Consider using Nabu Casa for secure remote access
- If using local HTTP, ensure your WiFi network is secure
- The access token grants full API access - treat it like a password

## Known Limitations

- Requires Android device with MacroDroid
- Only updates when device screen turns off
- Requires Leapmotor app to be installed and logged in
- Screen scraping is fragile and may break with app updates
- No real-time updates (updates every 30+ minutes)
- Device must be unlocked for macro to run
- Cannot control vehicle functions (read-only integration)

## Future Improvements

Potential enhancements:

- Support for multiple Leapmotor vehicles
- More frequent updates (with battery optimization)
- Additional data points (odometer, tire pressure, etc.)
- Notifications for low battery or unlock events
- Historical data tracking and charts

## Credits

- Created for Leapmotor C10 owners who want Home Assistant integration
- Uses MacroDroid for Android automation
- Dashboard built with Mushroom Cards and Mini Graph Card

## License

This project is provided as-is for personal use. Leapmotor is a trademark of their respective owners. This is an unofficial integration and is not affiliated with or endorsed by Leapmotor.

## Support

If you encounter issues:

1. Check the Troubleshooting section above
2. Review MacroDroid logs
3. Check Home Assistant logs
4. Verify all prerequisites are installed

## Changelog

### Version 1.0

- Initial release
- Support for Leapmotor C10
- Basic vehicle data tracking
- Home Assistant sensors and dashboard
