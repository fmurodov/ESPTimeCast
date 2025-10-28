# ESPHome LED Matrix Clock & Weather Display

A simple WiFi-connected LED matrix clock and weather display built with ESPHome. This project uses an ESP32-S2 and MAX7219 LED matrix modules to show the current time, day of the week, and local weather information.

**Based on [ESPTimeCast](https://github.com/mfactory-osaka/ESPTimeCast) by [@mfactory-osaka](https://github.com/mfactory-osaka)** - A comprehensive Arduino-based version with web interface and advanced features. This is a simplified ESPHome implementation for easier Home Assistant integration.

## Why ESPHome Version?

If you're already using Home Assistant and prefer:
- YAML-based configuration over web UI
- Native Home Assistant API integration
- Simpler codebase focused on core features
- Easy OTA updates through ESPHome

Then this version might be for you. For a more feature-rich standalone version with web configuration, check out the [original ESPTimeCast project](https://github.com/mfactory-osaka/ESPTimeCast).

## Features

- **Time Display**: Shows current time with weekday indicator
- **Weather Information**: Displays temperature, humidity, and rain status from OpenWeatherMap
- **Automatic Brightness**: Adjusts display brightness based on time of day
- **Easy Configuration**: All settings managed through ESPHome YAML configuration
- **Over-the-Air Updates**: Update firmware wirelessly via ESPHome
- **Home Assistant Integration**: Native API integration for monitoring and control

## Hardware Requirements

### Components
- ESP32-S2 Mini development board (or compatible ESP32/ESP8266)
- MAX7219 LED Matrix modules (4x 8x8 matrices for 32x8 display)
- 5V USB power supply

### Wiring

Connect the MAX7219 to your development board:

**Board → MAX7219**

| D1 Mini (Micro USB) | D1 Mini (USB C) | D1 Mini (ESP32) | S2 Mini | MAX7219 |
|:-------------------:|:---------------:|:---------------:|:-------:|:-------:|
| GND                 | GND             | GND             | GND     | GND     |
| 5V                  | 5V/VBUS         | 5V/VBUS         | 5V/VBUS | VCC     |
| D5                  | 14              | 18              | 7       | CLK     |
| D7                  | 13              | 23              | 11      | CS      |
| D8                  | 15              | 5               | 12      | DIN     |

**Important hardware notes:**
- Always double-check that VCC (5V), GND, and DIN/CS/CLK match your MAX7219 module's pin order — different modules sometimes label them differently
- Power the MAX7219 from the **5V USB rail**, not the 3.3V regulator, to ensure stable operation and sufficient brightness
- The MAX7219 works fine with 3.3V logic signals from the ESP (no need for level shifters)
- Pins in the default ESPHome config match the S2 Mini column - adjust in YAML if using different board

### 3D Printable Case

Want to give your display a home? The original ESPTimeCast project offers 3D printable cases:

- [Download from Printables](https://www.printables.com/model/1344276-esptimecast-wi-fi-clock-weather-display)
- [Download from Cults3D](https://cults3d.com/en/3d-model/gadget/wifi-connected-led-matrix-clock-and-weather-station-esp8266-and-max7219)

The case front panel (3mm) can be laser cut for a professional look!

## Software Setup

### Prerequisites
- [ESPHome](https://esphome.io/) installed (via Home Assistant add-on or standalone)
- [OpenWeatherMap API key](https://openweathermap.org/api) (free tier available - sign up [here](https://home.openweathermap.org/users/sign_up))
- Home Assistant (optional, but recommended for API integration)

### Configuration

1. **Create a secrets file** (`secrets.yaml`) in your ESPHome directory:

```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"
wifi_ap_password: "FallbackAPPassword"
openweather_api_key: "your_openweathermap_api_key"
api_encryption_key: "your_random_32_char_key"
ota_password: "your_ota_password"
```

2. **Copy the configuration** (`ESPTimeDisplay.yaml`) to your ESPHome directory

3. **Customize substitutions** at the top of the YAML file:

```yaml
substitutions:
  device: "esptime-1"                    # Unique device name
  name: "ESPTime Display - Living Room"  # Friendly name

  # Hardware pins (adjust if not using S2 Mini)
  pin_clk: "9"
  pin_cs: "11"
  pin_mosi: "12"

  # Display settings
  brightness_day: "5"                    # 0-15, brightness during day
  brightness_night: "0"                  # 0-15, brightness at night
  flip_display: "false"                  # "true" to rotate 180°
  time_format_24h: "true"                # "true" for 24h, "false" for 12h

  # WiFi (uses secrets.yaml)
  local_domain: ".home"

  # Weather
  weather_city: "Amsterdam"              # City name or ZIP code (US)
  weather_country: "NL"                  # Two-letter country code
  weather_api_key: ""                    # Your OpenWeatherMap API key
  unit_system: "metric"                  # "metric" (Celsius) or "imperial" (Fahrenheit)

  # Locale
  timezone: "Europe/Amsterdam"           # IANA timezone
  weekdays: "Zon,Maa,Din,Woe,Don,Vri,Zat"  # Weekday abbreviations
```

**Weather Location Options:**
- **City name**: e.g., `weather_city: "Tokyo"` with `weather_country: "JP"`
- **ZIP code (US only)**: e.g., `weather_city: "10001"` with `weather_country: "US"`
- **Coordinates**: Use latitude in city field and longitude in country field

4. **Install to device**:

```bash
# First time (via USB)
esphome run ESPTimeDisplay.yaml

# After first install (OTA)
esphome run ESPTimeDisplay.yaml --device esptime-1.local
```

## Display Behavior

The display automatically alternates between two screens:
- **Clock (20 seconds)**: Shows weekday abbreviation and current time (HH:MM)
- **Weather (15 seconds)**: Shows temperature and humidity, or temperature with rain indicator (R)

### What You'll See

| Display Mode | Time Available | Weather Available | Display Output                    |
|:------------:|:--------------:|:-----------------:|:----------------------------------|
| **Clock**    | ✅ Yes         | —                 | `Din 14:53` or `Din 2:53`        |
| **Clock**    | ❌ No          | —                 | `SYNC...` (syncing NTP time)     |
| **Weather**  | —              | ✅ Yes            | `21.5°C 65%` (temp + humidity)   |
| **Weather**  | —              | ✅ Yes (raining)  | `21.5°C R` (temp + rain indicator)|
| **Weather**  | —              | ❌ No             | `NO DATA` (no weather fetched)   |

### Brightness Schedule

The display automatically adjusts brightness:
- **Day mode** (8:00 - 19:00): Higher brightness (configurable)
- **Night mode** (19:00 - 8:00): Lower brightness (configurable)

Customize the brightness levels in substitutions and times in the `on_time:` section:

```yaml
time:
  - platform: sntp
    on_time:
      - seconds: 0
        then:
          - lambda: |-
              int hour = id(time_sntp).now().hour;
              // Adjust these hours (19 and 8) for your schedule
              int brightness = (hour >= 19 || hour < 8) ? ${brightness_night} : ${brightness_day};
              id(display_matrix)->intensity(brightness);
```

## Weather Data

Weather information is fetched from OpenWeatherMap:
- Updates every 5 minutes automatically
- Shows temperature in Celsius or Fahrenheit (configurable)
- Shows relative humidity percentage
- Indicates rain conditions with "R" symbol

### Rain Detection

The rain indicator appears for these weather condition codes:
- **2xx**: Thunderstorms
- **3xx**: Drizzle
- **5xx**: Rain

### OpenWeatherMap Setup

1. [Sign up for a free account](https://home.openweathermap.org/users/sign_up)
2. [Get your API key](https://home.openweathermap.org/api_keys)
3. Add it to your `secrets.yaml` file
4. Free tier includes 60 calls/minute and 1,000,000 calls/month (plenty for this project)

## Customization

### Change Display Font

The configuration uses a custom matrix font optimized for 8-pixel height displays. You can replace it with any compatible TrueType font:

```yaml
font:
  - file: "https://github.com/your-repo/your-font.ttf"
    id: font_display
    size: 8
    glyphs: '!"#$%&()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ abcdefghijklmnopqrstuvwxyz{}|~°'
```

### Adjust Screen Timing

Modify how long each screen displays:

```yaml
display:
  - platform: max7219digit
    lambda: |-
      static int timer = 0;
      static int page = 0;
      timer++;

      // Change these durations (in display update cycles)
      // 100 cycles × 200ms = 20 seconds for clock
      // 75 cycles × 200ms = 15 seconds for weather
      int duration = (page == 0) ? 100 : 75;

      if (timer >= duration) {
        timer = 0;
        page = (page + 1) % 2;
      }
```

### Flip Display Orientation

Set `flip_display: "true"` in substitutions to rotate the display 180 degrees. Useful if your display is mounted upside down.

### Weekday Abbreviations

Customize the weekday names in your language (comma-separated, starting with Sunday):

```yaml
weekdays: "Sun,Mon,Tue,Wed,Thu,Fri,Sat"  # English
weekdays: "Dim,Lun,Mar,Mer,Jeu,Ven,Sam"  # French
weekdays: "So,Mo,Di,Mi,Do,Fr,Sa"         # German
```

### Weather Update Frequency

Change how often weather is fetched (default: every 5 minutes):

```yaml
time:
  - platform: sntp
    on_time:
      - seconds: 0
        then:
          - lambda: |-
              // Change the modulo value (5) to your desired interval
              if (id(time_sntp).now().minute % 10 == 0) {  // Every 10 minutes
                id(fetch_weather).execute();
              }
```

## Troubleshooting

### Display shows "SYNC..."
- The device is attempting to synchronize time via NTP
- **Check**: WiFi connection status
- **Check**: Timezone setting is a valid IANA timezone
- **Wait**: Initial sync can take 30-60 seconds

### Weather shows "NO DATA"
- **Check**: OpenWeatherMap API key is valid ([check here](https://home.openweathermap.org/api_keys))
- **Check**: City and country code are correct
- **Check**: Internet connectivity (ping test from device)
- **Check**: API call limit hasn't been exceeded (unlikely with free tier)
- **Check**: ESPHome logs for HTTP error messages

### Display too dim or too bright
- Adjust `brightness_day` and `brightness_night` values (0-15)
- Modify the brightness schedule hours in the `on_time:` section
- Check if MAX7219 is powered from 5V (not 3.3V)

### WiFi connection issues
- Device creates fallback AP: Connect to AP named with your device name
- Default AP password is in your secrets.yaml (`wifi_ap_password`)
- Check ESPHome logs for connection errors
- Verify WiFi credentials in secrets.yaml
- Ensure 2.4GHz WiFi is available (ESP32/ESP8266 don't support 5GHz)

### Display shows garbled characters
- Check wiring connections (especially DIN/CLK/CS)
- Verify `num_chips: 4` matches your matrix module count
- Try `flip_display: "true"` if display appears mirrored
- Check font glyphs include all characters you're trying to display

### Weather not updating
- Check logs for HTTP request errors
- Verify the script is being called (add logger.log statements)
- Test API URL manually in browser
- Ensure device has internet access (not just local network)

## Home Assistant Integration

Once configured and connected, the device will automatically appear in Home Assistant (if you have the ESPHome integration and matching API encryption key).

### What you can do:
- **Monitor status**: View connection and sensor states
- **View logs**: Remote log viewing via ESPHome dashboard
- **Manual weather updates**: Create automation to trigger weather fetch
- **Automation**: Trigger actions based on weather or time
- **Remote control**: Adjust brightness or settings via Home Assistant

### Example Automation

Trigger a weather update when you want:

```yaml
automation:
  - alias: "Refresh weather display on demand"
    trigger:
      - platform: state
        entity_id: input_boolean.refresh_weather
        to: "on"
    action:
      - service: esphome.esptime_1_fetch_weather
      - service: input_boolean.turn_off
        entity_id: input_boolean.refresh_weather
```

## Logging

View logs to debug issues:

```bash
# Via ESPHome CLI
esphome logs ESPTimeDisplay.yaml

# Or via Home Assistant
# Go to ESPHome dashboard → Click "LOGS" on your device
```

The configuration has minimal logging to reduce overhead:
- General log level: `WARN`
- Component errors: `ERROR`

Increase to `DEBUG` for detailed troubleshooting:

```yaml
logger:
  level: DEBUG
```

## Comparison with Original ESPTimeCast

| Feature | ESPTimeCast (Original) | ESPHome Version (This) |
|:--------|:----------------------|:----------------------|
| **Configuration** | Web UI | YAML file |
| **Platform** | Arduino IDE | ESPHome |
| **Integration** | Standalone / Manual | Home Assistant native |
| **Updates** | USB / OTA manual | ESPHome OTA |
| **Features** | Advanced (many options) | Simplified (core features) |
| **Setup Complexity** | Medium (web setup) | Easy (if familiar with ESPHome) |
| **Customization** | Web interface | Code editing |
| **Best For** | Standalone device | Home Assistant users |

**Consider the original ESPTimeCast if you want:**
- Web-based configuration without coding
- More display modes and options
- Date display mode
- Countdown timers
- Nightscout glucose monitoring
- Weather description display
- Standalone operation without Home Assistant

## Credits

This project is based on [ESPTimeCast](https://github.com/mfactory-osaka/ESPTimeCast) by [@mfactory-osaka](https://github.com/mfactory-osaka).

**Original project features:**
- Comprehensive Arduino implementation
- Built-in web interface for all settings
- Advanced display modes and customization
- Featured on [Hackaday](https://hackaday.com/2025/10/02/building-a-desk-display-for-time-and-weather-data) and [XDA Developers](https://www.xda-developers.com/super-sleek-esp32-weather-station)

If you prefer a feature-rich standalone version, please check out the original project!

## License

This ESPHome configuration is provided as-is for personal and educational use.

## Resources

- [Original ESPTimeCast Project](https://github.com/mfactory-osaka/ESPTimeCast)
- [ESPHome Documentation](https://esphome.io/)
- [ESPHome MAX7219 Display Component](https://esphome.io/components/display/max7219digit.html)
- [OpenWeatherMap API](https://openweathermap.org/api)
- [MAX7219 Datasheet](https://datasheets.maximintegrated.com/en/ds/MAX7219-MAX7221.pdf)
- [3D Printable Cases](https://www.printables.com/model/1344276-esptimecast-wi-fi-clock-weather-display)
