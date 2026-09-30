# ESP32-S3 Zephyr Dual-Core IPC 예제 모음

**[English version](README.md)**

ESP32-S3의 두 코어(PROCPU/APPCPU)를 Zephyr RTOS에서 다루는 방법을, IPC(코어 간 통신) 관점에서 가장 저수준부터 실제 응용까지 단계별로 배우는 5개짜리 실습 시리즈입니다.

## 이 시리즈가 다루는 것

ESP32-S3에서 Zephyr는 **SMP가 아니라 AMP(Asymmetric Multiprocessing)**를 사용합니다. 즉 코어 0(PROCPU)과 코어 1(APPCPU)이 하나의 OS 이미지를 나눠 쓰는 게 아니라, 완전히 독립된 두 개의 Zephyr 이미지로 각각 빌드/플래시됩니다(`west build --sysbuild`로 한 번에 처리). 게다가 **APPCPU는 아직 UART 콘솔(printk/logging)을 지원하지 않기 때문에**, 두 코어가 서로 협력하려면 반드시 어떤 형태로든 IPC가 필요합니다.

Zephyr는 이 IPC를 위해 여러 계층의 API를 제공합니다 - 가장 저수준의 신호 전용 메일박스(MBOX/IPM)부터, 직접 공유 메모리를 관리하는 방식, 그리고 메시지 큐를 프레임워크 차원에서 대신 관리해주는 `IPC Service`까지. 이 시리즈는 이 스펙트럼을 실제 하드웨어(ESP32-S3-DevKitC-1)에서 하나씩 순서대로 체험하도록 구성했습니다.

## 실습 목록

| Lab | 주제 | 사용 API | 배우는 것 |
| --- | --- | --- | --- |
| [01](01_HELLO_DUALCORE_LAB/README_kr.md) | Hello Dual-Core | IPC 없음 (+ APPCPU 기동용 IPM) | `--sysbuild`로 두 이미지 동시 빌드/플래시. APPCPU는 로그가 안 나온다는 이 플랫폼의 근본적 제약을 먼저 체감 |
| [02](02_MBOX_DOORBELL_LAB/README_kr.md) | MBOX 도어벨 | `mbox.h` (신호 전용) | 데이터 없이 순수 신호(초인종)만 양방향으로 주고받기 |
| [03](03_SHM_RACE_LAB/README_kr.md) | 공유 메모리 + MBOX 알림 | `mbox.h` + 공유 SRAM 직접 관리 | 실제 데이터를 공유 메모리로 주고받을 때 생기는 레이스 컨디션을 실기로 직접 관찰 |
| [04](04_IPC_SERVICE_ICMSG_LAB/README_kr.md) | IPC Service (icmsg backend) | `ipc_service.h` | 표준 프레임워크(진짜 메시지 큐)가 Lab 03의 데이터 손실 문제를 어떻게 해결하는지 확인 |
| [05](05_AHT20_BMP280_MultiSensor/README_kr.md) | AHT20+BMP280 멀티센서 (응용) | `ipm.h` (레거시 IPM) | 레거시지만 여전히 실전에서 잘 동작하는 IPM API가 실제 센서→디스플레이 응용에 쓰이는 완성된 예제 |

각 Lab 폴더는 다음과 같이 구성되어 있습니다:

```
NN_LAB_NAME/
├── README.md              영어 설명 문서 (GitHub에서 폴더 진입 시 자동으로 표시됨)
├── README_kr.md           한국어 설명 문서
├── TROUBLESHOOTING.md / TROUBLESHOOTING_kr.md   (필요한 랩에만 존재)
└── lab/                   실습 코드 (procpu용 lab/, appcpu용 lab/remote/)
```

각 랩의 문서는 **독립적으로 실습할 수 있도록** 배선/빌드/코드 설명을 매번 처음부터 전부 담고 있습니다 - 이전 랩을 안 봤어도 그 랩 문서 하나만으로 실습이 가능합니다.

## 공통 하드웨어 규칙

- 보드: **ESP32-S3-DevKitC-1** 1개 (모든 랩 공통)
- 빌드 타겟: `esp32s3_devkitc/esp32s3/procpu` (코어0), `esp32s3_devkitc/esp32s3/appcpu` (코어1) - `west build --sysbuild`로 항상 두 이미지를 함께 빌드/플래시
- 하트비트/디버그 LED 관례: **PROCPU = GPIO2, APPCPU = GPIO42** (Lab 01에서 확정, Lab 02~04까지 계속 재사용)
- 버튼이 필요한 랩은 DevKitC-1 보드에 내장된 BOOT 버튼(GPIO0, `sw0` alias)을 그대로 사용 - 추가 배선 없음
- Lab 05는 LED 대신 실제 I2C 센서(AHT20, BMP280) + OLED(SSD1306)를 사용하며, I2C0(GPIO8/9)·I2C1(GPIO4/5) 두 버스로 코어별 역할을 분리합니다 - 자세한 배선은 해당 문서 참고

## 공통 빌드 명령

```bash
cd <NN_LAB_NAME>/lab

# procpu + appcpu 두 이미지를 한 번에 빌드
west build -p always --sysbuild -b esp32s3_devkitc/esp32s3/procpu .

# 두 이미지 모두 한 번에 플래시
west flash
```

- `--sysbuild`를 빠뜨리면 procpu 이미지만 빌드되고 `remote/`(appcpu)가 무시되니 주의하세요.
- Espressif 보드는 `--sysbuild` 사용 시 기본적으로 MCUboot도 같이 빌드하려 하는데, 이 시리즈의 모든 랩은 OTA와 무관한 단순 2-이미지 AMP 구성이라 각 랩의 `sysbuild.conf`에서 `SB_CONFIG_BOOTLOADER_NONE=y`로 꺼둡니다.
- PROCPU 쪽 로그는 평소처럼 시리얼 터미널(115200bps)로 확인하고, APPCPU 쪽은 LED나 (Lab 02부터는) IPC를 통해 PROCPU가 대신 출력하는 로그로 확인합니다.

## 이 시리즈에서 다루지 않기로 한 것

원래 Lab 05로 `IPC Service`의 `rpmsg`(OpenAMP 기반) backend를 다룰 계획이었으나, 실기 디버깅 중 bind 직후 시스템이 완전히 멈추는 문제를 겪었고, 조사 결과 ESP32-S3가 Zephyr의 해당 공식 샘플(`static_vrings`)의 지원 보드 목록에 없다는 것을 확인해 이 보드 조합에서는 채택하지 않기로 했습니다. 대신 Lab 05는 스펙트럼의 반대쪽 끝 - 레거시 IPM API의 실전 활용 - 을 다루는 것으로 대체했습니다.

## 더 볼거리

Lab 05에서 사용한 AHT20+BMP280 센서 코드의 원본은 별도 저장소 [`jyounnim/zephyr_sensor`](https://github.com/jyounnim/zephyr_sensor)에 있으며, 이 저장소에는 듀얼코어+IPC를 사용하는 다른 센서 예제들도 계속 추가되고 있습니다.
