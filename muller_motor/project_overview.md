---
name: muller-motor-project-overview
description: "Muller motor controller on STM32N6570-DK — hardware, boot model, architecture split, toolchain, and milestone status as of July 2026"
metadata:
  node_type: memory
  type: project
  originSessionId: 5da15695-411e-4859-a37b-b739ede023ae
---

# Muller Motor Controller — STM32N6570-DK

## Identity
- **Repo**: `D:\Maverick Ideas LLC\GitHub\Muller_Motor_STM32N6` → GitHub `MaverickIdeas/Muller_Motor_STM32N6` ( PRIVATE ).
- **What it is**: a five channel drive controller for a Muller motor with a sixteen magnet rotor. The five timed channels fire against the rotor, sequenced from rotor position.
- **Split**: time critical deterministic control ( commutation timing, position capture, channel sequencing, safety interlocks ) runs on the STM32N6. A separate mini PC ( the "brain" ) does telemetry, configuration, logging, and optimization. The two link over galvanically isolated Ethernet.
- **Board**: STM32N6570-DK Discovery. A custom board may follow.

## Hardware truth ( verified )
- **MCU**: STM32N657X0H3Q, Cortex M55 at 800 MHz, Helium vector DSP, Neural ART NPU, 4.2 MB contiguous SRAM. Device ID 0x486, Rev B.
- **No internal user flash.** The 0 in STM32N6570 is the flash size code ( zero Kbyte ). Code lives in external flash and is brought up by a signed first stage bootloader ( FSBL ).
- **External memory on the DK**: 1 Gbit Octo SPI NOR flash ( Macronix MX66UW1G45G ) and 256 Mbit Hexadeca SPI PSRAM.
- **Ethernet**: Gigabit with TSN, onboard RJ45 ( the isolated link to the brain ).
- **Onboard debugger**: STLINK V3EC. USB VID 0483 PID 3754. Presents a SWD debug interface plus a virtual COM port ( COM19 on this PC ). Probe SN 001F00453234511233353533, FW V3J15M6.

## Boot and address model
- The boot ROM loads a signed FSBL from external flash into internal SRAM. The FSBL sets clocks and external memory, then runs the application ( copied to SRAM, or memory mapped XIP from flash ).
- **FSBL address**: 0x70000000. **Application address**: 0x70100000.
- Unsigned images are rejected by the boot ROM.
- Boot switches are two position H and L ( BOOT0, BOOT1 ): DEV boot = BOOT1 H ( program external flash ); Flash boot = BOOT1 L and BOOT0 L, then power cycle ( run ).

## Toolchain ( on this PC )
- **STM32CubeProgrammer 2.22.0**: STM32_Programmer_CLI, STM32_SigningTool_CLI, and the external loader MX66UW1G45G_STM32N6570-DK.stldr.
- **STM32CubeIDE 2.1.1** at C:\ST\STM32CubeIDE_2.1.1 ( bundles arm-none-eabi-gcc for Cortex M55 and the headless builder ).
- Build with CubeIDE headless ( ST's CubeN6 examples are IDE projects, no CMake ). Sign and flash with the CubeProgrammer CLIs.

## Scope and rules
- **Current sensing is OUT OF SCOPE** ( handled elsewhere ). Do not implement or modify it.
- Keep the brain side and motor side grounds galvanically isolated ( Ethernet magnetics ). No plain USB tether between the mains PC and the motor side board in the deployed system.
- Repo conventions: commits allowed ( recommended on milestone verifications ); no hyphens in prose; verify specs against datasheets; prefer ST templates over hand written boot code.

## Status
- **Phase 1 ( boot chain ) VERIFIED 2026-06-07**: a signed FSBL boots from external flash and blinks LED1, via a repeatable build, sign, and flash pipeline. See `muller_motor_learnings.md` for the playbook.
- **Phase 2 comms VERIFIED 2026-06-07**: N6 to host Ethernet over a direct cable ( static IP, ping about 1 ms, app level UDP echo on port 5005 ). See `ethernet_comms.md`.
- **Phase 2 port plan and pin mapping done**: `phase2_port_plan.md`, `pin_mapping.md`. The split: time critical control on the N6, brain logic on the mini PC over isolated Ethernet.
- **Milestones as of 2026-07-01**: the shipping firmware is `firmware\control_app\main_lcd.c` ( M2 scheduler + M3 EXTI Hall capture + M4 safety + LTDC panel ), now flashed as the LIVE real Hall image ( DEMO_SYNTH_ROTOR=0 via `tools\build_lcd.bat live` ). **M5/M6 ( UDP telemetry out, commands in ) VERIFIED ON HARDWARE 2026-06-30**: ~50 Hz 112 byte telemetry decodes on the brain and the full command round trip is confirmed by telemetry readback. **M9 NOR persistence IMPLEMENTED and initialized on hardware** ( 68 byte cfg_record in two alternating 4 KB subsectors; save plus power cycle restore verification still PENDING ). Trim is now ABSOLUTE ROTOR DEGREES 0..360 on both ends ( firmware applies deg mod 45 ). Authoritative milestone ledger and the resume banner: `docs\RESUME.md`.
- **Brain web console OPERATIONAL**: `dashboard\web_console.py` ( aiohttp ) on 127.0.0.1:8500, ~15 Hz WebSocket telemetry including the cfg ack bits, commands as JSON mapped to the mmc_protocol encoders, confirmed by telemetry readback. Do not run dashboard.py at the same time ( both bind UDP 5005 ).
- **MOTOR RAN under its own power, proper direction, 2026-07-14**: on DERIVED trims ( placement instrument + coil geometry ) the machine SUSTAINED ~610 rpm on CH1 alone with 155 us jitter. M3/M4 are now bench proven on the real rotor. This session also: portrait panel ( learning 60 ), the firing induced Hall double count root cause ( 61, drive side flyback coupling, NOT sensor side ), the 600 MHz / dead TIM6 output timer fix ( 62 ), noise hardening + real fault latch ( 63 ), enabled channel median RPM ( 64 ), a provable firmware build id in v4 telemetry ( 65 ), the detached brain stack with activity log / firehose / autotune ( 66 ), and Rigol MSO5074 input+output power over isolated Ethernet with a persisted clamp tare ( 68, 69 ). See learnings 60 to 69 and `docs\audit_io_signals.md`, `docs\self_heal_design.md`, `docs\ch1_run_analysis_20260714.md`.
- **Open items**: the flyback coupling is a HARDWARE fix owed ( drive side snubber + sensor wire routing; the firmware only tolerates it ). Self healing / single master Hall mode is DESIGNED and adversarially reviewed but NOT built ( needs CH5 hardware fixed + a NOR offset calibration first; six blockers in `docs\self_heal_design.md` ). The M7 closed loop RPM PID is unbuilt ( the RPM target slider is inert; pulse width is the throttle ). Output power tare + a trustworthy COP were pending a clean bus energized rotor stopped tare at session end.
- **PIVOT 2026-06-08, the App is dropped; the mini PC is the HEAD.** The mini PC
  is now the primary UI and commander of the N6 over UDP, not a BLE relay for a
  phone. Authoritative plan: `docs\minipc_dashboard_plan.md` ( milestones U0 to
  U7 ). Extend the existing `muller-motor-python-gui` ( Tkinter ); swap its
  transport ( serial to UDP ) AND codec ( ASCII to the binary MMCProtocol, cmds
  0x01 to 0x0F, 112 byte v1 telemetry ), add BLE central for the nanos. Dropping
  the App removes the BLE GATT server, so a single WINDOWS box ( firmware dev +
  dashboard ) is now the clean default. `docs\minipc_brain_plan.md` ( B1 to B5 )
  is SUPERSEDED. The N6 touchscreen ( `docs\touchscreen_plan.md` ) stays as a thin
  link independent safety mirror + touch STOP ( D0 to D5 ), complementary to the
  mini PC dashboard.
- **Backend reality ( corrected 2026-06-08, still true 2026-07-01 )**: the production App does NOT push to any cloud backend; it is a pure BLE controller on master. The mini PC will be the FIRST server side writer to Firebase + BigQuery and needs its OWN Google Cloud service account key ( does not exist yet, must be minted in the motor's `muller-motor-controller` project; NOT `optix-2ddbc` ). Cloud auth remains UNPROVISIONED on this PC. The nano power boards ( input and output ) are BLE peripherals that NOTIFY a 140 byte v2 telemetry packet at ~5 Hz and today pair with the App; the mini PC must add a BLE central role to ingest them. See the two docs above and `muller_motor_learnings.md`.

## Primary references
- UM3234 ( boot ROM on STM32N6 ), UM3300 ( STM32N6570-DK ), UM3239 ( getting started with STM32CubeN6 ), the STM32N657X0 datasheet, and the STM32N6 reference manual.
