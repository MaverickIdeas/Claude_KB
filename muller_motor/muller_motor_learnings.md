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

## Phase 2: Ethernet comms to the host ( verified 2026-06-07 )

The N6 and a host PC ( temp brain ) exchange Ethernet over a direct cable. Board 192.168.10.10/24, host 192.168.10.20/24 ( static; there is no DHCP on a direct link ), ping about 1 ms, board MAC 00:80:E0:00:10:00. Firmware is ST's Nx_WebServer NetXDuo example adapted for a static IP with no SD card, plus a UDP echo on port 5005. Repo writeup: docs/ethernet_comms.md.

Gotchas we hit and SOLVED:

12. **No SD card is fatal in the NetXDuo web server example.** `MX_SDMMC2_SD_Init()` runs in main() BEFORE the console UART and BEFORE the network. Without a card it fails into Error_Handler ( red LED 1 Hz, no console, no network, no ARP ). It looks like a dead board or an Ethernet failure, but it is the SD init. Skip the call ( comment it ) or insert a card. This cost the most time in Phase 2.

13. **NetXDuo on a direct link needs a static IP.** No DHCP server, so set NX_APP_DEFAULT_IP_ADDRESS/NET_MASK in app_netxduo.h and, in nx_app_thread_entry, remove the nx_dhcp_start plus the `tx_semaphore_get( DHCPSemaphore, TX_WAIT_FOREVER )` wait ( it blocks forever ). Also convert App_Link_Thread_Entry's DHCP restart block ( nx_dhcp_reinitialize clears the static IP and then it hangs on a relink ). The NetX IP thread auto enables the link, so ARP answers once boot reaches it; no manual NX_LINK_ENABLE is needed on first boot.

14. **STM32_SigningTool_CLI hangs on an overwrite prompt** when the output file already exists and there is no console to answer. Pipe y into it ( `"y" | STM32_SigningTool_CLI ...`, ST's `echo y` trick ). Otherwise the board never gets reflashed and you chase a phantom firmware bug.

15. **A 1 Gbps link light means nothing about the stack.** The RTL8211 PHY auto negotiates even under unrelated firmware, so Windows shows the NIC up at 1 Gbps while the board answers nothing. Diagnose at the app layer ( ping, ARP, the board console ), not the link light.

16. **N6570-DK ETH is RGMII to an RTL8211; the ETH DMA descriptors and NetX packet pools sit in the non cacheable AXISRAM2 window** ( 0x341D4000 to 0x341FFFFF ) with the MPU set to match ( ST's linker does this; confirm in the build .map ). So cache coherency is usually not the bug; check the SD init and the static IP conversion first.

Win: app level test is `tools\udp_echo_test.ps1` ( host sends UDP, board echoes on port 5005 ). Set the host NIC static with `netsh interface ip set address name=Ethernet static 192.168.10.20 255.255.255.0`. The mini PC ( the real brain, arriving 2026-06-08 ) uses the same scheme.

## Phase 2 control milestones ( M0, M1 ) — 2026-06-07

- **M0 VERIFIED**: control skeleton from SRAM ( ST GPIO_IOToggle FSBL base, runs from AXISRAM2 ), the M55 DWT cycle counter as the microsecond timebase, and a 1 Hz LED heartbeat. Source preserved in the repo at `firmware\control_app\main.c`.
- **M1 built ( bench verify pending )**: channel 1 from a hardware timer ( TIM1_CH1 on PE9 = Arduino D3 ), a 10 ms / 2 ms one pulse, with a loopback self test ( jumper D3 to D14; TIM4_CH4 on PC1 captures both edges and the firmware checks duty and jitter ), reporting on LED1 ( 1 Hz = pass, 5 Hz = fail ). A scope on D3 gives the absolute timing.
- **Pin mapping** ( `docs\pin_mapping.md` ): 5 outputs on TIM1 CH1..4 plus TIM2 CH3; 5 Hall inputs on TIM4, TIM2, TIM16, TIM14; conflict free against the Ethernet, flash, LED, and VCP pins. The legacy Arduino pin numbers do NOT map.

More gotchas:

17. **Avoid HAL module surgery: use DWT and direct timer registers.** The minimal GPIO_IOToggle build does not enable HAL_TIM or HAL_UART. Rather than add HAL sources to the .project, use the M55 DWT cycle counter for the timebase and configure TIM1/TIM4 by direct register writes ( the HAL RCC and GPIO clock and pin macros ARE available ). Compiles clean, no build config change.
18. **N6 timer clock is uncertain ( 200 vs 400 MHz ).** PCLK is 200 MHz; whether TIMxCLK doubles is unconfirmed. M1 sets the prescaler for 200 MHz; the loopback self test checks duty and jitter ( clock independent ), and a scope on the output gives the absolute period ( if it is half, the clock is 400 MHz; halve the prescaler ).
19. **N6 pin mapping ground truth = the ST/Zephyr pinctrl** `stm32n657x0hxq-pinctrl.dtsi` plus the Zephyr `arduino_r3_connector.dtsi`, both on GitHub. The datasheet AF table is Akamai blocked; the Zephyr pinctrl is derived from it and is fetchable.

## Verified facts ( this PC, June 2026 )
- STLINK V3EC: VID 0483 PID 3754, COM19, SN 001F00453234511233353533, FW V3J15M6. Board Device ID 0x486, Rev B, 3.29 V.
- STM32CubeProgrammer 2.22.0; STM32CubeIDE 2.1.1 ( C:\ST\STM32CubeIDE_2.1.1 ); STM32 signing tool 2.22.0.
- External loader: MX66UW1G45G_STM32N6570-DK.stldr ( in the CubeProgrammer bin\ExternalLoader folder ).
