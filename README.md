# ESP32-C6-LCD-1.47 air quality companion

An [ESPHome](https://esphome.io) firmware that turns the [Waveshare ESP32-C6-LCD-1.47](https://www.waveshare.com/wiki/ESP32-C6-LCD-1.47) into a small animated companion for [Home Assistant](https://www.home-assistant.io). A little character breathes, blinks and looks around; its mood follows the air quality reported by your Home Assistant sensors, and the 4 values are shown below it.

| Air | Character | Message (default) |
|---|---|---|
| CO2 < 800 ppm and PM2.5 < 15 µg/m³ | Mint, smiling, pink cheeks, slow breathing | "Tout va bien" |
| CO2 < 1200 ppm and PM2.5 < 35 µg/m³ | Mustard, flat mouth, faster breathing | "L'air s'alourdit" |
| Above | Coral, dizzy eyes, gasping, sweating | "Aère vite !" |
| No data yet | Grey, sleeping | "J'attends HA..." |

It also shivers and turns blue below 18 °C, and sweats above 26 °C.

The default texts are French; every text is a substitution at the top of `compagnon.yaml`.

## Requirements

- A Waveshare ESP32-C6-LCD-1.47 (the non-touch version) and a USB-C data cable (some cables are charge-only: the board then powers on but never shows up in `lsusb`).
- Home Assistant with sensors for CO2, PM2.5, temperature and humidity. The defaults match an IKEA ALPSTUGA air quality monitor in a French Home Assistant setup.
- [uv](https://docs.astral.sh/uv/), to run ESPHome and esptool without installing them (or install ESPHome any other way). Tested with ESPHome 2026.11.0.

## Setup

1. Edit the substitutions at the top of `compagnon.yaml`: your sensor entity IDs, the texts, and `flash_size` (see [Flash size](#flash-size)).
2. Create `secrets.yaml` next to it (it is git-ignored):

   ```yaml
   wifi_ssid: "your-ssid"
   wifi_password: "your-wifi-password"
   api_encryption_key: "output of: openssl rand -base64 32"
   ```

3. Plug the board and flash it over USB (on Linux, your user must be in the `dialout` group):

   ```sh
   uvx esphome run compagnon.yaml --device /dev/ttyACM0
   ```

4. In Home Assistant, add the discovered "Compagnon" ESPHome device (Settings, Devices & services) with the same encryption key. The character wakes up as soon as Home Assistant sends the sensor values.

Next updates go over Wi-Fi:

```sh
uvx esphome run compagnon.yaml --device OTA
```

Home Assistant also gets 2 settings: the screen brightness (100% means 50% of the panel's maximum, see below) and the brightness of the onboard RGB LED, which glows with the mood color (green, amber, red; off by default). They are numbers, not lights, on purpose: an automation that switches off all the lights of a room must not switch the screen off.

## How it works

- The display uses the ESPHome [`mipi_spi`](https://esphome.io/components/display/mipi_spi/) platform, which ships a ready-made `WAVESHARE-ESP32-C6-LCD-1.47` model (pins, 34 px offset, color inversion, 40 MHz SPI).
- The character is drawn in code with circles, lines and triangles: there are no images. All movements are sine waves of the animation time; blinking, shivering and sweat drops are based on the frame number.
- An `interval` advances a global frame counter every 80 ms and redraws the screen. Without PSRAM, `mipi_spi` uses a 1/4 frame buffer (27 KB) and calls the drawing lambda once per slice, so the animation time comes from that global rather than from the clock: every slice draws the same frame.

## Board notes

### Chip

- ESP32-C6 with a 32-bit RISC-V HP core up to 160 MHz and a RISC-V LP core up to 20 MHz, 320 KB ROM, 512 KB HP SRAM, 16 KB LP SRAM, no PSRAM.
- Radios: 2.4 GHz Wi-Fi 6, Bluetooth 5 LE, IEEE 802.15.4 (Zigbee, Thread). Onboard ceramic antenna.
- USB: native USB-Serial/JTAG (`303a:1001`), seen as `/dev/ttyACM0` on Linux.

### Flash size

The docs announce an ESP32-C6FH4 with 4 MB of embedded flash, but some units ship with an ESP32-C6FH8 with 8 MB. Check yours:

```sh
uvx --from esptool esptool --port /dev/ttyACM0 chip-id
```

A 4 MB build runs on both; set `flash_size: 8MB` to use the whole flash of an FH8. Changing the flash size changes the partition table, so flash over USB after changing it, not over Wi-Fi.

### Onboard hardware and pinout

| Peripheral | Details | GPIO |
|---|---|---|
| LCD | 1.47 inch TFT, 172x320, 262K colors, ST7789 controller, SPI, not touch | MOSI 6, SCLK 7, CS 14, DC 15, RST 21, backlight 22 |
| RGB LED | Single addressable LED (driven as a WS2812, GRB order) | 8 |
| microSD (TF) slot | SPI mode (D1 and D2 not connected), shares MOSI and SCLK with the LCD | MISO 5, CS 4, MOSI 6, SCLK 7 |
| BOOT button | Strapping pin, low at reset enters download mode | 9 |
| RESET button | Chip enable | n/a |
| Regulator | ME6217C33M5G LDO, 3.3 V, 800 mA max | n/a |

There are no sensors and no battery charger on the board.

Waveshare advises keeping the backlight at 50% or lower: full brightness for long periods can leave dark shadows on the panel. This firmware caps the backlight PWM at 50%, so 100% in Home Assistant means 50% on the panel.

### Header pins

| Left side | Right side |
|---|---|
| 5V | TXD (UART0 TX) |
| GND | RXD (UART0 RX) |
| 3V3 (out) | GPIO13 |
| GPIO0 (ADC) | GPIO12 |
| GPIO1 (ADC) | GPIO23 |
| GPIO2 (ADC) | GPIO20 |
| GPIO3 (ADC) | GPIO19 |
| GPIO4 (ADC) | GPIO18 |
| GPIO5 (ADC) | GPIO9 |

All GPIOs support UART, PWM, I2S and I2C through the GPIO matrix; GPIO0 to GPIO5 also have ADC.

- GPIO4 and GPIO5 are also used by the microSD slot (CS and MISO): do not wire anything else to them when a card is used.
- GPIO9 is the BOOT button: if something pulls it low at reset, the board enters download mode.

### Download mode

Hold BOOT, press and release RESET, then release BOOT. Usually not needed: esptool and ESPHome reset the chip through the USB-Serial/JTAG interface on their own.

### Factory firmware

The board ships with an Arduino demo built with arduino-esp32 3.3.2 (ESP-IDF 5.5.1), showing `Hello Ardino and LVGL!` (typo included). The label comes from `Lvgl_Init()` in Waveshare's `LVGL_Driver.cpp` (LVGL 8 API). The image header sets the flash size to 4 MB.

| Name | Type | Subtype | Offset | Size |
|---|---|---|---|---|
| nvs | data | nvs | 0x009000 | 20K |
| otadata | data | ota | 0x00e000 | 8K |
| app0 | app | ota_0 | 0x010000 | 3072K |
| spiffs | data | spiffs | 0x310000 | 896K |
| coredump | data | coredump | 0x3f0000 | 64K |

Back up the whole flash before flashing anything else, to be able to go back to it:

```sh
uvx --from esptool esptool --port /dev/ttyACM0 --baud 921600 read-flash 0 ALL factory-flash.bin
uvx --from esptool esptool --port /dev/ttyACM0 verify-flash 0 factory-flash.bin
```

Restore it:

```sh
uvx --from esptool esptool --port /dev/ttyACM0 write-flash 0 factory-flash.bin
```

## Resources

- Waveshare: [wiki](https://www.waveshare.com/wiki/ESP32-C6-LCD-1.47), [docs](https://docs.waveshare.com/ESP32-C6-LCD-1.47), [schematic](https://files.waveshare.com/wiki/ESP32-C6-LCD-1.47/ESP32-C6-LCD-1.47_schemetics.pdf), [demos](https://files.waveshare.com/wiki/ESP32-C6-LCD-1.47/ESP32-C6-LCD-1.47-Demo.zip) (Arduino with LVGL 8, ESP-IDF 5.3+).
- Board support elsewhere: [Zephyr](https://docs.zephyrproject.org/latest/boards/waveshare/esp32c6_lcd_1_47/doc/index.html) (`esp32c6_lcd_1_47`), [CircuitPython](https://circuitpython.org/board/waveshare_esp32_c6_lcd_1_47/).
- ESPHome configs shared in [this Home Assistant community thread](https://community.home-assistant.io/t/help-needed-to-get-this-tiny-screen-working-esp32-c6-1-47in-display/800102).
- Known issue reported on [esp32.com](https://esp32.com/viewtopic.php?p=147339): the ST7789 driver of the Waveshare Arduino examples has a screen rotation bug.
- Community projects: VolosR/WaveShareC6lvglexample (LVGL 8 + SquareLine Studio decorative dashboard), thelastoutpostworkshop/ESP32-C6-LCD-1.47_LVGL9_Crypto_Monitor, thelastoutpostworkshop/ESP32-C6-LCD-1.47_video_player, nishad2m8/Squareline-OBP, moononournation/animation_clock.
