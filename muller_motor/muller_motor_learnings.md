---
name: muller-motor-learnings
description: "STM32N6 boot and flash playbook plus hard won gotchas — build, sign, flash, verified LED blink, and the toolchain and clone traps, so future sessions don't re-derive them"
metadata:
  node_type: memory
  type: project
  originSessionId: 5da15695-411e-4859-a37b-b739ede023ae
---

# Muller Motor / STM32N6 — playbook and hard won learnings ( June 2026 )

## Verified Phase 1 playbook: blink an LED from external flash
Proven end to end on 2026-06-07 with ST's GPIO_IOToggle example. This reproduces the signed boot chain.

1. **Get the reference** ( one time ): `tools\get_cuben6.bat` clones STM32CubeN6 into `vendor\` and inits the HAL and CMSIS device submodules.
2. **Build, sign, flash**: `tools\blink_demo.bat` headless builds GPIO_IOToggle_FSBL with STM32CubeIDE, signs it as fsbl, and programs it to 0x70000000.
3. **Switches**: DEV boot ( BOOT1 = H ) to flash, then Flash boot ( BOOT1 = L, BOOT0 = L ) and power cycle to run. LED1 blinks.

Exact commands the scripts run:
- Build: `stm32cubeidec.exe --launcher.suppressErrors -nosplash -application org.eclipse.cdt.managedbuilder.core.headlessbuild -data <ws> -import <FSBL project dir> -cleanBuild GPIO_IOToggle_FSBL/Release`
- Sign FSBL: `STM32_SigningTool_CLI -bin GPIO_IOToggle_FSBL.bin -nk -of 0x80000000 -t fsbl -hv 2.3 -o <out>_trusted.bin -align`
- Flash: `STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el MX66UW1G45G_STM32N6570-DK.stldr -hardRst -w <signed.bin> 0x70000000`

## Gotchas we hit and SOLVED ( don't re-derive )

1. **No internal flash; everything is signed and external.** Code lives in the MX66UW1G45G Octo SPI NOR. The boot ROM only runs a signed FSBL. FSBL at 0x70000000, application at 0x70100000.

2. **FSBL and application sign as DIFFERENT types.** FSBL: `-t fsbl` with `-of 0x80000000`. Application ( second stage ): `-t ssbl`. Both `-hv 2.3`. The original template placeholder signed both as fsbl ( wrong ) and was missing `-of 0x80000000`.

3. **`-align` is required from STM32CubeProgrammer 2.21.0 onward** ( bare `-align`, no value ). We are on 2.22.0, so it is needed. Drop it only on older versions.

4. **Cortex M55 needs a modern GCC.** The machine's stray GCC 7 ( 2018-q2 ) has NO `-mcpu=cortex-m55` ( M55 support landed in GCC 10 ). STM32CubeIDE 2.1.1 bundles the correct GCC. Do not try to build the N6 with the old GCC.

5. **ST's CubeN6 examples are STM32CubeIDE projects, no CMake.** Build them with CubeIDE headless ( stm32cubeidec.exe ), not a bare make or cmake flow. STM32CubeCLT alone would not build these.

6. **GPIO_IOToggle is built AS the FSBL** ( single image at 0x70000000 ), not a two stage FSBL plus app. The buildable project is `GPIO_IOToggle_FSBL` ( configs Debug and Release ). It links with `STM32N657X0HXQ_AXISRAM2_fsbl.ld` and runs from AXISRAM2 ( entry point seen at 0x3418078d ). The managed build auto produces a `.bin` ( objcopy is part of it ).

7. **st.com is Akamai gated for automated downloads.** TCP 443 connects but the HTTP response never returns ( Invoke-WebRequest, curl, and the server side WebFetch all hang or return http=000; community.st.com 403s ). GitHub is fine. To recover ST command syntax, use WebSearch plus ST's GitHub repos. The user must download the STM32Cube installers through a real browser logged into their ST account.

8. **Cloning STM32CubeN6 on Windows has two traps:**
   - **MAX_PATH**: checkout fails on the very long CMSIS DSP documentation filenames. Fix: `git -c core.longpaths=true clone ...` and set `core.longpaths true` in the clone.
   - **Submodules**: HAL, CMSIS device, and BSP are git submodules; a plain clone leaves them as empty folders ( and `git status` looks clean because they are merely uninitialized ). The GPIO blink needs only `Drivers/STM32N6xx_HAL_Driver` and `Drivers/CMSIS/Device/ST/STM32N6xx`. `tools\get_cuben6.bat` codifies both fixes.

9. **Windows batch trap: literal parentheses inside an `if (...)` block.** A `)` in an `echo` within a parenthesized block closes the block early ( error: "X was unexpected at this time" ). Keep parentheses out of echo text inside blocks. This bit the demo scripts twice.

10. **Boot switches are two position H and L** ( not the "1-2 / 1-3" terminal notation in ST's readme ). DEV boot ( program ): BOOT1 = H, BOOT0 = L. Flash boot ( run ): BOOT1 = L, BOOT0 = L, then power cycle. Serial boot: BOOT1 = L, BOOT0 = H.

11. **Debugger memory views are unreliable for the external XSPI ranges** ( around 0x70000000 and 0x90000000 ). Trust the programmer's exit code on `-w` ( erase, download, exit 0 ), not a memory read back, to confirm a flash write.

## Shipped wins
- **Phase 1 boot chain VERIFIED 2026-06-07**: signed FSBL boots from external flash, LED1 blinks. Build, sign, and flash all clean.
- **Repo tooling** ( all in `tools\` ): `detect_board.ps1` ( find the STLINK and COM port ), `get_cuben6.bat` ( clone the reference plus submodules ), `sign.bat` and `flash.bat` ( general two image project ), `blink_demo.bat` ( one shot GPIO blink build, sign, flash ).
- **Docs** in `docs\`: `boot_flash_signing.md` ( the why and how of the N6 boot ) and `README_FLASH.md` ( the operational runbook ).
- **GitHub**: the repo was made PRIVATE before any push; `MaverickIdeas/Muller_Motor_STM32N6`.

## Verified facts ( this PC, June 2026 )
- STLINK V3EC: VID 0483 PID 3754, COM19, SN 001F00453234511233353533, FW V3J15M6. Board Device ID 0x486, Rev B, 3.29 V.
- STM32CubeProgrammer 2.22.0; STM32CubeIDE 2.1.1 ( C:\ST\STM32CubeIDE_2.1.1 ); STM32 signing tool 2.22.0.
- External loader: MX66UW1G45G_STM32N6570-DK.stldr ( in the CubeProgrammer bin\ExternalLoader folder ).
