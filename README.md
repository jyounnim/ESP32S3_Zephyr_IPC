# ESP32-S3 Zephyr Dual-Core IPC Examples

**[한국어 버전](README_kr.md)**

A five-lab series that teaches how to work with ESP32-S3's two cores (PROCPU/APPCPU) under Zephyr RTOS, from an IPC (inter-processor communication) point of view - starting at the lowest level and working up to a real application.

## What this series covers

On ESP32-S3, Zephyr uses **AMP (Asymmetric Multiprocessing), not SMP**: core 0 (PROCPU) and core 1 (APPCPU) don't share a single OS image - they are two completely independent Zephyr images, each built and flashed separately (handled together in one shot with `west build --sysbuild`). On top of that, **APPCPU does not yet support the UART console** (printk/logging), so the two cores need some form of IPC just to cooperate at all.

Zephyr offers several layers of API for this - from the lowest-level, signal-only mailbox (MBOX/IPM), through directly managing shared memory by hand, up to `IPC Service`, which manages a real message queue for you at the framework level. This series walks that whole spectrum, one lab at a time, on real hardware (an ESP32-S3-DevKitC-1).

## Lab list

| Lab | Topic | API used | What you learn |
| --- | --- | --- | --- |
| [01](01_HELLO_DUALCORE_LAB/README.md) | Hello Dual-Core | none (+ IPM just to wake APPCPU) | Building/flashing both images at once with `--sysbuild`, and feeling this platform's fundamental constraint of APPCPU having no log output |
| [02](02_MBOX_DOORBELL_LAB/README.md) | MBOX Doorbell | `mbox.h` (signal only) | Exchanging a pure signal ("doorbell") with no data payload, in both directions |
| [03](03_SHM_RACE_LAB/README.md) | Shared Memory + MBOX Notification | `mbox.h` + hand-managed shared SRAM | Directly observing, on real hardware, the race condition that shows up once you pass real data through shared memory |
| [04](04_IPC_SERVICE_ICMSG_LAB/README.md) | IPC Service (icmsg backend) | `ipc_service.h` | Seeing how a standard framework (a real message queue) solves Lab 03's data-loss problem |
| [05](05_AHT20_BMP280_MultiSensor/README.md) | AHT20+BMP280 Multi-Sensor (application) | `ipm.h` (legacy IPM) | A complete example of the legacy-but-still-practical IPM API used in a real sensor-to-display application |

Each lab folder is laid out the same way:

```
NN_LAB_NAME/
├── README.md              English write-up (GitHub shows this automatically when you open the folder)
├── README_kr.md           Korean write-up
├── TROUBLESHOOTING.md / TROUBLESHOOTING_kr.md   (only for labs that need one)
└── lab/                   lab code (lab/ for procpu, lab/remote/ for appcpu)
```

Every lab's document is written to be **self-contained** - wiring, build steps, and code explanations are given in full every time, so you can do any lab on its own without having read the earlier ones.

## Shared hardware conventions

- Board: one **ESP32-S3-DevKitC-1** (same for every lab)
- Build targets: `esp32s3_devkitc/esp32s3/procpu` (core 0) and `esp32s3_devkitc/esp32s3/appcpu` (core 1) - always built and flashed together with `west build --sysbuild`
- Heartbeat/debug LED convention: **PROCPU = GPIO2, APPCPU = GPIO42** (established in Lab 01, reused through Labs 02-04)
- Labs that need a button reuse the DevKitC-1's own onboard BOOT button (GPIO0, the `sw0` alias) - no extra wiring
- Lab 05 uses real I2C sensors (AHT20, BMP280) and an OLED (SSD1306) instead of LEDs, split across two buses - I2C0 (GPIO8/9) and I2C1 (GPIO4/5) - one per core; see that lab's doc for full wiring details

## Common build commands

```bash
cd <NN_LAB_NAME>/lab

# build both images (procpu + appcpu) together
west build -p always --sysbuild -b esp32s3_devkitc/esp32s3/procpu .

# flash both images in one shot
west flash
```

- If you forget `--sysbuild`, only the procpu image gets built and `remote/` (appcpu) is silently ignored.
- Espressif boards default to also building MCUboot whenever `--sysbuild` is used. Every lab in this series is a plain two-image AMP build with nothing to do with OTA, so each lab's `sysbuild.conf` turns that off with `SB_CONFIG_BOOTLOADER_NONE=y`.
- Watch PROCPU's log as usual over a serial terminal (115200bps); APPCPU is observed via LED, or (from Lab 02 onward) via IPC messages that PROCPU logs on its behalf.

## What this series decided not to cover

Lab 05 was originally going to cover the `IPC Service` framework's `rpmsg` (OpenAMP-based) backend. Real-hardware bring-up hit a full system lockup right after a successful endpoint bind, and investigation showed ESP32-S3 isn't on the list of officially supported boards for Zephyr's own sample (`static_vrings`) for that backend, so this board combination doesn't use it. Lab 05 was replaced with the opposite end of the spectrum instead - the legacy IPM API put to real practical use.

## Further reading

The AHT20+BMP280 sensor code used in Lab 05 originates from a separate repository, [`jyounnim/zephyr_sensor`](https://github.com/jyounnim/zephyr_sensor), which keeps growing with more sensor examples that also use dual-core + IPC.
