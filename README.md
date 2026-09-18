# Interactive Weather UI

Weather detector firmware for an **ESP32-S3**. The device connects to Wi-Fi, downloads a seven-day forecast from the [Open-Meteo API](https://open-meteo.com/), stores the JSON response in SPIFFS, and is designed to display weather information on an ST7735S TFT screen.

> **Current status:** The Wi-Fi, HTTPS, JSON parsing, and SPIFFS cache paths are implemented. The forecast-to-TFT display flow is still incomplete: `main.c` currently fetches and logs the forecast, but does not yet render it on the screen.

## Features

- ESP32-S3 firmware built with ESP-IDF and FreeRTOS
- Wi-Fi station-mode connection with retry handling
- HTTPS request to Open-Meteo
- Seven-day forecast parsing with cJSON
- Forecast cache stored at `/spiffs/forecast.json`
- ST7735S TFT display driver over SPI
- Weather icons for sunny, cloudy, and rainy conditions
- Arduino/mock display abstraction in `tft_display.c`

## Hardware

- ESP32-S3 development board
- ST7735S 128x160 TFT display
- 2.4 GHz Wi-Fi network
- USB cable for flashing and serial logs

### TFT wiring

The low-level TFT driver defines these pins in `components/tft/lcd.h`:

| TFT signal | ESP32-S3 GPIO |
|---|---:|
| CS | 11 |
| SCLK | 10 |
| MOSI | 9 |
| DC/A0 | 8 |
| RST | 18 |
| LED | 17 |

Check your board and display datasheets before wiring. Some TFT modules use different pin labels or voltage requirements.

## Software requirements

- ESP-IDF with the ESP32-S3 toolchain
- Python, as required by ESP-IDF
- USB drivers for your ESP32-S3 board
- A 2.4 GHz Wi-Fi access point

The code uses ESP-IDF components including Wi-Fi, FreeRTOS, HTTPS/TLS, cJSON, SPI, GPIO, and SPIFFS.

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/hanklq/Interactive-Weather-UI.git
cd Interactive-Weather-UI
```

### 2. Configure Wi-Fi

Edit `Wifi.h` and replace the example credentials:

```c
#define WIFI_SSID "your-wifi-name"
#define WIFI_PASS "your-wifi-password"
```

Use a 2.4 GHz network. Do not commit real passwords to a public repository; use a secure configuration method for production hardware.

### 3. Configure the ESP32-S3 target

From an ESP-IDF terminal:

```bash
idf.py set-target esp32s3
idf.py menuconfig
```

Enable the ESP-TLS certificate bundle if it is not already enabled. HTTPS uses `esp_crt_bundle_attach` in `http.c`.

### 4. Configure the SPIFFS partition

`spiffs_store.c` expects a partition with the label `storage`:

```c
.partition_label = "storage"
```

Add or select a partition table containing a SPIFFS partition named `storage`. For example:

```csv
# Name,     Type, SubType, Offset,  Size
nvs,        data, nvs,     0x9000,  0x6000
phy_init,   data, phy,    0xf000,  0x1000
factory,    app,  factory, 0x10000, 1M
storage,    data, spiffs,           0xF0000
```

Adjust the offsets and sizes to match your board's flash capacity. The current repository does not include a partition table, so this step may be required before building.

### 5. Build, flash, and monitor

```bash
idf.py build
idf.py -p PORT flash monitor
```

Examples:

```bash
# Linux/macOS
idf.py -p /dev/ttyUSB0 flash monitor

# Windows
idf.py -p COM5 flash monitor
```

Exit the serial monitor with `Ctrl-]`.

## Weather API

The default endpoint is defined in `http.h`:

```text
https://api.open-meteo.com/v1/forecast?latitude=21.0&longitude=105.75&daily=temperature_2m_max,precipitation_probability_mean,weathercode&timezone=auto
```

The coordinates target the Hanoi area. Change `latitude` and `longitude` in `WEATHER_API_URL` to use another location.

The parser expects these arrays under the API response's `daily` object:

- `time`
- `temperature_2m_max`
- `precipitation_probability_mean`
- `weathercode`

## Project structure

```text
main.c                 Application entry point and forecast task
Wifi.c / Wifi.h        Wi-Fi initialization and retry handling
http.c / http.h        HTTPS request and Open-Meteo JSON parsing
spiffs_store.c/.h      SPIFFS mount and forecast.json cache
CMakeLists.txt         ESP-IDF source registration
tft_display.c/.h       TFT rendering abstraction
tft_icons.c/.h         Weather icon functions
components/tft/        ST7735S driver, drawing utilities, fonts, and images
```

### Runtime flow

1. `app_main()` starts in `main.c`.
2. `wifi_init_sta()` initializes NVS, networking, and Wi-Fi.
3. SPIFFS is mounted and `/spiffs/forecast.json` is read if it exists.
4. A FreeRTOS task calls `get_weather_forecast()`.
5. `http.c` downloads and parses the forecast, then saves the response to SPIFFS.
6. Forecast values are logged. TFT rendering integration remains a TODO.

## Common errors and solutions

### `idf.py: command not found`

ESP-IDF is not installed or its environment is not active. Open an ESP-IDF terminal or run the ESP-IDF export script, then verify:

```bash
idf.py --version
```

### CMake cannot find `project()`

The repository currently contains a component-style `CMakeLists.txt`, but not a complete top-level ESP-IDF project file. A normal root `CMakeLists.txt` should include:

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(weather_detector)
```

The existing `idf_component_register(...)` file should normally be placed in the application component directory, such as `main/CMakeLists.txt`.

### `cJSON.h: No such file or directory`

Declare cJSON as an ESP-IDF component dependency. Depending on the ESP-IDF version, add a component manifest or include `cjson` in the component's `REQUIRES` list.

### Wi-Fi connection failures

Check the SSID and password in `Wifi.h`, use a 2.4 GHz network, and inspect the serial monitor for the disconnect reason. The current Wi-Fi configuration uses `WIFI_AUTH_OPEN`; for a WPA2 network, change the authentication threshold to an appropriate WPA2 setting or remove that override.

The startup code waits indefinitely for the Wi-Fi event. If the access point is unavailable, the application may appear to hang. Add a timeout if cached/offline operation is required.

### `SPIFFS mount failed` or `storage partition not found`

Create/select a partition table with a data partition labeled `storage`. The application mounts that partition and stores the forecast at `/spiffs/forecast.json`.

### HTTPS or certificate errors

Enable the ESP-TLS certificate bundle in `idf.py menuconfig`. Also verify that the board has a valid system time and internet access. Test the endpoint from a computer:

```bash
curl -I "https://api.open-meteo.com/v1/forecast?latitude=21.0&longitude=105.75&daily=temperature_2m_max,precipitation_probability_mean,weathercode&timezone=auto"
```

### `Failed to parse JSON` or `Missing 'daily' object`

The response may be empty, an API error, or a changed response format. Check the HTTP status code and inspect the returned body. Confirm that the expected `daily` arrays are present.

### TFT screen is blank

The current `main.c` does not call `tft_init()` or `tft_render_day()`, and the TFT sources under `components/tft/` may need to be added to the ESP-IDF component build. Verify the wiring, ST7735S model, GPIO definitions, reset sequence, and display orientation.

### Arduino library errors

The Arduino implementation in `tft_display.c` is compiled only when `ARDUINO` is defined. For a normal ESP-IDF build, use the mock backend or integrate the low-level ESP-IDF TFT driver. If using Arduino, install Adafruit GFX and Adafruit ST7735 and configure the project accordingly.

## Known limitations and next steps

- Add the missing complete ESP-IDF project structure and partition table.
- Add the TFT component sources to the build system.
- Call `tft_init()` during startup.
- Convert parsed Open-Meteo data into `tft_day_t` values and call `tft_render_day()`.
- Add periodic forecast refresh instead of fetching only once.
- Add a timeout and offline fallback for Wi-Fi.
- Move Wi-Fi credentials out of source code.
- Check HTTP status before parsing the response.
- Add on-screen error states for Wi-Fi, HTTP, JSON, and SPIFFS failures.
