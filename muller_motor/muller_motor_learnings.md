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

## Phase 2 system architecture ( brain, backend, power boards ) — found 2026-06-08

Cross repo investigation of the App, Uno Q relay, nano power firmware, and
protocol. Three findings that correct working assumptions:

20. **The App does NOT push to any cloud backend.** On master and develop,
    `Muller_Motor_App` is a pure BLE controller; its only network call is an
    anonymous read of the public GitHub Releases API. There is no Firebase SDK,
    no BigQuery client, and no server credential on the shipping branches. So the
    mini PC is the FIRST server side writer in the system, not an inheritor of an
    existing pipeline. ( An unmerged `feature/autotune` branch started Firebase /
    Firestore work against a real motor project `muller-motor-controller`, but
    that is client config plus interactive Firebase CLI login, NOT a server key. )
21. **`google-services.json` is a client config, not a server credential.** The
    mini PC, an unattended server, needs its OWN Google Cloud service account
    JSON key with least privilege IAM ( `roles/datastore.user`,
    `roles/bigquery.dataEditor` scoped to one dataset, `roles/bigquery.jobUser` ).
    It does not exist yet; mint it in `muller-motor-controller`. Do NOT reuse
    `optix-2ddbc` ( that is the separate OPTIX stock sentiment product ). Store
    the key only on the mini PC at `/etc/muller-brain/sa.json`, referenced via
    `GOOGLE_APPLICATION_CREDENTIALS` in a systemd `EnvironmentFile`, never in git.
    Never let a key reach the auto pushing Claude_KB. Repo writeup:
    `docs\minipc_backend_creds.md`.
22. **The nano power boards are BLE peripherals; the mini PC must become a BLE
    central to read them.** Each board ( PwrMeter3 ) advertises the standard
    Muller service `19B10000-...` with name `MMC-Input-PWR` or `MMC-Output-PWR`,
    and NOTIFYs a 140 byte v2 telemetry packet on TELEM `19B10002-...` at ~5 Hz
    with a 28 byte power block at offset 112 ( two ACS37800 slots per board:
    pwr_v[2], pwr_i[2], pwr_w[2], flags, state ). Today they pair directly with
    the App; the Uno Q relay was a GATT server only, so the BLE central role is
    new work ( brain milestone B4 ). Input vs output is set at build time
    ( `-DMMC_ROLE_OUTPUT` ) and read from the advertised name or the INFO char.
    No energy ( Wh ) and no temperature are measured. Repo writeup:
    `docs\nano_power_dataflow.md`.

## Dashboard pivot ( App dropped, mini PC is the head ) — 2026-06-08

23. **The Android App is dropped; the mini PC is the primary UI and commander.**
    It commands the N6 directly over the UDP link ( `[CMD]` out, `[TELEM]` 112 byte
    v1 back ), is BLE central for the nano power boards, and is the config master.
    This DELETES the BLE GATT server ( the old brain plan's main job ), the v1 to
    v2 splice toward the App, and the dual role radio need. Authoritative plan:
    `docs\minipc_dashboard_plan.md` ( U0 to U7 ); `docs\minipc_brain_plan.md` is
    superseded.
24. **Build the dashboard by extending `muller-motor-python-gui` ( Tkinter, 454
    lines, v1.7 ), but swap TWO layers, not one.** Reuse its structure, threading,
    widgets ( ~70 percent ). ( a ) Transport: it is serial / pyserial 115200 only,
    swap to a UDP datagram client to 192.168.10.10:5005. ( b ) Codec: it speaks an
    ASCII line protocol ( `RUN`, `STOP`, `CONFIG:FREQ:..` ) proven against the
    Arduino sim, but the N6 speaks the BINARY MMCProtocol ( cmds 0x01 to 0x0F,
    little endian, 112 byte v1 telemetry per `static_assert` in MMCProtocol.h ).
    Replacing only the socket is the trap; the wire codec changes too.
25. **No v2 ( 140 byte ) telemetry frame exists in the protocol header.** The 112
    byte v1 struct has motor data only; the power V / I / W / COP comes from the
    nano BLE block as a SEPARATE input. The dashboard merges the two in memory; it
    never needs a v2 wire frame. ( Earlier notes that called the nano packet a
    "140 byte v2" describe the nano's own BLE payload, not an N6 telemetry frame. )
26. **Dropping the App flips the OS choice: a single Windows box is now the clean
    default.** The only hard on Windows job was the App facing BLE GATT server,
    which is gone. The remaining BLE is central only ( `bleak`, fine on Windows ),
    everything else is OS neutral, and the firmware toolchain already runs on
    Windows. Revisit Linux only if it becomes a 24/7 unattended appliance.

## Verified facts ( old dev PC, June 2026 )
- STLINK V3EC: VID 0483 PID 3754, COM19, SN 001F00453234511233353533, FW V3J15M6. Board Device ID 0x486, Rev B, 3.29 V.
- STM32CubeProgrammer 2.22.0; STM32CubeIDE 2.1.1 ( C:\ST\STM32CubeIDE_2.1.1 ); STM32 signing tool 2.22.0.
- External loader: MX66UW1G45G_STM32N6570-DK.stldr ( in the CubeProgrammer bin\ExternalLoader folder ).

## Move to the lab mini PC ( the brain box ) and M2 authored ( 2026-06-08 )

The mini PC ( the real brain we were waiting for ) arrived; firmware dev moved to
it. State found and advanced this session:

27. **The lab mini PC is a FRESH box: the board is here but the ST toolchain is
    NOT.** `tools\check_toolchain.ps1` on the mini PC reports PRESENT: git, the
    control source, the cloned vendor CubeN6 tree, and the board ( STLINK V3EC
    detected on USB ). MISSING: STM32CubeProgrammer, STM32_SigningTool_CLI, the
    external loader `.stldr`, and STM32CubeIDE. A machine wide search found no ST
    installers anywhere. So on this box NOTHING can be built, signed, or flashed
    until a human installs CubeProgrammer + CubeIDE from st.com in a browser ( the
    download is Akamai gated, gotcha 7 ). That install is the single gate on all
    firmware progress on the mini PC. `tools\get_cuben6.bat` ran clean here
    ( GitHub is not gated ), so the vendor HAL is already on disk.

28. **M2 ( five channel open loop sequencer ) is AUTHORED, ahead of that install.**
    `firmware\control_app\main.c` now ports the proven Muller scheduler ( legacy
    onHallEdge + driveOutputs ): per rotor edge, schedule a phase shifted one shot
    pulse at now + wrap01( phase_base + trim[ch] ) * T, width pulse_frac * T or an
    abs ms with a ~20 us guard band, plus overlap protection. A SYNTHETIC rotor
    ( five edges per period, channel k staggered by k * T/5, 600 RPM / 12.5 ms
    corner ) drives it so the sequencing, phase, trim, guard band, and overlap are
    exercised open loop with no rotor. Outputs are plain push pull GPIOs off the
    DWT timebase ( OUT1 PE9, OUT2 PA9, OUT3 PE13, OUT4 PE14, OUT5 PB10 ). LED1
    self test: 1 Hz = all enabled channels firing one pulse per edge, 5 Hz = a
    channel starved. M3 swaps the synthetic edges for real Hall captures with no
    scheduler change. M2 supersedes the M1 single channel demo ( M1 image still on
    the board, bench verify still pending ). Committed to origin main.

29. **Design choice: M2 uses a SOFTWARE scheduler, not hardware one pulse.** The
    phase2 plan eyed hardware OPM for jitter, but both legacy controllers used a
    software micros() scheduler, so M2 ports that logic verbatim ( de risked, and
    it is the exact path M3 reuses ). Hardware OPM is a later separable refinement
    that changes only the final edge emission. To keep the minimal GPIO_IOToggle
    build linking unchanged, M2 avoids libm: no floorf ( wrap01 uses bounded adds )
    and no roundf ( integer +0.5f casts ). Register and AF macro names were
    verified against the cloned HAL headers since the box cannot compile yet.

30. **Memory junction is NOT set up on the mini PC.** The live memory folder
    `~\.claude\projects\C--Users-Muller-Motor-Control-Documents-GitHub\memory` is
    an empty real dir here, not a junction into Claude_KB. The canonical KB
    ( Claude_KB\muller_motor ) stays the source of truth; future sessions should
    re establish the junction per the Claude_KB README if they want live memory to
    write straight into the repo on this box. ( The session working dir is the
    GitHub parent, so decide whether one junction serves the whole folder or per
    repo. )

## M2 flashed + verified, and the touchscreen ( D0/D1 ) brought up ( 2026-06-08 )

The ST toolchain landed; M2 built clean ( 0 errors ), a 15 agent adversarial
review found no blocking bug, and M2 plus a live LTDC display are flashed and
verified on the board.

31. **Re flashing the N6 needs DEV boot ( BOOT1 = H ) AND a power cycle, then
    mode=UR.** Two distinct failures seen: ( a ) HOTPLUG with the board running
    its old image gives "Unable to get core ID" because the boot pins were latched
    at the last power on; mode=UR ( connect Under Reset ) re samples them and
    fixed it. ( b ) Even with UR, if the board is in FLASH boot the external flash
    is memory mapped and erase fails with "failed to erase memory". Fix: set
    BOOT1 = H ( DEV boot ), POWER CYCLE so the flash is not mapped, then flash with
    mode=UR. After flashing, BOOT1 = L ( Flash boot ) + power cycle to run.
    `tools\build_control.bat` and `tools\build_lcd.bat` now use mode=UR. The
    programmer's clean exit ( erase, download, exit 0 ) is the source of truth,
    not a memory readback ( gotcha 11 ).

32. **LTDC display bring up on the N6570-DK is far simpler than the full BSP: the
    RK050HR18 is a dumb parallel RGB panel.** Drive it via HAL_LTDC DIRECTLY
    ( ST's `LTDC_Horizontal_Mirroring` FSBL example is the base ), no MIPI / panel
    command driver. What it needs: the 800x480 timing ( HSync 4, VSync 4, AccHBP
    12, AccVBP 12, AccActiveW 812, AccActiveH 492, Total 820x500 ), a PLL4 pixel
    clock ( PLL4 M=1 N=25 in SystemClock_Config, ON not NONE ), backlight + on via
    GPIOQ ( LCD_BL_CTRL PQ6, LCD_ONOFF PQ3 ), and the N6 RIF master config
    ( HAL_RIF_RIMC_ConfigMasterAttributes for RIF_MASTER_INDEX_LTDC1 +
    HAL_RIF_RISC_SetSlaveSecureAttributes for LTDCL1 ) or the LTDC cannot read the
    framebuffer. HAL_LTDC_MspInit ( the RGB pin + clock config ) lives in
    `stm32n6xx_hal_msp.c`, a separate compiled file, so swapping the project's
    main.c keeps it. The BSP submodules ( `Drivers/BSP/STM32N6570-DK`, components
    `rk050hr18`, `gt911` ) are NOT in the base clone; init them ( get_cuben6 only
    did HAL + CMSIS ).

33. **Framebuffer choices that de risk the display ( verified D1 ).** Use a SMALL
    layer ( we used 256x160 RGB565 = 80 KB ) as a plain RAM static array, not a
    full 800x480 buffer at 0x34000000: 80 KB fits the FSBL RAM region, so no
    custom linker region, no 0x34000000 access / extra RIF question. Keep D-cache
    OFF ( the LTDC example never enables it ), so CPU framebuffer writes are
    coherent with the LTDC DMA and no SCB_CleanDCache is needed; enabling D-cache
    ( as the GPIO_IOToggle M2 base did ) would show a stale / garbage screen
    unless every frame is flushed. Single buffered is fine for a liveness animation
    ( minor tearing ); a double buffer + DMA2D is the D3 refinement. The motor I/O
    pins ( PE9, PA9, PE13, PE14, PB10 ) do not collide with any LTDC pin
    ( pin_mapping cross check held ), so M2 and the panel coexist in one FSBL.

34. **The combined firmware ( M2 + live display ) is `firmware\control_app\main_lcd.c`
    built in the LTDC project via `tools\build_lcd.bat`; pure M2 stays
    `firmware\control_app\main.c` via `tools\build_control.bat`.** main_lcd.c
    inlines the M2 scheduler verbatim ( a shared control core is a follow up ) and
    redraws at ~20 Hz from the main loop ( draw_frame pauses the control poll for
    under a millisecond every 50 ms; acceptable open loop, revisit with DMA2D ).
    The five on screen activity blocks ( green when a channel fired ) are an easy
    live readout of the control loop and confirmed all five channels firing.
