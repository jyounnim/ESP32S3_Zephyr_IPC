# ESP32-S3 Zephyr Dual-Core IPC 예제 로드맵

기존 `roadmap_KR.md`(OLED/TFT 디스플레이 시리즈, 01~08번)와는 **완전히 별개의 새 트랙**입니다. 번호는 01번부터 독립적으로 다시 시작합니다.

## 배경 / 전제 조건 (2026-09-12 기준 확정)

- 보드: ESP32-S3-DevKitC-1, 빌드 타겟은 `esp32s3_devkitc/esp32s3/procpu`와 `esp32s3_devkitc/esp32s3/appcpu` **두 개**(완전히 별도의 Zephyr 이미지) — `west build --sysbuild`로 동시 빌드/플래시
- ESP32-S3는 Zephyr에서 **SMP가 아니라 AMP만 지원** — 코어 두 개가 각자 독립된 OS 이미지를 실행하며, IPC로만 통신
- **appcpu는 아직 UART 콘솔/로그(printk 등)를 지원하지 않음** (Zephyr 공식 문서 명시) — appcpu 쪽 동작 확인은 LED/GPIO 토글로 하거나, IPC로 procpu에 데이터를 보내 procpu가 대신 출력해야 함
- Zephyr IPC 계층 구조(저수준 → 고수준): (레거시) `IPM driver API` / `MBOX driver API` → (직접 구현하는) `공유 메모리 + MBOX 알림` → `IPC Service` 프레임워크의 `icmsg` backend
- **★ 이 보드/Zephyr 조합의 핵심 플랫폼 특성 (2026-09-06, Lab 01 실기 디버깅으로 확정)**: **PROCPU와 APPCPU 양쪽 이미지에서, `&ipm0`(레거시 IPM) 또는 `&mbox0`(신식 MBOX) 둘 중 하나가 활성화돼 있지 않으면 APPCPU 코어 자체가 reset에서 풀리지 않는다.** 실제 IPC를 쓰든 안 쓰든 무관 — GPIO도, 어떤 코드도 전혀 실행되지 않음
- **★ 정책 확정 (2026-09-11, Lab 02 실기 검증으로 확정 — 2026-09-06자 "항상 IPM" 정책을 대체)**: ESP32-S3 devicetree상 `&ipm0`와 `&mbox0`는 **완전히 같은 물리 하드웨어(같은 레지스터 블록, 같은 공유메모리, 같은 인터럽트 소스 FROM_CPU_INTR0/1)에 대한 서로 다른 두 드라이버 바인딩**임을 확인. Lab 02에서 `&mbox0`만 켜고(`&ipm0`는 끈 채로) 실기 테스트한 결과 **APPCPU가 정상 기동**했음 — 즉 APPCPU 기동에 필요한 건 `ipm0`라는 특정 드라이버가 아니라 이 공유 하드웨어 자체의 활성화. **따라서 이 시리즈의 모든 dual-core/IPC 랩은 `ipm0`/`mbox0` 중 그 랩이 실제로 가르치는 API 하나만 켠다** (동시에 켜면 같은 인터럽트 소스를 두 드라이버가 요구하게 되므로 지양)
- 이 트랙의 GPIO 관례: 하트비트/디버그 LED는 **PROCPU=GPIO2 (`&gpio0`), APPCPU=GPIO42 (`&gpio1`, 로컬 인덱스 42-32=10)**로 확정. 버튼이 필요한 랩은 DevKitC-1 보드 내장 BOOT 버튼(GPIO0, Zephyr가 이미 `sw0` alias로 노출)을 재사용 — 추가 배선 없음
- 레거시 IPM API와 최신 MBOX 바인딩(`espressif,mbox-esp32`)은 ESP32-S3 Zephyr에 **둘 다 존재**함 — 02~04번은 최신 API(MBOX/IPC Service) 학습 목적으로 진행. 레거시 IPM 자체를 실제 통신에 쓰는 예제는 05번(아래 참고)

## 새 로드맵 (진행 중)

| 번호 | 주제 | 사용 API | 핵심 학습 목표 | 상태 |
| --- | --- | --- | --- | --- |
| 01 | Hello Dual-Core (IPC 없음) | — (+ APPCPU 기동용 IPM) | `--sysbuild`로 procpu+appcpu 동시 빌드/플래시 흐름 익히기. appcpu는 printk가 안 되므로 LED 토글로 "살아있음"만 확인 — appcpu 디버깅 제약을 먼저 체감 | **완료 - 실기 검증 완료 (2026-09-06)**. KR/EN 문서 + 트러블슈팅 문서(KR/EN) + 코드 전달 완료 |
| 02 | MBOX 도어벨 | `mbox.h` (raw, IPM 미사용) | 데이터 없이 순수 신호만 양방향 전달 (procpu BOOT 버튼 → appcpu LED 트리플 플래시, appcpu → procpu 3초 주기 도어벨). `mbox_send_dt`/`mbox_register_callback_dt` 감 잡기 + `&mbox0` 단독으로 APPCPU 기동 확인 | **완료 - 실기 검증 완료 (2026-09-11)**. KR/EN 문서 + 코드 전달 완료 |
| 03 | 공유 메모리 + MBOX 알림 | `mbox.h` (raw) + 기존 `ipmmem0` 재사용 | 작은 구조체(카운터 등)를 공유 SRAM에 쓰고 MBOX로 "새 데이터 있음"만 알림. 레이스 컨디션을 직접 겪어보고 flag/atomic으로 해결 — 프레임워크 없이 만들면 뭐가 힘든지 체감 | **완료 - 실기 검증 완료 (2026-09-11)**. KR/EN 문서 + 코드 전달 완료 |
| 04 | IPC Service — icmsg backend | `ipc_service.h` (icmsg) | Lab 03을 표준 프레임워크로 재구현. instance/endpoint 개념, 콜백 기반 request-response 예제 | **완료 - 실기 검증 완료 (2026-09-11)**. KR/EN 문서 + 코드 전달 완료 |
| 05 | 응용: AHT20+BMP280 멀티센서 → SSD1306 디스플레이 (레거시 IPM 기반) | `ipm.h` (`CONFIG_ESP32_SOFT_IPM`, `&ipm0`, channel 2) | 사용자의 `zephyr_sensor/01_AHT20_BMP280_MultiSensor` 실기 검증 코드를 이 로드맵 형식(KR/EN 문서 + LAB 폴더)으로 정리/이식 완료. **아래 "Lab 05 상세" 참고** | **완료 - 원 저장소 기준 실기 검증 완료, 이 로드맵 문서/폴더 정리 완료 (2026-09-12)**. KR/EN 문서(+트러블슈팅 KR/EN) + 코드 전달 완료 |

### Lab 01 상세 (완료, 2026-09-06 실기 검증 완료)

- 파일: `01_HELLO_DUALCORE_LAB/{README.md, README_kr.md, TROUBLESHOOTING.md, TROUBLESHOOTING_kr.md, lab/...}` (랩 루트에 위치, doc/ 폴더 없음 — 2026-09-30 개정) — zip으로 전달 완료
- IPC 없음 - procpu/appcpu 두 이미지가 완전히 독립적으로 실행되는 것만 증명하는 최소 스캐폴딩. procpu: GPIO2 LED 500ms 점멸 + `LOG_INF` 하트비트 로그. appcpu: GPIO42 LED 200ms 점멸만 (로그 없음 - UART 콘솔 미지원)
- **실기 디버깅 요약**: 처음엔 APPCPU LED가 전혀 안 켜져서, GPIO 핀(42→18→2 순서로 시도)/배선/LED 부품/빌드 캐시/플래시 전체 삭제까지 전부 의심하고 하나씩 배제했으나 전부 무관했음. 최종적으로 **PROCPU/APPCPU 양쪽에 `&ipm0` 활성화가 빠져있던 게 원인**으로 확정 — IPM을 켜자(실제 통신 코드 없이 노드/Kconfig만 추가) 즉시 정상 동작. 전체 디버깅 경위는 `01_HELLO_DUALCORE_LAB_TROUBLESHOOTING_KR.md`/`_EN.md`에 정리 (Lab 02에서 밝혀진 ipm0/mbox0 관계에 따라 "후속 확인" 절도 갱신됨)
- sysbuild로 MCUboot 비활성화(`SB_CONFIG_BOOTLOADER_NONE=y`), Lab 05 원본과 동일 패턴

### Lab 02 상세 (완료, 2026-09-11 실기 검증 완료)

- 파일: `02_MBOX_DOORBELL_LAB/{README.md, README_kr.md, lab/...}` (랩 루트에 위치, doc/ 폴더 없음 — 2026-09-30 개정) — zip으로 전달 완료 (트러블슈팅 문서 없음 - 디버깅이 아니라 의도된 실기 검증)
- 신호 전용(payload 없음) 양방향 MBOX 도어벨: procpu의 DevKitC-1 내장 BOOT 버튼(GPIO0, `sw0` alias, 추가 배선 없음) → appcpu 도어벨 → appcpu LED(GPIO42) 트리플 플래시 반응. 반대 방향: appcpu가 3초마다 자체적으로 procpu 도어벨을 울리고, procpu가 로그로 카운트 출력. 양쪽 다 Lab 01의 하트비트 LED 점멸도 그대로 유지
- devicetree: `mbox-consumer` 노드(`compatible = "vnd,mbox-consumer"`, Zephyr 공식 MBOX 샘플과 동일 패턴, mainline에 이미 내장된 테스트 바인딩)에 `mboxes`/`mbox-names`로 tx/rx 채널 지정. ESP32 MBOX 하드웨어는 방향당 채널 1개뿐이라 tx/rx 모두 `&mbox0 0`
- **핵심 실기 검증 결과**: `&ipm0`는 끄고 `&mbox0`만 켠 상태로 빌드 → APPCPU 정상 기동 확인. 이로써 "APPCPU 기동에 ipm0라는 특정 드라이버가 필요한 게 아니라, ipm0/mbox0가 가리키는 공유 하드웨어 자체의 활성화가 필요하다"는 게 실기로 증명됨 → 위 "배경" 절의 정책이 이 결과로 갱신됨
- API: `mbox_send_dt()`/`mbox_register_callback_dt()`/`mbox_set_enabled_dt()` (Zephyr 공식 `samples/drivers/mbox` 샘플의 `_dt` 헬퍼 패턴을 그대로 따름 - `MBOX_DT_SPEC_GET(DT_PATH(mbox_consumer), name)`으로 `struct mbox_dt_spec` 획득)

### Lab 03 상세 (완료, 2026-09-11 실기 검증 완료)

- 파일: `03_SHM_RACE_LAB/{README.md, README_kr.md, lab/...}` (랩 루트에 위치, doc/ 폴더 없음 — 2026-09-30 개정) — zip으로 최종본 전달 완료, 저장소에도 반영 완료
- **`reserved-memory` 결정 (사용자 확정, 2026-09-11)**: 새 영역을 devicetree에 정의하지 않고, `&ipm0`/`&mbox0`가 이미 예약해둔 `ipmmem0`(0x3fce5000, 0x400바이트)를 그대로 재사용. `drivers/mbox/mbox_esp32.c` 소스 확인 결과, `mbox_send_dt()`를 NULL 페이로드로만 호출하면(이 시리즈의 모든 랩이 그렇게 함) 드라이버가 이 공유메모리 버퍼를 전혀 건드리지 않는다는 것을 검증 — 따라서 안전하게 재사용 가능
- 설계: procpu가 200ms마다 `SHM_P2A_ADDR`(ipmmem0 앞쪽 절반)에 증가하는 `seq` 값을 쓰고 `&mbox0`로 신호(페이로드 없음)만 보냄. appcpu는 일부러 더 느린 350ms 주기로만 대기열을 처리하도록 설계 — procpu가 더 빠르므로 시간이 지날수록 backlog가 쌓임
- **레이스 컨디션 설계 + 실기 검증 결과**: `atomic_t`로 만든 대기 카운터(`pending`)는 알림이 아무리 몰려도 개수를 잃어버리지 않는다는 것(`drained_count`가 정확히 1씩 증가)과, 공유메모리 슬롯이 하나뿐이라 "그 알림이 담고 있던 실제 데이터 값"은 손실된다는 것(`last_seen_seq`가 445→446→448로 447을 건너뜀)을 실기 로그로 직접 확인 완료 — "알림 개수 보존"과 "데이터 보존"이 서로 다른 문제라는 것, 그리고 이게 왜 Lab 04의 진짜 메시지 큐 기반 `IPC Service` 프레임워크가 필요한 이유인지를 실측으로 증명

### Lab 04 상세 (완료, 2026-09-11 실기 검증 완료)

- 파일: `04_IPC_SERVICE_ICMSG_LAB/{README.md, README_kr.md, lab/...}` (랩 루트에 위치, doc/ 폴더 없음 — 2026-09-30 개정) — zip으로 최종본 전달 완료, 저장소에도 반영 완료
- **공유 메모리 발견**: ESP32-S3 SoC devicetree에 `ipmmem0`(1KB, Lab 02/03이 사용) 바로 다음에 `shm0`(16KB, `memory@3fce5400`)라는, 지금까지 어떤 랩도 쓰지 않은 별도 블록이 이미 선언되어 있는 것을 발견 — 이름과 위치로 볼 때 SoC 포팅 담당자가 범용 코어 간 공유용으로 마련해둔 것으로 추정. mainline Zephyr에 이 블록을 실제로 쓰는 기존 샘플은 찾지 못해 추론으로 설계했으나, **실기 검증 결과 문제없이 정상 동작함을 확인**
- 설계: `shm0`의 16KB를 8KB씩 나눠 icmsg의 tx/rx `reserved-memory` 영역으로 사용(`dcache-alignment = <0>`, `mboxes = <&mbox0 0>, <&mbox0 0>` — 기존 시리즈 관례 재사용). procpu가 200ms마다 증가하는 seq를 `ipc_service_send()`로 보내고, appcpu가 `received` 콜백 안에서 즉시 그대로 되돌려 보냄(icmsg 콜백이 워크큐 컨텍스트에서 락 없이 호출된다는 것을 `subsys/ipc/ipc_service/lib/icmsg.c` 소스로 확인한 뒤 콜백 내부에서 바로 `ipc_service_send()` 호출하도록 설계)
- **실기 검증 결과**: Lab 03이 겪은 "공유 슬롯 하나로는 중간 데이터가 손실된다"는 문제가 icmsg에서는 재현되지 않음을 확인 — `recv reply seq=`가 268, 269, 270...처럼 하나도 안 빠지고 순서대로 이어졌고, `total_received`가 매번 `total_sent`와 정확히 일치함(Lab 03의 `drained_count` vs `last_seen_seq`가 계속 벌어지던 것과 대비). `icmsg error`/`send buffer full` 없이 안정적으로 동작
- API: `ipc_service_open_instance()`, `ipc_service_register_endpoint()`(콜백 `bound`/`received`/`error`), `ipc_service_send()` — Zephyr 공식 `samples/subsys/ipc/ipc_service/icmsg` 샘플의 API 패턴을 그대로 따름 (devicetree/주소는 ESP32-S3용으로 새로 설계, 기존 샘플은 nRF5340 기준)
- 이 시리즈에서는 icmsg가 IPC Service 프레임워크를 대표하는 랩으로 마무리됨 (rpmsg/OpenAMP backend는 이 보드 조합에서 채택하지 않기로 함 — 아래 "미확정" 절 참고)

### Lab 05 상세 (2026-09-04, 사용자가 완성된 코드 제공)

- 원본: [`jyounnim/zephyr_sensor/01_AHT20_BMP280_MultiSensor`](https://github.com/jyounnim/zephyr_sensor/tree/main/01_AHT20_BMP280_MultiSensor) — 사용자가 zip으로 직접 전달, **실기 검증 완료** 상태
- **IPC는 이미 사용되어 있음** (사용자 질문에 대한 답) — 레거시 **IPM(Inter-Processor Mailbox)** API:
  - Kconfig: `CONFIG_IPM=y`, `CONFIG_ESP32_SOFT_IPM=y` (양쪽 코어 prj.conf 동일) — 소스 코드 주석에 "Kconfig 심볼은 `ESP32_SOFT_IPM`이 맞고 `IPM_ESP32`는 존재하지 않아 첫 빌드에서 경고/중단났었다"는 실전 트러블슈팅 기록이 남아있음
  - Devicetree: 양쪽 오버레이 모두 `&ipm0 { status = "okay"; };`
  - 채널: `IPM_SENSOR_CHANNEL = 2` (0/1은 플랫폼 예약, 2/3이 애플리케이션 가용 — 이 채널 관례는 사용자의 기존 `zephyr_curriculum` 로드맵 Lab 18에서 확립된 것)
  - 페이로드: `struct ipm_sensor_payload` (AHT20 온습도 + BMP280 온도/기압, 24바이트) — ESP32 IPM 드라이버의 메시지당 64바이트 제한 이내
  - 전송측(core0/procpu): `ipm_send(dev_ipm, 0, IPM_SENSOR_CHANNEL, &payload, sizeof(payload))`, 뮤텍스로 직렬화, **threshold 트리거(AHT20 온도 ≥1°C 변화 또는 BMP280 기압 ≥10 hPa 변화 시 즉시 전송) + 1초 heartbeat**로 전송 정책 구성 — 고정 주기 폴링이 아님
  - 수신측(core1/appcpu): `ipm_register_callback()` + `ipm_set_enabled()`로 등록, ISR 컨텍스트인 콜백은 `k_msgq_put(K_NO_WAIT)`로 메시지 큐에 복사만 하고 즉시 리턴, 실제 SSD1306 렌더링은 별도 display 스레드가 `k_msgq_get(K_MSEC(2000))`으로 블로킹 대기 — 2초 이상 무응답 시 "LINK: FAIL" 표시로 링크 끊김 감지
- **코어 분담**: core0(procpu)가 I2C0(SDA=GPIO8/SCL=GPIO9, 이 프로젝트 관례와 일치)로 AHT20+BMP280 읽기, core1(appcpu)이 I2C1(SDA=GPIO4/SCL=GPIO5, 이 랩에서 처음 쓰는 버스)로 SSD1306 출력 — AMP라 두 이미지가 물리 I2C 컨트롤러 하나를 공유할 수 없어서 애초에 버스를 분리한 설계
- 빌드: `west build -p always --sysbuild -b esp32s3_devkitc/esp32s3/procpu 05_AHT20_BMP280_MultiSensor/lab`, `sysbuild.conf`에서 `SB_CONFIG_BOOTLOADER_NONE=y`로 MCUboot 비활성화(단순 2-image AMP 빌드에는 불필요)
- AHT20 self-heal 로직(CRC는 유효하지만 물리적으로 불가능한 고정값에 갇히는 현상 감지 시 0xBA 소프트리셋 명령 직접 전송) 등 실기 디버깅 노하우가 코드 주석에 상세히 남아있음 — 트러블슈팅 문서(KR/EN)도 이미 원본에 포함되어 있음
- **완료 (2026-09-12)**: 이 로드맵 번호 체계에 맞게 폴더/파일명을 `05_AHT20_BMP280_MultiSensor`로 정리하고, 출처(`jyounnim/zephyr_sensor`) 명시 섹션, 준비물/배선 표, 디렉터리 구조, 정상 동작 체크리스트, 로드맵 마무리 섹션을 추가해 이 시리즈의 KR/EN 문서 포맷에 맞춰 재작성함 (원본 코드는 100% 그대로 사용, 문서만 재구성)

## 미확정 / 착수 시 결정할 사항

- **rpmsg/OpenAMP backend(IPC Service `static_vrings`)는 이 로드맵에서 채택하지 않기로 함 (2026-09-12 확정)** — devicetree 바인딩 수정, 버퍼 크기 수정까지 마치고 실기에서 bind까지는 성공했으나, bind 직후 procpu가 아무 로그/GPIO 반응 없이 완전히 멈추는 문제가 재현됨. 조사 결과 Zephyr 공식 `static_vrings` 샘플의 지원 보드 목록에 ESP32 계열이 없어, 이 보드 조합에서는 다루지 않기로 결정. 관련 코드/문서/디버깅 기록은 전부 폐기함
- ~~Lab 05 폴더/파일명을 이 로드맵 번호 체계에 맞게 리네이밍할지, 원본 구조를 그대로 유지할지~~ → **해결 (2026-09-12): 폴더/파일명은 로드맵 번호 체계(`05_AHT20_BMP280_MultiSensor`)로 리네이밍, 내부 구조(`lab/`, `lab/remote/`, `lab_tools/`)는 원본 그대로 유지** (문서 위치(`doc/`)는 이후 2026-09-30에 다시 개정됨 — 아래 항목 참고)
- 파일 구조 컨벤션 (2026-09-30 개정): `NN_TOPIC_LAB/{README.md, README_kr.md, (필요 시) TROUBLESHOOTING.md, TROUBLESHOOTING_kr.md, lab/{CMakeLists.txt, prj.conf, sample.yaml, sysbuild.cmake, sysbuild.conf, boards/*.overlay, src/main.c, remote/{CMakeLists.txt, prj.conf, boards/*.overlay, src/main.c}}}` 패턴 — GitHub이 랩 폴더 진입 시 README.md를 자동으로 표시하도록, 기존 `doc/NN_TOPIC_LAB_KR.md(+_EN.md, +_TROUBLESHOOTING_KR/EN.md)` 형태의 `doc/` 하위 폴더를 없애고 문서를 랩 루트로 이동. 모든 README.md/README_kr.md, TROUBLESHOOTING.md/TROUBLESHOOTING_kr.md 파일 상단에는 한/영 전환 링크를 추가
- ~~Lab 03의 `reserved-memory` 영역을 새로 정의할지, `ipmmem0`를 재사용할지~~ → **해결 (2026-09-11): `ipmmem0` 재사용으로 확정** (위 "Lab 03 상세" 참고)

## 참고 (조사 근거)

- ESP32-S3-DevKitC 보드 문서: AMP 지원, procpu/appcpu 빌드 타겟, appcpu UART 미지원 명시 (https://docs.zephyrproject.org/latest/boards/espressif/esp32s3_devkitc/doc/index.html)
- IPC Service 개요/icmsg 샘플 문서 (https://docs.zephyrproject.org/latest/samples/subsys/ipc/ipc.html, https://docs.zephyrproject.org/latest/samples/subsys/ipc/ipc_service/icmsg/README.html)
- ESP32 MBOX devicetree 바인딩 (https://docs.zephyrproject.org/latest/build/dts/api/bindings/mbox/espressif,mbox-esp32.html)
- Zephyr MBOX API 헤더(`include/zephyr/drivers/mbox.h`) 및 공식 샘플 `samples/drivers/mbox`(README/src/main.c/오버레이) — `struct mbox_msg`, `mbox_callback_t`, `MBOX_DT_SPEC_GET`, `_dt` 헬퍼 함수, `vnd,mbox-consumer` 테스트 바인딩 확인 (https://github.com/zephyrproject-rtos/zephyr, `drivers/mbox/mbox_esp32.c`, `drivers/ipm/ipm_esp32.c`, `dts/xtensa/espressif/esp32s3/esp32s3_common.dtsi`, `boards/espressif/esp32s3_devkitc/esp32s3_devkitc_procpu.dts`)
- Lab 05 원본 소스: [`jyounnim/zephyr_sensor/01_AHT20_BMP280_MultiSensor`](https://github.com/jyounnim/zephyr_sensor/tree/main/01_AHT20_BMP280_MultiSensor) (사용자 제공 zip, 2026-09-04)
- Lab 01의 "IPM 활성화 = APPCPU 기동 조건" 발견은 온라인 문서가 아니라 이 세션에서의 실기 디버깅으로 확정한 것 (2026-09-06). Lab 02에서 "`ipm0`/`mbox0` 중 하나만 있으면 됨"으로 정책을 갱신한 것도 devicetree/드라이버 소스 조사 + 실기 검증으로 확정한 것 (2026-09-11)
- Lab 03의 `ipmmem0` 재사용 안전성 판단은 `drivers/mbox/mbox_esp32.c`의 `esp32_mbox_send` 구현을 직접 조사해서 확정 — `msg == NULL || msg->data == NULL`일 때 공유메모리 buffer에 대한 memcpy가 전혀 실행되지 않음을 소스 레벨에서 확인 (2026-09-11)
- Lab 04는 `zephyr,ipc-icmsg` devicetree binding 공식 문서, ICMsg backend 문서(https://docs.zephyrproject.org/latest/services/ipc/ipc_service/backends/ipc_service_icmsg.html), 공식 샘플 `samples/subsys/ipc/ipc_service/icmsg`(nRF5340 기준 main.c/prj.conf), `subsys/ipc/ipc_service/backends/ipc_icmsg.c`/`lib/icmsg.c`, `esp32s3_common.dtsi`(및 원조 `esp32_common.dtsi`)의 `shm0` 노드 발견을 근거로 설계 — ESP32 자체의 기존 icmsg 예제는 없었지만 실기 검증 완료 (2026-09-11)

---

> 이 파일은 저장소 최상위에 두고, 랩이 진행/완료될 때마다 계속 갱신됩니다. (프로젝트 문서 `claude/ipc_roadmap_KR.md`와 동일한 내용을 유지)
