# Crazyflie firmware code analysis

Read-only review of this tree (`johngmuender/fork_crazyflie-firmware`) for a NanoCockpit port onto a Crazyflie 2.1 Brushless with an AI-deck 1.1. The companion note on `johngmuender/fork_nanocockpit` is the source of the 2021.06 application behavior cited below. No application source was modified for this note. The only new file is this document.

How to read the labels:

- **[Repo-stated].** A fact that is in this repository, with a path and a line.
- **[Derived].** A conclusion from reading that code, from arithmetic on its constants, from a build performed for this note, or from a public Bitcraze page or release note outside the tree. Not a measurement on a drone.
- **[Opinion].** A judgment. Not something the repository proves.

Builds are in [section 12](#12-what-was-actually-built). Anything not listed there was not compiled. Nothing was flashed.

---

## 1. Which upstream this fork is

**[Repo-stated].** `origin` is `johngmuender/fork_crazyflie-firmware`. `HEAD` is `b65d8c5704743ca40c9eec78240b103f7d6c87f7`, commit subject "Merge pull request #1752 from bitcraze/rik/remove-attitude-reversion-print", author date 2026-11-12. `git describe --tags` prints `2026.08-104-gb65d8c57`. The default branch is `master`.

**[Derived].** GitHub's compare of `bitcraze/crazyflie-firmware` `master` to `johngmuender:master` returns `status: identical`, `ahead_by: 0`, `behind_by: 0`, same SHA `b65d8c57`. The fork was created 2026-11-15 and its parent is `bitcraze/crazyflie-firmware`. There is no fork-specific commit, defconfig, deck driver, or app. Every behavior below is upstream Bitcraze at that SHA.

**[Repo-stated].** Tag `2026.08` is `54f31e243a0b28b67efef5ba20dbb6d9890a5478` (2026-08-20, "lighthouse-fix-simultaneous-sensor-hit"). `2026.08..HEAD` is 104 commits, 69 files, +3358/−2028. Commits in that range that matter for this port:

| Commit | Date | What it changes for NanoCockpit |
| --- | --- | --- |
| `ce330b51` | 2026-10-12 | `examples/app_state_stream_aideck` sends the full `!STA` state, not a short subset |
| `518a11f4` | with #1737 | README tells you the UART1 decks this example cannot share |
| `8f7471ee` / `0f6644c6` | #1736 | ESP32 ROM bootloader baud is raised without re-init of UART2 |
| `68e62a8b` | #1742 | DSHOT motor-channel switch happens before the timer restart |
| `3761ec6d` | #1740 | Syslink DMA stays paused while the nRF flow-control line is raised |
| `d714030b` | #1744 | Log and param names are checked at link time |

**[Repo-stated].** `cf21bl` is not in tag `2021.06`. Commit `35f5092a` (2024-06-04, "Add platform configuration for CF2.1 brushless") adds the platform files. Commit `7db59d19` (2024-08-07, "Add cf21bl build target") adds the defconfig. That commit is an ancestor of tag `2024.10` (`80ee72ad`, 2024-10-16) and is not an ancestor of tag `2023.11`. The first release tag that can build this airframe is **2024.10**. This checkout is about two years of firmware past that tag, and 104 commits past **2026.08**.

**[Opinion].** Pin a port to this SHA, or to a later `master`, rather than to tag `2024.10`. The state-streamer example and the supervisor arming port are both newer than 2024.10, and both are on the path described in section 10.

---

## 2. Repository layout

**[Repo-stated].** Top level:

| Path | Role |
| --- | --- |
| `Makefile`, `Kconfig`, `configs/` | Kbuild entry and the platform fragments (`cf2_defconfig`, `cf21bl_defconfig`, `bolt_defconfig`, `tag_defconfig`, `flapper_defconfig`) |
| `src/` | Firmware: `platform/`, `hal/`, `drivers/`, `deck/`, `modules/`, `utils/`, `init/`, `lib/` |
| `vendor/` | Git submodules: FreeRTOS, CMSIS, libdw1000, Unity, CMock (`.gitmodules`) |
| `examples/` | Out-of-tree apps. The two that matter here are `app_state_stream_aideck` and `app_stm_gap8_cpx` |
| `app_api/` | Compile-only contract for the stable app API. Not a flight image |
| `tools/make/` | Linker scripts (`F405/linker/`), `oot.mk`, version script |
| `tools/kbuild/`, `scripts/kconfig/` | The Kbuild/Kconfig machinery |
| `docs/` | Building, app layer, supervisor, CRTP |
| `pixi.toml`, `bindings/`, `test/` | Reproducible host env, Python bindings, unit tests |
| `.github/workflows/CI.yml` | CI builds the platform defconfigs in `bitcraze/builder` |

There is no vendored GAP8 SDK, no ESP-IDF tree, and no NanoCockpit controller. Those live in the other fork.

**[Repo-stated].** `pixi.toml` lines 7–8 list authors "Elia Cereda" and "Arnaud Taffanel". Line 28 pins `cfclient >= 2025.12.1` as a PyPI dependency of the firmware dev environment. That is a host-tool pin, not an application.

---

## 3. Build system

### 3.1 Kbuild and platforms

**[Repo-stated].** The image is always an STM32F405-class Cortex-M4, hard-float, linked for the radio bootloader:

```5:9:Makefile
CPU                        = stm32f4
LOAD_ADDRESS_stm32f4       = 0x8000000
LOAD_ADDRESS_CLOAD_stm32f4 = 0x8004000
```

`tools/make/F405/linker/DEF_CLOAD.ld` places flash at `0x08004000`, length 1008K, RAM 128K at `0x20000000`, CCM 64K at `0x10000000`. `CLOAD=0` uses the full 1024K at `0x08000000`. There is one linker script for every platform.

**[Repo-stated].** `make cf21bl_defconfig` is the generic `%_defconfig` rule in `scripts/kconfig/Makefile`, which loads `configs/cf21bl_defconfig`. That file is five lines:

```1:4:configs/cf21bl_defconfig
CONFIG_PLATFORM_CF21BL=y

CONFIG_MOTORS_REQUIRE_ARMING=y
CONFIG_MOTORS_ESC_PROTOCOL_DSHOT=y
```

`configs/cf2_defconfig` is only `CONFIG_PLATFORM_CF2=y`. Everything else is a Kconfig default. `Kconfig` lines 60–68: `PLATFORM_CF2` selects BMI088/BMP3xx **and** MPU9250/LPS25H; `PLATFORM_CF21BL` selects BMI088/BMP3xx and `MOTORS_REQUIRE_ARMING`. Objects are chosen in `src/platform/src/Kbuild`.

**[Repo-stated].** Generated `.config` for the two defconfigs (section 12), the lines that differ in a way that matters:

| Option | `cf2` | `cf21bl` |
| --- | --- | --- |
| Platform | `CONFIG_PLATFORM_CF2` | `CONFIG_PLATFORM_CF21BL` |
| IMU drivers linked | BMI088/BMP3xx and MPU9250/LPS25H | BMI088/BMP3xx only, I2C |
| ESC protocol | `ONESHOT125` (Kconfig default, `src/modules/src/Kconfig` line 138) | `DSHOT`, 300 kHz PWM |
| Bidirectional DSHOT | off (protocol is not DSHOT) | on, RPM window 1000–3500 for 200 ms, 12 poles |
| Arming | off unless a menuconfig turns it on | required, idle thrust 7000 |
| AI-deck / CPX | `DECK_AI=y`, `ENABLE_CPX=y`, UART2 at 576000, Wi-Fi setup off | same |
| Estimator / controller | auto-select; Kalman linked; UKF not linked | same |

**[Repo-stated].** Runtime device type comes from the OTP string, matched in `platformInitConfiguration` (`src/platform/src/platform.c` lines 72–94). A miss returns 1. `src/init/main.c` lines 52–56 then sit in `while(1)` before the scheduler starts.

| Platform file | OTP token | Sensors | Motor map |
| --- | --- | --- | --- |
| `platform_cf2.c` 39–52 | `CF20` | MPU9250 + LPS25H | `motorMapDefaultBrushed` |
| `platform_cf2.c` 48–52 | `CF21` | BMI088 + BMP3xx | `motorMapDefaultBrushed` |
| `platform_cf21bl.c` 39–43 | `C21B` | BMI088 + BMP3xx | `motorMapCF21Brushless` |
| `platform_bolt.c` | `RZ10`, `CB10`, `CB11` | BMI088 on SPI | Bolt brushless maps |
| `platform_tag.c` | `RR10` | BMI088 | `motorMapNoMotors` |

**[Derived].** A `cf2.bin` on a board whose OTP says `C21B` halts in `main` and never configures a timer. It does not "almost fly" on the brushed map. The only way to get the brushed map onto that board is to override the device string at build time. Do not.

### 3.2 Toolchain

**[Repo-stated].** `docs/building-and-flashing/build.md` lines 18–28: the policy is "gcc-arm-none-eabi 10.3 or anything newer", installed with `apt-get install make gcc gcc-arm-none-eabi git python3`. Python 3.7+ is enough for the version script and the log/param name check. `pixi.toml` line 22 is stricter: `xpack-gcc-arm-none-eabi >= 13.3.1.1.1, <14`, and Python `>=3.13,<3.14`.

**[Repo-stated].** `Makefile` lines 30–31 pass `-Wno-stringop-overflow` and `-Wno-stringop-overread`. That is why GCC 13 builds this tree. The 2021.06 tree the NanoCockpit fork vendors does not have those flags and fails the same compiler (companion note, section 11).

**[Repo-stated].** Submodules are required. `vendor/FreeRTOS`, `vendor/CMSIS`, and `vendor/libdw1000` were empty in the clone used for this note until `git submodule update --init`. Unity and CMock are for the host tests, not for `make`.

### 3.3 Out-of-tree apps

**[Repo-stated].** An app is not a second flash partition. `tools/make/oot.mk` builds the **whole firmware** with `objs-y += $(OOT)` (`Makefile` line 83), merging `app-config` over the platform defconfig. The binary is still `cf21bl.bin` / `cf2.bin`, linked at `0x08004000`. `make cload` runs `python3 -m cfloader` and flashes that file as `stm32-fw`.

**[Repo-stated].** `examples/app_state_stream_aideck/Makefile` is the pattern: `CRAZYFLIE_BASE := ../..`, `OOT_CONFIG := $(PWD)/app-config`, `include tools/make/oot.mk`. `app-config` is:

```
CONFIG_APP_ENABLE=y
CONFIG_APP_PRIORITY=1
CONFIG_APP_STACKSIZE=350
```

There is no `make app_api_defconfig` target and no `APP_ENTRY` symbol. `app_api/` is a CI smoke test (`configs/app_api.conf` is `CONFIG_APP_API_TEST=y`). `docs/development/apis_versions_deprecation.md` says the symbols compiled by `app_api/src/app_main.c` are the stable set.

**[Opinion].** Put the NanoCockpit controller in a new directory next to `examples/app_state_stream_aideck` and build it with `make cf21bl_defconfig && make` from that directory. Do not set `APP=1` in the top-level Makefile. That flag existed in 2021.06 and is gone.

---

## 4. FreeRTOS tasks, priorities, stacks

**[Repo-stated].** `src/config/FreeRTOSConfig.h`: `configMAX_PRIORITIES` is 6 (usable priorities 0–5, line 93), tick is 1000 Hz (line 77), `configMINIMAL_STACK_SIZE` is `FREERTOS_MIN_STACK_SIZE`, which `src/config/config.h` line 55 sets to **150 words**. `FREERTOS_HEAP_SIZE` defaults to 30000 bytes (`config.h` lines 52–54). Static allocation is on. The timer task is priority 1 with stack `4 * 150` words. Stack overflow checking is on (`FreeRTOSConfig.h` line 85). `configMAX_TASK_NAME_LEN` is 10 (line 81), so several of the names below are truncated in the kernel.

**[Repo-stated].** Priorities and stack budgets are the macros in `src/config/config.h` lines 63–218. Stacks are in **words** (4 bytes on this M4). `150` words is 600 bytes.

| Priority | Name | Stack (words) | What it is |
| --- | --- | --- | --- |
| 5 | `STABILIZER` | 450 | 1 kHz loop: estimator, commander, controller, motors |
| 5 | `RATE_SUPERVISOR` | 150 | Loop-rate watchdog |
| 5 | `PASSTHROUGH` | 300 | USB-to-ESC 4-way (BLHeli), only while that mode is active |
| 4 | `SENSORS` | 300 | IMU read at 1 kHz, gives the stabilizer its semaphore |
| 3 | `SYSLINK` | 300 | UART to the nRF51 |
| 3 | `LH` | 300 | Lighthouse UART1 task, if that deck is present |
| 3 | `FLOW` | 300 | Flow deck, if present |
| 3 | `MR` | 300 | Multi-ranger, if present |
| 3 | `LPS` | 450 | Loco deck, if present |
| 3 | `UART2` | 150 | CPX UART transport tasks use `AI_DECK_TASK` priority, not this one; the name exists |
| 2 | `SYSTEM` | 300 | Boot, then low priority housekeeping |
| 2 | `CRTP-TX` / `CRTP-RX` | 150 / 300 | Radio packet dispatch |
| 2 | `CMDHL` | 300 | High-level commander |
| 2 | `KALMAN` | 450 | Kalman task, file is linked in the default build |
| 2 | `ERROR_UKF` | 600 | Only if `CONFIG_ESTIMATOR_UKF_ENABLE`. It is **not** in the default `.config` |
| 1 | `LOG`, `PARAM`, `MEM`, `WORKER`, `SUPERVISOR` | 300 each | TOC, worker, supervisor CRTP |
| 1 | `CPX`, `APP` | 150, and `CONFIG_APP_STACKSIZE` | CPX router; the app task |
| 0 | `PWRMGNT`, `CRTP-SRV`, `PLATFORM-SRV` | 150 | Power and the low-priority services |

There is no commander task. `commanderGetSetpoint` runs inside `stabilizerTask`.

**[Repo-stated].** `src/modules/interface/static_mem.h` is still the allocator. The comment at the top says it was added so tasks and queues are not taken from the heap. Default task stacks sit in normal RAM and the `StaticTask_t` sits in CCM. Queues sit in CCM. A second macro puts the whole stack in CCM and must not be used for DMA buffers. `app_handler.c` lines 39–49 use the default macro with `CONFIG_APP_STACKSIZE`.

**[Derived].** The two links in section 12 show what an app stack actually costs. Stock `cf21bl` BSS is 86564 bytes and CCM BSS is 55176. The state-streamer app (350-word stack) raises BSS to 88004 (+1440) and CCM to 55272 (+96). 350 words × 4 = 1400 bytes, and a `StaticTask_t` is on the order of the CCM delta. That matches the header's split. CCM on the stock image is already 84% full (55176/65536). A large app stack belongs in normal RAM, which the default macro already does. Do not move the app stack to CCM to "save RAM"; CCM is the tighter region, and RAM still has about 35 KB free on the stock `cf21bl` link.

**[Opinion].** The NanoCockpit controller, with a Kalman filter of its own and a 100-deep state history, will not fit in 350 words. Start at 800 words (`CONFIG_APP_STACKSIZE=800`, 3.2 KB) and read the high-water mark. Priority 1 is the right default: the stabilizer is priority 5 and will preempt a 100 Hz loop. A priority at or above 4 will jitter the IMU.

---

## 5. Sensors and the brushless motor path

### 5.1 IMU and pressure

**[Repo-stated].** The brushless board's only onboard sensor implementation is `SensorImplementation_bmi088_bmp3xx` (`platform_cf21bl.c` line 41). The driver is `src/hal/src/sensors_bmi088_bmp3xx.c`. It reads at `SENSORS_READ_RATE_HZ` 1000 and gives a semaphore the stabilizer waits on. The same file accepts a BMP388 chip id. There is no ICM-42688 and no BMI270 in this tree. The MPU9250 driver is compiled into `cf2` images only, for the `CF20` OTP entry.

**[Repo-stated].** Bolt and Flapper use the SPI variant (`SENSORS_BMI088_SPI`). `cf21bl` uses I2C (`CONFIG_SENSORS_BMI088_I2C=y` in the generated config). The deck-facing Bosch driver (`sensors_bosch.c`) is a separate stack for the breakout deck, not the airframe IMU.

**[Derived].** NanoCockpit's 2021.06 tree already spoke BMI088 on a Crazyflie 2.1, so the estimator inputs (gyro, accel, baro) do not need a new unit conversion. What changes is the actuator behind the controller, and the arming gate in front of it.

### 5.2 Motors

**[Repo-stated].** `motorMapCF21Brushless` (`src/drivers/src/motors_def.c` lines 807–813) is open-drain DSHOT on the four motor connectors:

| Motor | Pin | Timer |
| --- | --- | --- |
| M1 | PA1 | TIM2_CH2 |
| M2 | PB11 | TIM2_CH4 |
| M3 | PA15 | TIM2_CH1 |
| M4 | PB10 | TIM2_CH3 |

`motorMapDefaultBrushed` (lines 735–740) uses the same PA1, PB11, and PA15, and puts **M4 on PB9 / TIM4_CH4**. The brushless map moves M4 to PB10. Each brushless peripheral sets `hasPC15ESCReset`.

**[Repo-stated].** `cf21bl_defconfig` selects DSHOT. Kconfig then defaults to 300 kHz PWM and **bidirectional** DSHOT (`src/modules/src/Kconfig` lines 156–175), which the help text says is required for Bluejay ESC firmware. Arming RPM defaults: minimum 1000, maximum 3500, for 200 ms, 12 poles, "not responding" below 500 RPM. Idle thrust default is 7000 (`Kconfig` lines 228–232). The generated `.config` matches those defaults (section 12).

**[Repo-stated].** `motorsPrepareDshot` (`src/drivers/src/motors.c` lines 601–649) shifts the 16-bit thrust ratio down by 5 to get an 11-bit throttle, forces anything below `DSHOT_MIN_THROTTLE` to 0, appends the nibble checksum, and inverts the checksum when bidirectional DSHOT is on. `motorsBurstDshot` (lines 653–657) documents that TIM2_CH2 and TIM2_CH4 share DMA channel 3, so M2 is started from `DMA1_Stream6_IRQHandler` and is late by a fraction of a frame. Commit `68e62a8b` moved the channel switch to before the timer restart; the latency comment is still there.

**[Repo-stated].** Power distribution is the shared quadrotor mixer (`POWER_DISTRIBUTION_QUADROTOR` depends on `PLATFORM_CF2 || PLATFORM_CF21BL || ...`, `Kconfig` lines 267–269). Thrust curves and PID defaults for the brushless airframe are in `src/platform/interface/platform_defaults_cf21bl.h`, included from `platform_defaults.h` when `CONFIG_PLATFORM_CF21BL` is set. Per-motor `THRUST_MAX` is 0.2 N (line 65). The brushed 2.1 header uses 0.12 N. Mass default is **39.3 g**, described in the comment on lines 46–47 as the brushless airframe with propeller guards, a 350 mAh pack, and a **Lighthouse** deck. `PID_VEL_THRUST_BASE` is 30000 (line 135). Rate Kp is 200 (line 77).

**[Repo-stated].** ESC passthrough for BLHeli is `src/modules/src/vcp_esc_passthrough.c` plus `serial_4way.c`. It is a USB mode, priority 5, and it suspends sensors and syslink while it owns the signal lines. It is not on the flight path.

**[Repo-stated].** The supervisor, not the app, decides whether motors are allowed to run. `configs/cf21bl_defconfig` forces `CONFIG_MOTORS_REQUIRE_ARMING`. `docs/userguides/autonomous_flight.md` lines 33–48: auto-arm is the brushed default; the brushless and the Bolt need `supervisorRequestArming(true)` (`src/modules/interface/supervisor.h` lines 88–96). After arming, the vehicle must take off before `supervisor.prefltTimeout`, default 30 s (`platform_defaults.h`). With bidirectional DSHOT the arming state also waits until every motor reports an RPM inside the Kconfig window. `docs/userguides/autonomous_flight.md` line 29: the supervisor does **not** wait for the position estimate to converge. That is the app's job.

**[Opinion].** A port that only copies `commanderSetSetpoint` will sit on the pad with the motors stopped, and after 30 s it will be disarmed again. The first thing the new `appMain` should do, once `supervisorCanArm()` is true and the estimator the flight actually uses has settled, is request arming. The RPM check is not optional on this defconfig.

**[Repo-stated].** `MOTORS_DEFAULT_PROP_TEST_PWM_RATIO` defaults to 0, and the help text (lines 243–247) says brushless motor tests then do not spin, on purpose. The health/prop-test path will look "dead" until that value is set. That is a configuration fact, not a failed ESC.

---

## 6. Estimators and controllers

### 6.1 Estimators

**[Repo-stated].** `src/modules/interface/estimator/estimator.h` lines 31–48. The enum is auto-select, complementary, Kalman (if enabled), UKF (if enabled), and an out-of-tree slot. `src/modules/src/estimator/estimator.c` line 19 sets `DEFAULT_ESTIMATOR` to complementary. Kconfig's choice defaults to `CONFIG_ESTIMATOR_AUTO_SELECT` (`src/modules/src/Kconfig` around line 99). Both generated configs set `CONFIG_ESTIMATOR_KALMAN_ENABLE=y` and leave `CONFIG_ESTIMATOR_UKF_ENABLE` unset. The UKF sources are `estimator_ukf.c` / `estimator_ukf.h` ("error-state unscented Kalman filter"). `src/modules/src/estimator/Kbuild` line 3 compiles `estimator_ukf.o` only when that option is on. The `cf21bl` build directory contains `estimator_kalman.o` and `estimator_complementary.o` and does not contain `estimator_ukf.o`.

**[Repo-stated].** Decks can demand Kalman. Flow v1, flow v2, and Lighthouse set `requiredEstimator = StateEstimatorTypeKalman` (`flowdeck_v1v2.c` lines 204 and 251, `lighthouse.c` line 95). `system.c` calls `deckGetRequiredEstimator()` and passes it into `stabilizerInit`. The AI-deck driver does not set `requiredEstimator`. Boot with only an AI-deck stays on the complementary filter until a parameter or a deck says otherwise.

**[Repo-stated].** The Kalman state is 9 floats: position, body velocity, attitude error (`kalman_core.h`, `KC_STATE_DIM`). Prediction in the Kalman task is 100 Hz; the stabilizer still copies the latest state at 1 kHz. Measurement models under `src/modules/src/kalman_core/` cover flow, ToF, lighthouse sweep angles, TDoA, TWR distance, external position, external pose, yaw error, and absolute height. Complementary attitude runs at 250 Hz and its position estimator at 100 Hz (`estimator_complementary.c`).

**[Derived].** The NanoCockpit follow loop consumes a position and a velocity. On complementary-only, with no flow, lighthouse, loco, or external pose, those logs stay near zero. The controller will then command a velocity toward a subject it cannot place in the world. A flight needs Kalman plus one position source. Section 7 says which sources coexist with the AI-deck.

### 6.2 Controllers

**[Repo-stated].** `src/modules/interface/controller/controller.h`: auto-select, PID, Mellinger, INDI, Brescianini, Lee, and an out-of-tree slot. `controller.c` defaults auto-select to PID. The same five are linked for `cf2` and `cf21bl`. There is no brushless-only controller. Selection at runtime is the parameter `stabilizer.controller`.

| Controller | Files | What a setpoint has to contain |
| --- | --- | --- |
| PID | `controller_pid.c`, `attitude_pid_controller.c`, `position_controller_pid.c` | Attitude at 500 Hz, position at 100 Hz. This is the one the NanoCockpit velocity setpoints already target |
| Mellinger | `controller_mellinger.c` | Position modes absolute, geometric attitude. Uses the mass and the thrust curve |
| INDI | `controller_indi.c`, `position_controller_indi.c` | Outer position loop plus INDI attitude |
| Brescianini | `controller_brescianini.c` | Force/torque output, 100 Hz position block |
| Lee | `controller_lee.c` | Force/torque, geometric attitude at 500 Hz |

**[Repo-stated].** The stabilizer rate is `RATE_MAIN_LOOP` 1000 Hz, `ATTITUDE_RATE` 500 Hz, `POSITION_RATE` 100 Hz (`stabilizer_types.h`). `stabilizer.c` order, per tick: wait for the sensor semaphore, `sensorsAcquire`, estimator, `commanderGetSetpoint`, collision avoidance, supervisor override, controller, then `powerDistribution` and, on DSHOT, `motorsBurstDshot`. Motors are skipped unless `supervisorAreMotorsAllowedToRun()`.

**[Repo-stated].** `commander.h` lines 35–53:

```35:53:src/modules/interface/commander.h
#define COMMANDER_PRIORITY_DISABLE   0
#define COMMANDER_PRIORITY_LOWEST    1
#define COMMANDER_PRIORITY_HIGHLEVEL 1
#define COMMANDER_PRIORITY_CRTP      2
#define COMMANDER_PRIORITY_EXTRX     3
...
void commanderSetSetpoint(setpoint_t *setpoint, int priority);
void commanderRelaxPriority(void);
```

`commander.c` lines 81–89: a call wins when `priority >=` the current one, and any priority above `COMMANDER_PRIORITY_HIGHLEVEL` stops the high-level planner. There is no priority above 3 in the header. A larger integer still wins the comparison; nothing clamps it.

**[Derived].** In the 2021.06 header, `COMMANDER_PRIORITY_CRTP` was 1 and `COMMANDER_PRIORITY_EXTRX` was 2. NanoCockpit's `frontnet_config.h` uses `COMMANDER_PRIORITY_CRTP + 1` so the follow loop beats the radio. Recompiled against this header, that expression becomes 3, which still beats CRTP (2) and ties the external-receiver priority. A hardcoded `1` meaning "the old CRTP level" now loses to the radio and ties the high-level commander. Keep the macro. Do not keep the integer.

**[Repo-stated].** `setpoint_t` (`stabilizer_types.h` lines 245–274) has `mode` as a `setpoint_mode_t` (`modeDisable`, `modeAbs`, `modeVelocity` on x, y, z, roll, pitch, yaw, quat) plus `jerk`. The 2021.06 struct had no `jerk` member. Zero-initializing the struct still means "disabled", which is what a hover-on-timeout path wants.

**[Opinion].** Stay on PID for the first port. Mellinger, INDI, Brescianini, and Lee all consume `CF_MASS`, and the default 39.3 g is a lighthouse setup, not an AI-deck setup. The AI-deck is a different mass. PID's `PID_VEL_THRUST_BASE` of 30000 will hover the wrong vehicle too, but it is one parameter (`pid_vel` / the velocity thrust base) rather than a model.

---

## 7. Decks, AI-deck, CPX, and the UART1 conflict

### 7.1 Discovery

**[Repo-stated].** Decks are discovered by one-wire memory (`src/deck/backends/deck_backend_onewire.c`), by the forced-name backend, and by the DeckCtrl backend. A driver is a `DECK_DRIVER()` row matched on VID/PID, then on name (`src/deck/core/deck_info.c` `findDriver`). `CONFIG_DECK_FORCE` (`src/deck/backends/Kconfig`) accepts a colon-separated list of driver names and synthesizes a deck with a name and no VID/PID. NanoCockpit's Makefile added `-DDECK_FORCE=bcAI`. The equivalent here is `CONFIG_DECK_FORCE="bcAI"`, and it is only needed when the one-wire memory cannot be read. A real AI-deck 1.1 enumerates as VID `0xBC`, PID `0x12`, name `bcAI` (`aideck.c` lines 370–376).

**[Repo-stated].** `checkPeriphAndGpioConflicts` (`deck_info.c` lines 187–224): if two decks share a non-bus peripheral, or share a GPIO, the firmware prints an error and sets the deck count to **zero**. No deck initializes, including the ones that did not conflict. I2C and SPI are the only shared buses.

### 7.2 What the AI-deck driver does now

**[Repo-stated].** `aideck.c` lines 25–27 and 312–338. `aideckInit`:

1. Holds `DECK_GPIO_IO4` low. That is the shared reset for the GAP8 and the ESP32.
2. Starts the CPX UART transport, the internal router, the external router, and the CPX task.
3. Releases `IO4` and leaves it pulled up.
4. Does **not** call `uart1Init`. The file includes `uart1.h` and does not use it on this path.
5. Wi-Fi setup is compiled out. `src/deck/drivers/src/Kconfig` lines 23–31 default to `DECK_AI_WIFI_NO_SETUP`, and both generated configs keep that default.

The driver claims `DECK_USING_UART2` and GPIOs `IO_1` and `IO_4` (lines 375–376).

**[Repo-stated].** Two bootloaders, both owned by this driver:

| Target | How the STM32 enters the bootloader | Transport after that |
| --- | --- | --- |
| GAP8 | `resetGap8ToBootloader` (lines 202–213) sends CPX `CPX_T_ESP32` / `CPX_F_SYSTEM` command `ESP32_SYS_CMD_RESET_GAP8` (0x10). The ESP32 is expected to reset the GAP8 into its bootloader | CPX `CPX_F_BOOTLOADER` (0x0F): `GAP8_BL_CMD_START_WRITE` (0x02) then `GAP8_BL_CMD_MD5` (0x04), lines 94–95 and 160 |
| ESP32 | `resetEspToBootloader` (lines 231–257) drops the CPX UART, sets UART2 to 115200, holds `IO4` low and `IO1` low (the strapping pin), releases `IO4`, then releases `IO1` | `src/modules/src/esp_deck_flasher.c` speaks the ROM bootloader (SLIP) and then raises the baud. Commit `8f7471ee` is that baud change |

**[Derived].** GAP8 flashing only works if the ESP32 image understands `ESP32_SYS_CMD_RESET_GAP8` and forwards `CPX_F_BOOTLOADER`. That is the Bitcraze AI-deck ESP firmware. The NanoCockpit NINA image, per the companion note, routes SPI payloads to Wi-Fi and does not implement an STM32 route. With that image, `cfclient`'s GAP8 flasher will not reach the bootloader. Flash the GAP8 with JTAG, which is how NanoCockpit already does it, or put the Bitcraze ESP image back before using the deck-memory flasher.

### 7.3 CPX, and how it differs from NanoCockpit's CPX

**[Repo-stated].** `src/modules/interface/cpx/cpx.h` lines 33–74:

| Field | This firmware |
| --- | --- |
| Targets | `STM32=1`, `ESP32=2`, `WIFI_HOST=3`, `GAP8=4` |
| Functions | `SYSTEM=1`, `CONSOLE=2`, `CRTP=3`, `WIFI_CTRL=4`, `APP=5`, `TEST=0x0E`, `BOOTLOADER=0x0F` |
| Version bits | `CPX_VERSION` is 0 |
| Max payload | `CPX_MAX_PAYLOAD_SIZE` 100, of which 2 bytes are the packed route |

There is no `CPX_F_STREAMER`. NanoCockpit adds `CPX_F_STREAMER = 0x06` on the GAP8/ESP SPI link (companion note, section 6.2). The value 6 does not collide with `APP` (5) or `BOOTLOADER` (0x0F). It is simply not decoded on this STM32. Camera frames never cross this UART.

**[Repo-stated].** The UART framing is not the SPI framing. `cpx_uart_transport.c` lines 123–169: a frame is `0xFF`, a length, the packed route plus payload, and an XOR of every preceding byte. A length of 0 is the CTS mark (`0xFF 0x00`), not a packet. The TX task (lines 191–196) sends that mark every 100 ms until the peer has answered with CTS. Default baud is `CONFIG_CPX_UART2_BAUDRATE` 576000 (`src/hal/src/Kconfig`).

NanoCockpit's SPI CPX, from the companion note: a 2-byte length, the same style of packed route byte, DMA up to 4092 bytes, 30 MHz when the host-to-GAP8 direction is discarded and 7.2 MHz when it is not. That link is GAP8 master to ESP32 slave. This firmware never sees it.

**[Repo-stated].** `examples/app_stm_gap8_cpx/src/stm_gap8_cpx.c` lines 49–64 are the supported STM32↔GAP8 pattern in this tree: `cpxRegisterAppMessageHandler` for packets with destination `CPX_T_STM32` and function `CPX_F_APP`, and `cpxSendPacketBlocking` toward `CPX_T_GAP8` / `CPX_F_APP`. The example's README points at `bitcraze/aideck-gap8-examples`, not at NanoCockpit.

**[Derived].** Three different "CPX"s are easy to mix:

1. NanoCockpit GAP8 ↔ ESP32 over SPI, MTU 4092, optional `STREAMER=6`, used for images.
2. This firmware's STM32 ↔ ESP32 over UART2, MTU 100, 576000 baud, XOR CRC, CTS handshake, used for CRTP bridging, Wi-Fi control, and the GAP8 bootloader.
3. NanoCockpit STM32 ↔ GAP8, which is **not CPX**. It is UART1 at 115200 with `!STA` / `!RNG` / `!TOF` one way and a 4-byte header plus a 20-byte inference the other way (companion note, section 6.1). `frontnet_aideck_protocol.c` still says `TODO: replace this with CPX`.

A port that "switches the STM32 app to CPX" without changing the GAP8 binary will talk to nobody. The GAP8 binary still speaks (1) and (3).

**[Repo-stated].** `cpxlink` can publish CRTP to `CPX_T_WIFI_HOST` with function `CPX_F_CRTP`, so a Wi-Fi client can be a second CRTP link. That path needs the Bitcraze ESP image and a completed CTS handshake. It is unrelated to NanoCockpit's TCP port 5000 streamer.

### 7.4 UART1, flow, multi-ranger, lighthouse

**[Repo-stated].** Who claims which peripheral:

| Driver | Claims | Citation |
| --- | --- | --- |
| AI-deck `bcAI` | UART2, IO1, IO4 | `aideck.c` 375–376 |
| Lighthouse `bcLighthouse4` | UART1, and it requires Kalman | `lighthouse.c` 94–95. The task calls `uart1Init(230400)` (`lighthouse_core.c` line 575) |
| Flow v1 `bcFlow`, Flow v2 `bcFlow2` | SPI and I2C, GPIO IO3, Kalman | `flowdeck_v1v2.c` 202–204 and 249–251. PMW3901 is SPI. The range sensor is the zRanger / zRanger2 VL53 driver |
| Multi-ranger `bcMultiranger` | I2C only | `multiranger.c` 201–202. Five VL53L1x, XSHUT on a PCA95x4 |
| Loco `bcLoco` | SPI, and by default GPIO IO1 plus the UART1 pins PC10/PC11 used as GPIO | `locodeck.c` 671–674 |
| Buzzer | UART2 | `buzzdeck.c` |
| Active marker | I2C | no UART |

**[Derived].** AI-deck plus Flow v2 is legal: SPI/I2C versus UART2, and IO3 versus IO1/IO4. AI-deck plus multi-ranger is legal: I2C is a shared bus. AI-deck plus Lighthouse is legal **at the deck layer**, because one wants UART2 and the other wants UART1. AI-deck plus Loco on the default pins is not legal: both use IO1, the conflict check drops every deck, and the vehicle boots with no AI-deck and no ranging. AI-deck plus the buzzer is not legal: both want UART2.

**[Repo-stated].** The UART1 conflict that NanoCockpit actually hits is one level up. `examples/app_state_stream_aideck/README.md` lines 9–12: the app owns UART1 at 115200 and "cannot be used together with other decks that use UART1." The deck driver does not know the app has called `uart1Init`. Lighthouse will also call `uart1Init(230400)` (`lighthouse_core.c` line 575). Two owners, two baud rates, no Kconfig error.

**[Repo-stated].** UART2 DMA is `DMA1_Stream6` (`src/drivers/interface/uart2.h` lines 41–44). DSHOT M1 is also `DMA1_Stream6` (`motors.c` line 965, "M1"). `uart2.c` lines 152–155 therefore skip `uart2DmaInit` when `CONFIG_MOTORS_ESC_PROTOCOL_DSHOT` is set. `uart2SendDataDmaBlocking` (lines 224–226) falls back to the interrupt sender. CPX uses `uart2SendData` (`cpx_uart_transport.c` lines 193, 211, 233), which is the interrupt path, so CPX still transmits on a brushless build. It just does not use DMA.

**[Repo-stated].** UART1 DMA is `DMA1_Stream3`, channel 4 (`uart1.h` lines 43–46). The four DSHOT streams are 6, 7, 5, and 1 (`motors.c` lines 929–986). They do not overlap Stream3.

**[Derived].** The state streamer's `uart1SendDmaIfAvailable` is compatible with cf21bl DSHOT. The thing that is not compatible with DSHOT is UART2 DMA, and the firmware already routes around that. Do not "fix" CPX by re-enabling UART2 DMA on this platform; that stream is the M1 DSHOT burst.

**[Opinion].** For a brushless + AI-deck flight that still uses the NanoCockpit UART protocol, the position source that does not steal UART1 is the Flow deck (or an external pose over the radio). Lighthouse is the better position source and it takes the same UART the `!STA` frames need. Pick one, or move the state/inference link onto CPX and change the GAP8 at the same time. Do not do half of that.

---

## 8. CRTP, radio, log, param

**[Repo-stated].** `CRTP_PROTOCOL_VERSION` is **12** (`src/modules/interface/crtp.h` line 33). `CRTP_MAX_DATA_SIZE` is 30. The on-wire packet is a 1-byte header plus 30 data bytes. Ports include console 0x00, param 0x02, setpoint 0x03, log 0x05, localization 0x06, generic setpoint 0x07, high-level 0x08, **supervisor 0x09**, platform 0x0D. The supervisor port was not in the 2021.06 header. Commit `5821eb00` (2026-02-24, "Migrate emergency stop handling to CRTP supervisor port") is the change that put protocol 12's emergency stop on that port.

**[Repo-stated].** The STM32 does not speak USB to a Crazyradio. `commInit` installs `radiolink` as the CRTP link. `radiolink` exchanges `SYSLINK_RADIO_*` packets with the nRF51 over the syslink UART (`src/hal/interface/syslink.h`, `src/hal/src/syslink.c`). The radio address, channel, and rate in `config.h` lines 221–223 are the defaults the nRF is configured with: channel 80, 2 Mbit/s, address `0xE7E7E7E7E7`. `system.c` lines 398–408 store the nRF version string and print it. Nothing compares it to the STM32 version. A mismatch does not abort boot.

**[Repo-stated].** This tree does not contain the string "Crazyradio 2.0". Flashing docs (`docs/building-and-flashing/build.md` around lines 187–210) say "Crazyradio" and "Crazyradio PA".

**[Derived], from Bitcraze's Crazyradio 2.0 documentation, not from this tree.** The "Crazyradio PA compatible USB and Radio protocol" page for the Crazyradio 2.0 firmware says firmware 5.0 and later enumerates as USB VID `0x1915`, PID `0x7777` and implements a superset of the Crazyradio PA USB protocol, which is the protocol `cflib/drivers/crazyradio.py` opens. The 5.1 release is marked retired on the Crazyradio 2.0 release list; 5.2 and later are the ones that list says to run. The newest tag observed while writing this note is **5.5**. A Crazyradio 2.0 on firmware older than the 5.0 USB-version change reported the Zephyr version as `bcdDevice` and is the wrong firmware to pair with this cflib.

**[Repo-stated].** `pixi.toml` line 28 requires `cfclient >= 2025.12.1`. The `pyproject.toml` of `bitcraze/crazyflie-clients-python` at tag `2025.12.1` depends on `cflib~=0.1.31` (that is, `>=0.1.31` and `<0.2`). A current pip resolve of that specifier installs the newest 0.1.x. The newest release on the cflib release list while writing this note is **0.1.34** (30 Sep 2026). Release 0.1.32 is the one whose notes include the supervisor subsystem (PR #581) and the "protocol &lt; 12 falls back to the legacy port" change (PR #597, merged 2026-04-10).

**[Repo-stated].** This firmware still accepts arming on the old platform port and prints a deprecation warning (`platformservice.c` lines 137–148). The native call is supervisor port command `CMD_ARM_SYSTEM` (`crtp_supervisor.c` lines 158–168).

**[Opinion].** Use Crazyradio 2.0 firmware **5.2 or newer** (5.5 is the current tag) and **cflib 0.1.32 or newer**, which a fresh install of cfclient 2025.12.1 will pull. That pair opens the dongle and can arm a brushless airframe on the supervisor port. NanoCockpit's pin `cflib==0.1.20.1` (2022-09-07) looks for the same USB PID, so it may open a Crazyradio 2.0 that is already on the PA-compatible firmware, and it has no supervisor client. Do not point `make cload` at that virtualenv. Do not flash the nRF51 with the 2021.06 image the NanoCockpit README asks for; ship the nRF image from the same Bitcraze release as this STM32 SHA. The STM32 will boot either way and the radio will not.

### Log and param

**[Repo-stated].** Log variables are `LOG_GROUP_START` / `LOG_ADD` rows in the `.log.*` linker section (`src/modules/interface/log.h`). The log task serves `CRTP_PORT_LOG`. Limits in `src/modules/src/log.c`: 26 bytes of payload in a log item line, 128 operations, 16 blocks. The `cf21bl` link reports **328 parameters and 576 log variables**. The `cf2` link reports 336 and 572. Param rows use the same linker-section pattern (`param.h`), with `PARAM_PERSISTENT` stored under the `prm/` prefix. `d714030b` checks the names at build time, so a new app group that collides or breaks the length rules fails the link instead of failing the TOC later.

**[Repo-stated].** The compressed state the NanoCockpit `!STA` frame wants is the log group `stateEstimateZ` (`stabilizer.c` lines 822–888): `x,y,z` in mm, `vx,vy,vz` in mm/s, `ax,ay,az` in mm/s², `quat` as the `quatcompress.h` uint32, `rateRoll/Pitch/Yaw` in milliradians/s. Names and units match the struct in `examples/app_state_stream_aideck/src/state_streamer.c` lines 48–78.

---

## 9. App-layer API, and what moved since 2021.06

**[Repo-stated].** `src/modules/interface/app.h` is still `appInit` plus `appMain`. `app_handler.c` lines 43–61: the weak `appInit` creates a task named `APP` at `CONFIG_APP_PRIORITY` with `CONFIG_APP_STACKSIZE` words, the task blocks in `systemWaitStart`, then calls `appMain`. `system.c` lines 148–150 call `appInit` only when `CONFIG_APP_ENABLE` is set. `docs/userguides/app_layer.md` lines 13–18: if you define your own `appInit`, the firmware does **not** create the task and does **not** call `appMain`. `appInit` must return.

**[Derived].** NanoCockpit's `frontnet_main.c` defines `appInit` and, in the 2021.06 tree, that was the right entry because `APP=1` only defined `APP_ENABLED` and the weak default was the thing being overridden. The same override still works here **if** `appInit` creates every task the controller needs and returns. The smaller change is to rename the body to `appMain` and delete the custom `appInit`, so the weak one in `app_handler.c` owns the task. Do one or the other. An `appInit` that only creates queues and returns will boot a drone whose controller never runs.

Other deltas that a straight recompile of the 2021.06 app will hit:

| 2021.06 app | This tree | What to do |
| --- | --- | --- |
| `APP=1` adds `-DAPP_ENABLED` | `CONFIG_APP_ENABLE=y` in `app-config` | Change the build, not a `#ifdef` |
| `COMMANDER_PRIORITY_CRTP` is 1 | It is 2. High-level is the new 1. Extrx is 3 | Keep `COMMANDER_PRIORITY_CRTP + 1`. Drop raw integers |
| `commanderNotifySetpointsStop()` | Not in `commander.h`. `commanderRelaxPriority()` is (lines 51–53) | Call the new function when giving the radio back |
| `setpoint_t` without `jerk` | `jerk` and a named `setpoint_mode_t` | `memset` the setpoint. Set `mode.x/y/z = modeVelocity` and `velocity_body = false` for the follow law |
| `APPCHANNEL_MTU` 31 | 30, matching `CRTP_MAX_DATA_SIZE` (`app_channel.h`) | Only matters if the app uses the app channel. Frontnet's radio path does not |
| `static_mem.h`, `workerSchedule` | Both still present. `worker.h` is not in the `app_api` smoke list | Linking still works. The static-UART-buffer bug in the old `aideck.c` (companion note, section 6.1) is avoided by not using that file |
| `stateGet()` | No such function | Read `stateEstimateZ` through `logGetVarId` / `logGetInt`, which is what `state_streamer.c` lines 95–132 do |
| Own copy of `aideck.c` that calls `uart1Init(115200)` and parses inference in a static buffer | Stock `aideck.c` owns UART2 and CPX. UART1 is free until the app takes it | Do not replace `src/deck/drivers/src/aideck.c`. Add the UART1 protocol in the app |
| No arming API | `supervisorRequestArming` | Required on this defconfig. See section 5.2 |
| Battery cutoff parameter that nothing reads | Supervisor low-battery at 3.35 V, critical at 3.0 V (`platform_defaults_cf21bl.h` lines 37–38) | Delete the dead parameter. Use the supervisor |

**[Repo-stated].** `examples/app_state_stream_aideck/src/state_streamer.c` is already the transmit half of NanoCockpit's UART protocol, copyright 2026, and the README says the format comes from NanoCockpit. It sends `!STA` plus a CRC32 at 100 Hz on UART1 at 115200. It does not receive the inference frame, it does not send `!RNG` or `!TOF`, and it does not keep a timestamped history. The README says the fields are read one by one from the log, so the message is not one estimator tick. The companion note's history queue (`STATE_FWD_HISTORY_COUNT`, 100 deep, peeked by the inference timestamp) is the piece this example left out on purpose.

**[Opinion].** That example is the right base for the port. Extending it is a smaller change than rebasing the 2021.06 `aideck.c` onto this driver. The history queue is worth putting back if the experiment is the paper's latency table. It is not required to see the first closed-loop hover.

---

## 10. Porting plan

This is the concrete sequence. It assumes the GAP8 image stays the NanoCockpit PULP-Frontnet build (UART protocol unchanged, SPI CPX to the ESP unchanged) and the flight controller becomes an out-of-tree app on this firmware, platform `cf21bl`.

### 10.1 What stays outside this repository

**[Opinion], from the companion note plus the drivers above.**

- GAP8: keep NanoCockpit's PMSIS image. It still wants `!STA` at 115200 on the GAP8 UART, which is the STM32's UART1, and it still wants its SPI CPX to the ESP32 for the Wi-Fi stream. Nothing in this firmware replaces that image.
- ESP32: two consistent choices.
  - **Bitcraze AI-deck ESP firmware.** STM32 CPX on UART2 completes the CTS handshake. `cfclient` can flash the ESP and, through it, the GAP8. Wi-Fi is whatever that image does. NanoCockpit's TCP-5000 viewer will not speak to it, and `CPX_F_STREAMER` does not exist on this STM32 anyway. The camera path would have to be the Bitcraze one.
  - **NanoCockpit NINA image.** Wi-Fi streaming stays as the paper describes (TCP 5000, SSID in that image's `sdkconfig`). The STM32 CPX TX task will send `0xFF 0x00` forever, because that image does not answer CTS (`cpx_uart_transport.c` lines 191–196). The task is priority 1, so the stabilizer still runs. Do not also let that ESP print debug bytes on the STM32 UART: the RX task `ASSERT`s on a bad CRC (`cpx_uart_transport.c` line 164) and a failed assert halts the firmware. GAP8 flashing from `cfclient` will not work; use JTAG.
- Do not disable `CONFIG_DECK_AI` to "get rid of CPX" while a NanoCockpit ESP is installed. The driver is also what releases the GAP8/ESP reset on IO4 (lines 319–330). Leave the driver in. The CTS spin is the cheaper problem.

### 10.2 The STM32 app

1. New directory, copied from `examples/app_state_stream_aideck`. `Makefile` includes `tools/make/oot.mk`. `app-config` sets `CONFIG_APP_ENABLE=y`, `CONFIG_APP_PRIORITY=1`, `CONFIG_APP_STACKSIZE=800` to start, and does **not** set a platform. The platform comes from `make cf21bl_defconfig`.
2. Keep `appMain`. Do not define `appInit` unless the task is created by hand.
3. Transmit side: keep `state_streamer.c`'s `!STA` layout, header, CRC32, 100 Hz, `uart1Init(115200)`. It already matches the GAP8 parser in the companion note.
4. Receive side: add the inference parser the old `aideck.c` had (header `90 19 08 32`, 20-byte payload, no CRC). Copy the bytes out of the UART buffer before any queue handoff. The old code parsed a single static buffer from the worker, which the companion note flags as a use-after-reuse. A 1-deep queue of a `struct`, filled in the UART path, is the fix.
5. Optional, and the one that reproduces the paper's timestamp: a ring of recent `!STA` payloads keyed by the FreeRTOS tick, written when the frame is sent, read when the inference comes back carrying that tick. The stock log read is not that ring. The 2021.06 stabilizer patch that pushed a snapshot at the estimator rate is not in this tree; the log group is the supported substitute.
6. Controller: the velocity law from `frontnet_ctrl.c` can move across with the setpoint changes in section 9. Priority `COMMANDER_PRIORITY_CRTP + 1`. On the 750 ms inference timeout, send a zero-velocity setpoint rather than going silent: silence trips the commander watchdog (`docs` and `supervisor.c` describe a 500 ms warning and a 2000 ms cut). The old app already hovers on that timeout; keep the behavior, and keep the packets coming.
7. Before any nonzero setpoint that should leave the ground: wait until the position estimate you are actually using has settled, call `supervisorRequestArming(true)`, and command a takeoff-class setpoint inside `supervisor.prefltTimeout` (30 s). From the host, the same arming call is `cf.supervisor.send_arming_request(True)` in cflib ≥ 0.1.32.
8. Give the vehicle a position. Flow deck (SPI/I2C) plus this UART1 app is the combination the deck matrix allows. Lighthouse plus this UART1 app is not. External pose over CRTP localization also works and uses no deck UART.
9. Set `stabilizer.estimator` to Kalman when a position source is attached. The AI-deck alone will not flip the estimator.
10. Set the vehicle mass if you leave PID. The 39.3 g default is the wrong aircraft once the AI-deck is on. `platform_defaults_cf21bl.h` lines 49–52 honor `CONFIG_MODIFIED_CF_MASS` in milligrams.
11. Build from the app directory: `make cf21bl_defconfig && make -j`. Flash `build/cf21bl.bin` as `stm32-fw` with cfclient/cfloader from the cflib in section 8. The image is the whole firmware. There is no "app only" address.
12. Do not flash a `cf2.bin`. `main` halts when the OTP token is not `CF20` or `CF21` (`main.c` lines 52–56, `platform_cf2.c` lines 39–52).

### 10.3 Host versions

| Piece | Version to use | Why |
| --- | --- | --- |
| This firmware | `cf21bl` config of SHA `b65d8c57` (or later `master`) | First tag with the platform is 2024.10; this SHA has the state streamer and protocol 12 |
| nRF51 | The image from the **same** Bitcraze release as the STM32, not 2021.06 | Syslink is a pair. This STM32 only prints the version (`system.c` 408) |
| Crazyradio 2.0 firmware | **≥ 5.2**, current tag seen is 5.5. Not 5.1 | PA-compatible USB, VID/PID `1915:7777` |
| cflib | **≥ 0.1.32**, current release seen is 0.1.34 | Supervisor port. 0.1.20.1 does not have it |
| cfclient | **≥ 2025.12.1**, which is what `pixi.toml` line 28 already requires | Depends on `cflib~=0.1.31`, so a fresh install resolves to a 0.1.x that includes 0.1.32+ |
| GAP8 SDK | Unchanged from NanoCockpit: GAP SDK 3.8.1 | This repo does not build the GAP8 |
| ESP-IDF | Only if you keep the NanoCockpit NINA image: the lockfile's 5.3.3, not the root README's 5.3.1 (companion note) | This repo does not build the ESP32 |

**[Opinion].** Order on the bench: confirm `cfclient` from that pin can connect and read `stateEstimateZ` with the stock `cf21bl` image and the motors disarmed. Then flash the app image and confirm `!STA` bytes leave UART1 (a logic analyzer, or the GAP8 log) before `enable`-style control is turned on. Then arm. The old `frontnet.enable_control` parameter has to be reimplemented; it is not in this tree.

---

## 11. Gaps and likely bugs

1. **DSHOT wait loop cannot fail.** `motors.c` lines 611–615: if the DMA stream does not disable, the `while` spins, and the body is `ASSERT(true)`, which passes. The comment above it says this is a DSHOT state-machine failure. On a failure the stabilizer task, priority 5, never returns. **[Repo-stated]** the `ASSERT(true)`. **[Opinion]** that is a stuck motor task, not a detected fault. The recent M2 timer fix (`68e62a8b`) is evidence this state machine has already needed patches.

2. **CPX UART parser trusts the length and then asserts.** `cpx_uart_transport.c` lines 151–164 read `payloadLength` and copy that many bytes into `payload[CPX_UART_TRANSPORT_MTU]` (100) with no comparison, then `ASSERT` the CRC. A `0xFF` followed by a large length walks off the struct. A bad CRC halts the firmware. **[Repo-stated]** the missing check and the assert. **[Opinion]** this is acceptable against a Bitcraze ESP image and a bad idea against a NanoCockpit ESP image that ever transmits on that UART.

3. **CPX sync does not time out.** Lines 191–196 send CTS forever if the peer never answers. Priority is the AI-deck priority, 1 (`config.h` line 100). Flight tasks still run. **[Derived].** It will show up as a steady 100 ms UART2 transmit on a brushless deck whose ESP is the NanoCockpit image.

4. **UART2 DMA and DSHOT M1 share `DMA1_Stream6`.** Handled by compiling the DMA init out (`uart2.c` 152–155). CPX uses the IRQ sender, so this is a throughput limit, not a dropped feature. UART1 DMA (`DMA1_Stream3`) does not collide. **[Repo-stated].**

5. **M1 and M2 share a DMA channel.** M2 is started from M1's completion interrupt (`motors.c` 653–656). **[Repo-stated].** **[Opinion]** the residual latency is small next to a 1 kHz loop and is still a reason not to assume the four motors are sample-aligned.

6. **The stock `!STA` sender is not a snapshot, and it does not receive.** `state_streamer.c` lines 92–94 and the README. No inference RX, no `!RNG`, no `!TOF`, no history. **[Repo-stated].** The closed-loop controller is not in this repository at all. **[Derived].**

7. **AI-deck plus Loco, and AI-deck plus buzzer, initialize zero decks.** GPIO IO1 and UART2 respectively (`deck_info.c` 211–223). A failed deck does not fall back to "the AI-deck anyway". **[Repo-stated].**

8. **Lighthouse plus the UART1 app is silent corruption, not a deck error.** The deck matrix allows it. The app README forbids it. Two baud rates on one UART. **[Derived].**

9. **Default estimator with only an AI-deck is complementary.** No position. The follow law then commands from zeros. **[Derived].**

10. **Default mass is a lighthouse aircraft.** 39.3 g, `platform_defaults_cf21bl.h` lines 46–52. An AI-deck changes it. PID hover thrust 30000 was identified on the default mass. **[Repo-stated]** the comment. **[Opinion]** weigh the vehicle before using any controller that consumes `CF_MASS`.

11. **UKF is source-only on the default configs.** Enabling it costs a 600-word stack and is not required for this port. **[Repo-stated].**

12. **Prop-test and battery-test PWM default to 0 on purpose** for brushless (`Kconfig` 238–259). A "motors do not spin in the health tab" report is that default. **[Repo-stated].**

13. **`configMAX_TASK_NAME_LEN` is 10** (`FreeRTOSConfig.h` line 81) while names such as `RATE_SUPERVISOR` are longer. Debug listings truncate. Not a flight bug. **[Repo-stated].**

14. **No Crazyradio 2.0 support is implemented here**, because the dongle never talks to the STM32. The gap is on the host pin, and section 8 is the fill. **[Repo-stated]** the absence. **[Derived]** the versions.

15. **GCC 13 is papered over, not cleaned up.** `-Wno-stringop-overflow` (`Makefile` line 31) is why this tree links on Ubuntu 24.04's `gcc-arm-none-eabi` 13.2.1 and the vendored 2021.06 tree does not. **[Repo-stated].** **[Opinion]** leave the flag. Do not copy the 2021.06 sources back in without it; the companion note's build died on `quatdecompress` in `crtp_commander_generic.c` and `crtp_localization_service.c`.

---

## 12. What was actually built

Environment: Ubuntu 24.04.5, `arm-none-eabi-gcc (15:13.2.rel1-2) 13.2.1 20231009`, Python 3.12.3. Submodules `vendor/FreeRTOS`, `vendor/CMSIS`, and `vendor/libdw1000` were checked out at the gitlinks in `.gitmodules`. Unity and CMock were not. Outputs were written under `/tmp` and under the example's `build/` directory; the example `build/` was deleted afterward and is not part of the change set. **Nothing was flashed. No radio, no drone, no debugger.**

| Build | Command | Result |
| --- | --- | --- |
| cf21bl | `make KBUILD_OUTPUT=/tmp/cf-build-cf21bl cf21bl_defconfig && make -j4` | Exit 0. `cf21bl.bin`. Version string `Build 104:b65d8c570474 (2026.08 +104) CLEAN`. Flash 297804/1032192 (29%), text 289784, data 8020. RAM 94584/131072 (72%), bss 86564. CCM 55176/65536 (84%). 328 parameters, 576 log variables. |
| cf2 | same, `cf2_defconfig`, output `/tmp/cf-build-cf2` | Exit 0. `cf2.bin`. Same version string. Flash 305176/1032192 (30%), text 297120, data 8056. RAM 101996/131072 (78%), bss 93940. CCM 55640/65536 (85%). 336 parameters, 572 log variables. |
| State streamer as a cf21bl app | in `examples/app_state_stream_aideck`: `make cf21bl_defconfig && make -j4` | Exit 0. `build/cf21bl.bin`. Flash 298260/1032192 (29%), text 290240, data 8020. RAM 96024/131072 (73%), bss 88004. CCM 55272/65536 (84%). Same 328 parameters and 576 logs: the app adds no log group. |

**[Derived].** The app image is 456 bytes more flash than stock `cf21bl` and about 1.4 KB more BSS, which is the 350-word stack plus a small amount of code. Both platform images link on the compiler this note actually had, which is one minor version below the pixi pin (`>=13.3.1`) and above the documented minimum (10.3). No warning was promoted to an error. UKF was not in either link.

**[Opinion].** The builds show the firmware and the UART1 example are compilable for `cf21bl`. They say nothing about DSHOT timing, the CTS handshake, or the inference parser, because those were not run.

---

## 13. Short verdict

**[Repo-stated].** This fork is Bitcraze `crazyflie-firmware` at `b65d8c57`, 104 commits after tag `2026.08`, with no local diff. `cf21bl` is a real platform: OTP token `C21B`, BMI088 on I2C, open-drain DSHOT on PA1/PB11/PA15/PB10, arming required, bidirectional DSHOT at 300 kHz. The AI-deck driver speaks CPX on UART2 at 576000 and can bootload the GAP8 (via the ESP32) and the ESP32 (ROM bootloader). UART1 is free, and `examples/app_state_stream_aideck` already sends NanoCockpit's `!STA` frame on it.

**[Derived].** The 2021.06 application does not drop in. The build flag, the commander priority numbers, the setpoint struct, the arming gate, the motor map, and the AI-deck driver have all moved. The port is an out-of-tree app on `cf21bl_defconfig`, keeping the GAP8 UART protocol, leaving the stock AI-deck driver on UART2, and using cflib ≥ 0.1.32 with Crazyradio 2.0 firmware ≥ 5.2. Flow (or external pose) is the position source that fits next to that UART. Lighthouse does not, unless the GAP8 link moves to CPX as well.

**[Opinion].** Do not flash the NanoCockpit STM32 image on this airframe, and do not flash this `cf2` image on it either. The first is the wrong motor protocol and will not match OTP `C21B`. The second halts in `main` before the scheduler, which is the safe failure, and it is still the wrong image to "try".
