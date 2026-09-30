# Lab 05: AHT20 + BMP280 Multi-Sensor (Dual-Core, IPM, SSD1306)

**[한국어 버전](README_kr.md)**

> The fifth and final lab in this roadmap. Built and verified on real hardware
> with the ESP32-S3-DevKitC-1.

## 0. Where this lab comes from

This lab is adapted from the `01_AHT20_BMP280_MultiSensor` example in the
separate [`jyounnim/zephyr_sensor`](https://github.com/jyounnim/zephyr_sensor)
repository, restructured to match this IPC roadmap's numbering and document
format (KR written first, then EN, with the write-up living as
README.md/README_kr.md at the lab root next to the `lab/` folder). The code itself is carried over exactly as it was already
verified on real hardware. Building on the layered view of IPC from Labs
01-04 (raw MBOX signaling all the way up to the `IPC Service` framework),
this lab's purpose is to see the **legacy IPM API** used in a real
application - sensors feeding a display.

The `zephyr_sensor` repository keeps growing with more sensor examples that
also use dual-core + IPC, beyond just this one - once this lab makes sense,
it's worth browsing that repository for other sensors you might be
interested in.

## 1. Overview

core0 (procpu) reads two sensors - AHT20 (temperature + humidity) and BMP280
(temperature + pressure) - each in its own thread. core1 (appcpu) receives
those values and displays them on an SSD1306 OLED. The two cores are
completely independent Zephyr images (AMP, built with `west build
--sysbuild`), and values are handed off between them over ESP32's IPM
(inter-processor mailbox) - unlike the MBOX / IPC Service APIs used in Labs
01-04, this is the only lab in the series that uses the **legacy IPM API**
(`ipm.h`, `CONFIG_ESP32_SOFT_IPM`) for real communication (`&ipm0` itself
already showed up in Lab 01 as the thing that has to be enabled for appcpu
to boot at all, but no actual data went through it there).

```
core0 (procpu)                              core1 (appcpu)
────────────────                            ────────────────
[aht20_thread]  100ms  (temperature + humidity)
   -> push immediately if temperature moves >=1C
[bmp280_thread] 100ms  (temperature + pressure)
   -> push immediately if pressure moves >=10 hPa

[heartbeat_thread] 1000ms - push unconditionally (link keep-alive)

   --(all three paths above)--> push_snapshot_to_core1()
                                    always sends the FULL snapshot
                                    (all 4 values: AHT20 T/H, BMP280 T/P)
                                        |
                                ipm_send(channel=2)
                                        |
                                        v  (physical mailbox)
                                              [ipm_callback] (ISR context)
                                                      |
                                              k_msgq_put (K_NO_WAIT)
                                                      |
                                                      v
                                            [display_thread] (k_msgq_get, 2s timeout)
                                                      |
                                    SSD1306 (I2C1) redraw: LINK, AHT20 T/H, BMP280 T/P
```

**Push triggers**:
- AHT20 temperature: pushes immediately when it moves by >=1.0C since the
  last value actually sent
- BMP280 pressure: pushes immediately when it moves by >=10 hPa since the
  last value actually sent
- AHT20 humidity / BMP280 temperature: "passenger" values with no threshold
  of their own - they ride along in whichever payload goes out next
  (a threshold trigger on the other channel, or the 1s heartbeat)
- core1 considers the link down if no message has arrived in 2 seconds

**Why split the cores, and why split the I2C buses too**: on ESP32-S3,
Zephyr's core0/core1 is AMP, not SMP - the two cores run completely
independent images built and flashed separately. Letting both cores drive
the same physical I2C controller at once is unsafe, so sensors live on
I2C0 (owned exclusively by core0) and the display lives on I2C1 (owned
exclusively by core1).

## 2. Sensors

### AHT20 (temperature + humidity)
- Made by Aosong, fixed I2C address `0x38`
- A capacitive humidity sensor plus a temperature sensor on one chip.
  Sequence: `0xAC` trigger command -> ~80ms conversion time -> read 6 data
  bytes + 1 CRC byte
- Uses Zephyr's in-tree driver: devicetree compatible `"aosong,aht20"`
  (`drivers/sensor/aosong/dht20/dht20.c` - AHT20/DHT20/AM2301B share
  register layouts, so one driver file covers all three)
- **Datasheet**: [Official Aosong AHT20 datasheet (PDF)](https://www.aosong.com/userfiles/files/media/Data%20Sheet%20AHT20.pdf)
- **Self-heal**: on real hardware, this particular unit was observed to get
  stuck returning a CRC-valid but physically impossible reading (~145C,
  humidity pinned at exactly ~50% - the classic signature of half of the
  ADC's 20-bit range, i.e. an unconverted/reset register rather than a
  real measurement). A Zephyr MCU reset alone did not clear it; only
  physically removing and reapplying the sensor's own VCC did. Since that
  isn't practical in the field, the AHT20's documented soft-reset command
  (`0xBA`, no payload, ~20ms to complete) is sent directly over I2C,
  bypassing the Zephyr sensor API entirely (it has no public hook for
  this) - triggered automatically whenever a reading falls outside
  AHT20's rated operating range (-40C to 85C). See `aht20_soft_reset()`
  in `main.c`.

### BMP280 (temperature + pressure)
- Made by Bosch, I2C address is `0x76` (SDO tied to GND) or `0x77` (SDO
  tied to VDD) depending on the module - **this project's module was
  confirmed at 0x77 via a real I2C bus scan**
- A piezoresistive pressure sensor plus a temperature sensor on one chip.
  The temperature channel exists less to report ambient temperature to
  the user and more because the pressure-compensation math needs a
  `t_fine` value derived from it
- Uses Zephyr's in-tree driver: devicetree compatible `"bosch,bme280"` -
  this driver reads the chip ID register (0xD0) to auto-detect BME280
  (has humidity, ID 0x60) vs BMP280 (no humidity, ID 0x58), so there is
  no separate BMP280-only compatible string
- The pressure channel comes back in kPa; this lab multiplies by 10 to
  get hPa
- **Datasheet**: [Official Bosch Sensortec BMP280 datasheet (PDF)](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp280-ds001.pdf)
- **Known limitation**: pressure occasionally jumps to an implausible
  value (e.g. mid-600s hPa). Unlike AHT20, this lab does not implement an
  automatic recovery for it yet - see "Known limitations" below.

### SSD1306 (128x64 OLED, owned exclusively by core1)
- OLED controller made by Solomon Systech
- **Datasheet**: [Solomon Systech SSD1306 datasheet (PDF, Adafruit mirror)](https://cdn-shop.adafruit.com/datasheets/SSD1306.pdf)
- This lab drives it with a custom I2C driver (`ssd1306_display.c`)
  reused from an earlier lab in this display series, instead of Zephyr's
  built-in display subsystem - this avoids a confirmed platform issue
  where the ESP32 Zephyr I2C driver doesn't reliably handle multi-segment
  write transactions, by always merging the control byte and the pixel
  payload into a single `i2c_write()` call
- Its address (0x3C or 0x3D) is auto-probed at boot
- Text is rendered with a small original 5x7 dot-matrix font
  (`font5x7.h`) covering just the labels, digits, and symbols this lab
  needs

## 3. IPM protocol (`ipm_protocol.h`)

```c
#define IPM_SENSOR_CHANNEL 2   /* channels 0/1 are reserved by the platform, 2/3 are free */

struct ipm_sensor_payload {
    float aht20_temp_c;         /* has a threshold (1.0C) */
    float aht20_humidity_pct;   /* passenger - no threshold */
    uint8_t aht20_valid;

    float bmp280_temp_c;        /* passenger - no threshold */
    float bmp280_pressure_hpa;  /* has a threshold (10 hPa) */
    uint8_t bmp280_valid;
};
```
`lab/src/ipm_protocol.h` and `lab/remote/src/ipm_protocol.h` are two
hand-maintained copies (sysbuild compiles them as fully separate Zephyr
applications, so there's no shared build target to enforce a single
copy) - update both if you change the fields.

## 4. Parts and wiring

Unlike the earlier labs, this one wires up three real I2C sensor/display
modules instead of LEDs or a button. Everything is listed from scratch so
this lab stands on its own - this wiring is new, separate from Labs 01-04's
LED wiring.

| Part | Qty |
| --- | --- |
| ESP32-S3-DevKitC-1 | 1 |
| AHT20 (temperature+humidity sensor module) | 1 |
| BMP280 (pressure sensor module) | 1 |
| SSD1306 128x64 OLED (I2C mode) | 1 |
| Breadboard + jumper wires | a few |
| USB cable | 1 |

| Signal | GPIO | Connects to | Owning core |
| --- | --- | --- | --- |
| I2C0 SDA | **GPIO8** (`&gpio0`) | AHT20 SDA, BMP280 SDA | core0 (procpu) |
| I2C0 SCL | **GPIO9** (`&gpio0`) | AHT20 SCL, BMP280 SCL | core0 (procpu) |
| I2C1 SDA | **GPIO4** (`&gpio0`) | SSD1306 SDA | core1 (appcpu) |
| I2C1 SCL | **GPIO5** (`&gpio0`) | SSD1306 SCL | core1 (appcpu) |
| VCC / GND | 3.3V / GND | all of AHT20, BMP280, SSD1306 | - |

GPIO8/9 (I2C0) reuse the convention already established elsewhere in this
project (the `zephyr_display` track); GPIO4/5 (I2C1) is this lab's first use
of the second I2C bus. **Wire AHT20/BMP280 to core0's I2C0 and SSD1306 to
core1's I2C1 - never put the sensors and the display on the same bus**
(see "why split the cores, and why split the I2C buses too" in section 1
for why). AHT20 and BMP280 can safely share I2C0 (multiple I2C devices on
one bus is normal as long as their addresses differ).

After wiring, always scan the actual bus with `lab_tools/i2c0_scanner`
(a separate diagnostic-only build) before touching any address in the
overlay: AHT20 is fixed at `0x38`, but BMP280 comes out at `0x76` or `0x77`
depending on the module's SDO wiring, and SSD1306 modules commonly use
either `0x3C` or `0x3D`. This code's default (BMP280 = 0x77) matches the
module actually verified in this project - a different module may differ.

## 5. Building

Main lab (dual-core):
```
west build -p always --sysbuild -b esp32s3_devkitc/esp32s3/procpu \
    05_AHT20_BMP280_MultiSensor/lab
west flash
```
`lab/sysbuild.conf` needs `SB_CONFIG_BOOTLOADER_NONE=y` - Espressif boards
default to building MCUboot alongside the app whenever `--sysbuild` is
used, which this lab doesn't need since it's a plain two-image AMP build
with nothing to do with OTA.

I2C bus scanner (diagnostic tool, no sysbuild needed):
```
west build -p always -b esp32s3_devkitc/esp32s3/procpu \
    05_AHT20_BMP280_MultiSensor/lab_tools/i2c0_scanner
west flash
```

## 6. Directory layout

```
05_AHT20_BMP280_MultiSensor/
├── README.md                                           (this document)
├── README_kr.md                                        (Korean version)
├── TROUBLESHOOTING.md                                  (troubleshooting, English)
├── TROUBLESHOOTING_kr.md                               (troubleshooting, Korean)
├── lab/                                                (main lab - dual-core)
│   ├── CMakeLists.txt
│   ├── prj.conf
│   ├── sample.yaml
│   ├── sysbuild.cmake
│   ├── sysbuild.conf
│   ├── boards/
│   │   └── esp32s3_devkitc_esp32s3_procpu.overlay
│   ├── src/                                            (core0/procpu)
│   │   ├── main.c
│   │   ├── sensor_hub.c / sensor_hub.h                 (sensor value snapshot store)
│   │   └── ipm_protocol.h                              (IPM payload definition)
│   └── remote/                                         (core1/appcpu)
│       ├── CMakeLists.txt
│       ├── prj.conf
│       ├── boards/
│       │   └── esp32s3_devkitc_esp32s3_appcpu.overlay
│       └── src/
│           ├── main.c
│           ├── ssd1306_display.c / .h                  (OLED driver)
│           ├── font5x7.h                                (5x7 dot-matrix font)
│           └── ipm_protocol.h                          (hand-synced with core0's copy)
└── lab_tools/
    └── i2c0_scanner/                                    (diagnostic - not part of sysbuild)
        ├── CMakeLists.txt
        ├── prj.conf
        ├── boards/
        │   └── esp32s3_devkitc_esp32s3_procpu.overlay
        └── src/main.c
```

## 7. Expected results

### core0 boot log (normal)
```
[00:00:00.xxx,xxx] <inf> app_procpu: 05_AHT20_BMP280_MultiSensor (core0/procpu) starting
```

### core1 screen
```
LINK: OK

AHT20 T:24.3C
AHT20 H:55.2%

BMP280 T:28.3C
BMP280 P:998 hPa
```
- Before the first message arrives, the screen shows `WAITING FOR CORE0...`
- If nothing arrives from core0 for 2+ seconds, the top line switches to
  `LINK: FAIL` (the last known readings stay on screen as-is)

## 8. Real-hardware checklist

- [ ] `west build --sysbuild` builds both core0 and core1 with no errors
      (an MCUboot configure error means `lab/sysbuild.conf` is missing
      `SB_CONFIG_BOOTLOADER_NONE=y`)
- [ ] Serial terminal shows `05_AHT20_BMP280_MultiSensor (core0/procpu) starting`
- [ ] No `I2C0 bus not ready` / `I2C1 bus not ready` errors
- [ ] No `AHT20 device not ready` / `BMP280 device not ready` /
      `IPM device not ready` warnings (if any appear, re-check wiring and the
      overlay's I2C addresses with `lab_tools/i2c0_scanner` first)
- [ ] The SSD1306 shows `WAITING FOR CORE0...` briefly, then switches to
      `LINK: OK` with all four AHT20/BMP280 values displayed
- [ ] The displayed values keep updating every few seconds (breathing on the
      sensor or holding it in your hand is an easy way to force a visible
      change)
- [ ] Cutting power to core1 only (or core0 only) for a few seconds flips the
      top line to `LINK: FAIL` after 2 seconds (this confirms the link-down
      detection logic - optional if you'd rather not physically interrupt
      power)

## 9. Known limitations

- **BMP280 pressure occasionally spikes to an implausible value.** Both
  sensors have been observed misbehaving at roughly the same time on this
  hardware, which points more toward a shared root cause (power rail
  quality, breadboard contact, missing decoupling capacitance) than two
  independently defective chips. This lab does not add an automatic
  recovery for BMP280 the way it does for AHT20 - if needed, the next
  step would be a similar plausibility-check-and-reset using BMP280's own
  hardware reset register (write `0xB6` to register `0xE0`).
- **AHT20's 100ms period is not perfectly exact.** The sensor's own
  conversion time is ~80ms, leaving little slack in a 100ms loop. On top
  of that, both sensor threads on core0 share one I2C0 bus mutex
  (`g_i2c0_bus_lock`), so the BMP280 thread can get pushed later than a
  clean 100ms while AHT20 is mid-conversion. This is an intentional
  trade-off: never letting the two threads' I2C0 transactions overlap
  was prioritized over exact timing.

## 10. Ideas for extension

- Add an AHT20-style plausibility check + hardware-reset self-heal for
  BMP280
- Give AHT20 humidity and BMP280 temperature their own independent
  thresholds so all four channels can trigger a push on their own
- Use IPM channel 3 for a core1 -> core0 backchannel (e.g. adjusting
  thresholds live from a button on core1)

## 11. Wrapping up this roadmap

Labs 01-05 complete this ESP32-S3 Zephyr dual-core IPC roadmap:

- Lab 01: booting both cores at once with no IPC at all, and feeling the
  limits of the appcpu console first-hand
- Lab 02: passing a pure signal (no data) back and forth over MBOX
- Lab 03: the race condition that shows up once shared memory + MBOX is
  used to pass real data between cores
- Lab 04: solving Lab 03's problem at the framework level with the
  `IPC Service` (icmsg) framework
- Lab 05 (this lab): the legacy `IPM` API put to work in a complete
  sensor-to-display application

In other words, the series climbs from "the single lowest-level signal"
(Lab 02) up to "a framework that manages the message queue for you"
(Lab 04), then finishes at the opposite end of that spectrum with "a
legacy API that still works well in practice" (IPM, this lab). If you're
curious about more sensors handled with dual-core + IPC, keep an eye on
the [`jyounnim/zephyr_sensor`](https://github.com/jyounnim/zephyr_sensor)
repository mentioned in section 0.
