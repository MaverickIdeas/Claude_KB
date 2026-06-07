---
name: muller-motor-project-overview
description: "Muller motor controller on STM32N6570-DK — hardware, boot model, architecture split, toolchain, and Phase 1 status as of June 2026"
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
- **Next ( Phase 2 )**: port the motor firmware, then split time critical control ( stays on the N6, ideally from SRAM or TCM for low jitter ) from brain logic ( mini PC over isolated Ethernet ).

## Primary references
- UM3234 ( boot ROM on STM32N6 ), UM3300 ( STM32N6570-DK ), UM3239 ( getting started with STM32CubeN6 ), the STM32N657X0 datasheet, and the STM32N6 reference manual.
