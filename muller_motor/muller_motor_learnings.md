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

## M4 safety, M3 EXTI capture, and the brain link plan ( 2026-06-08 )

35. **M4 safety interlocks are in `main_lcd.c` ( the firmware going forward ); the
    pure `main.c` is frozen at M2.** Three interlocks, all link independent:
    g_run_enable global STOP ( drive_outputs forces every output low and drops
    schedules ); a stuck high watchdog ( an output may never stay high beyond one
    rotor period or a 1 s fallback, else force low + latch a per channel fault );
    and the REAL end of period pulse guard the M2 review deferred - on_rotor_edge
    now clamps each pulse to end >= guard before the next edge ACCOUNTING FOR THE
    PHASE OFFSET ( not just width ), in both frac and abs modes, skipping the pulse
    if the phase leaves no room. Overlap protection is now a backstop, not the
    mechanism. The dashboard shows RUN/STOP and a red LED per faulted channel.

36. **M3 = real Hall capture via EXTI, behind DEMO_SYNTH_ROTOR ( 1 = synthetic
    demo, no IRQs; 0 = LIVE ).** Five Hall pins map to distinct EXTI lines
    ( PC1 line1 ch0, PH9 line9 ch1, PA5 line5 ch2, PA3 line3 ch3, PG2 line2 ch4 );
    the rising callback timestamps with now_cyc() and feeds the SAME on_rotor_edge.
    Concurrency ( ISR schedules, loop drives ): shared state volatile, the five
    EXTI IRQs share one NVIC priority ( no mutual preemption ), and the output RMW
    is in a brief PRIMASK critical section. Build verifies both modes; LIVE needs a
    Hall signal source ( signal gen or rotor ) to verify on the bench.

37. **Brain link architecture DECIDED: raw HAL_ETH + a minimal UDP/IP/ARP, NOT
    NetXDuo.** The control firmware is a bare main loop ( + LTDC ) with no RTOS;
    NetXDuo drags in ThreadX, which fights that. CubeN6 has no LwIP. So M5
    ( emit telemetry ) and M6 ( parse commands ) on the N6 should use stm32n6xx_hal_eth
    directly with a small UDP/IP/ARP/ICMP, RTOS free, polled from the control loop,
    descriptors in non cacheable RAM ( gotcha 16 ). This is the next big firmware
    effort and what connects the real board to the dashboard. ( Phase 2 proved the
    link with a NetXDuo example; the INTEGRATED control firmware should not.)

38. **The mini PC dashboard ( the brain UI ) is BUILT and emulator verified, in
    `dashboard\` in the N6 repo.** Built fresh ( the legacy muller-motor-python-gui
    is not on this machine ): Tkinter UI + UDP client + the binary MMCProtocol
    codec ( mmc_protocol.py ) + a software N6 emulator ( n6_emulator.py, also the
    executable reference for M5/M6 ) + selftest.py ( passes ). Python 3.12 via
    winget; tkinter ships with it. **UDP link contract NOW DEFINED** ( resolved the
    open decision ): datagram = [ type ][ payload ], CMD 0x01 / TELEM 0x02 ( the
    112 byte v1 struct ) / INFO 0x03 / DEBUG 0x04, fire and forget; plus a link
    extension CMD_SET_RUN 0x10 ( global STOP/RUN -> M4 g_run_enable ) since the App
    is dropped and the N6<->brain contract is ours. The N6 M5/M6 must implement
    this exact framing. Run: `python n6_emulator.py` then `python dashboard.py --ip 127.0.0.1`.

## LIVE image, M5/M6 on hardware, M9 NVM persistence, panel redesign, web console ( 2026-06-30 to 2026-07-01 )

39. **LIVE build: `tools\build_lcd.bat live` stages main_lcd.c and flips
    DEMO_SYNTH_ROTOR=0 via `tools\set_demo_synth_rotor.py`.** Two batch gotchas
    fixed in that script: nested parentheses inside a parenthesized `if (...)`
    block break cmd ( "was unexpected at this time", gotcha 9 strikes again ),
    and a ReadOnly stale signed bin makes `del` fail so the signing tool hangs
    on its overwrite prompt ( gotcha 14 in a new coat ). Fix: `attrib -r` before
    the del, now in the script.

40. **M5/M6 VERIFIED ON HARDWARE 2026-06-30.** ping/ARP works ( board MAC
    00:80:E0:00:10:00 at 192.168.10.10 ), ~50 Hz 112 byte telemetry decodes on
    the brain, and the full command round trip is confirmed by telemetry
    readback. Gotcha: the board only streams after it learns the peer; ping it
    or send any UDP to 5005 first. Win: the dashboard keepalive opens the
    Windows firewall return path without admin.

41. **Trim semantics are now ABSOLUTE ROTOR DEGREES 0..360 on both ends**
    ( `dashboard\mmc_protocol.py` TRIM range and the firmware clamp changed in
    lockstep ). Physics note: with 8 identical Hall edges per rev and one pulse
    per edge, any trim beyond 45 degrees lands on the same firing pattern as
    its mod 45 remainder, so the firmware applies deg mod 45 and NO multi edge
    scheduler was needed.

42. **M9 persistent config store is REAL and verified to init on hardware.**
    A 68 byte cfg_record ( magic CMM1, version, seq, CRC32 ) in two alternating
    4 KB NOR subsectors at offsets 0x01000000 and 0x01001000, read back verify
    after write, outputs quiesced during the blocking erase, and cfg_load at
    boot BEFORE the scheduler arms. CMD 0x0C/0x0D now do real save/load. The
    ack bits ( save counter, ok, dirty, restored ) ride the telemetry header
    _reserved byte.

43. **THE XSPI GOTCHA: BSP_XSPI_NOR_Init fails on this app** because the DK BSP
    silently assumes a 200 MHz XSPI2 kernel clock and the LTDC derived clock
    tree never routes one. Fix: HAL_RCCEx_PeriphCLKConfig with
    Xspi2ClockSelection = RCC_XSPI2CLKSOURCE_IC3 and IC3 = PLL1 ( 1200 MHz ) / 6
    BEFORE BSP init, and use OPI_MODE + DTR_TRANSFER ( plain SPI reads are out
    of spec at the 200 MHz the BSP switches to after init ). Failure shows on
    the panel as CFG NVM ERR with the BSP error code.

44. **`tools\xspi_enable.py` reapplies the vendor XSPI patch every build** ( the
    hal_conf define, mx66uw1g45g_conf.h and aps256xx_conf.h copied from the
    VENC_RTSP_Server application, four source links in .project ), same
    idempotent pattern as eth_enable.py.

45. **The panel 5x7 font has ONLY digits, A to Z, space, period, colon, minus
    and percent.** No hash, no asterisk, no degree sign. Status lines use A:1 /
    EDITED / plain numbers for degrees.

46. **The N6 panel was redesigned to align with the brain console vocabulary**:
    title bar RUN/LIVE, then RPM SET AUTO, then PH PW CFG ( A:1 OK / UNSAVED /
    SAVED flash / EDITED / NVM ERR n ), then LINK CMD GD 50HZ, then five rows of
    CHn state LED, period ms, trim degrees in title blue, edge count, and an
    activity bar. draw_dashboard now takes link_up from net_link_up(). New
    counters: g_cmd_rx and g_guard_skips.

47. **The brain web console is real: `dashboard\web_console.py`** ( aiohttp,
    pinned in requirements-web.txt ) on 127.0.0.1:8500 rides brain_core, ~15 Hz
    WebSocket telemetry including the cfg ack bits, commands as JSON mapped to
    the mmc_protocol encoders, and a journal that confirms by telemetry
    readback. Run: `python dashboard\web_console.py --ip 192.168.10.10`. Do NOT
    run dashboard.py at the same time ( both bind UDP 5005 ).

48. **DEV boot discipline: flipping BOOT1 alone is NOT enough.** The boot mode
    latches at power on, so a power cycle after the flip is required or the NOR
    stays memory mapped and the erase fails ( the second failure in gotcha 31,
    re confirmed ). A blank screen in DEV boot is correct, not a brick.

## 2026 07 02 ( M10 temps, telemetry v3, full screen panel, close out verified )

49. **M9 persistence VERIFIED END TO END on hardware 2026 07 02.** The full
    close out passed: brain command round trip with journal confirmation, panel
    trim and enable updates, SAVED flash then A:1 OK on save, and the config
    RESTORED across a power cycle from the N6 NOR with no brain attached.

50. **ADC on the N6 needs its kernel clock routed, same class of gotcha as
    XSPI.** Mirror ST's ADC_MultiChannelSingleConversion example exactly:
    `RCC_ADCCLKSOURCE_TIMG` with `AdcDivider = 5`, `__HAL_RCC_ADC12_CLK_ENABLE`,
    `HAL_ADCEx_Calibration_Start( ..., ADC_SINGLE_ENDED )`, and
    `HAL_GPIO_ConfigPinAttributes( ..., GPIO_PIN_SEC | GPIO_PIN_NPRIV )` per
    analog pin. tools\adc_enable.py reapplies the vendor hal_conf and .project
    patch every build.

51. **A0 ( PA5 ) has NO ADC input on the STM32N657.** Verified against the
    Zephyr pinctrl: the five coil sensors sit on A1..A5 = PA9 ( INP10 ),
    PA10 ( INP11 ), PA12 ( INP13 ), PF3 ( INP16 ), PB10 ( INP8 ), all ADC1.

52. **Floating analog pins read STABLE phantom temperatures ( ~1 V, drifting
    with die temp ), and the analog pulldown is NOT effective on this part**
    even though the HAL writes PUPDR. A pure noise spread test fails. The
    robust discriminator is the DUAL PRESENCE PROBE: force the pin low then
    high, sampling after each release; a driven TMP36/LM35 returns to its own
    value both times, a float follows the forcing. Implemented as a per channel
    state machine in temp_task.

53. **Telemetry v3 is 144 bytes** ( v1 plus temps, warn/fault bits, Hall edge
    phase vs CH1 in 0.1 deg, rotor period jitter, guard skip and cmd counters ).
    The compiler caught a 31 byte stack overflow because net_send_telemetry
    still had a 113 byte buffer: SIZE BUFFERS FROM THE STRUCT
    ( `uint8_t buf[1U + sizeof(mmc_telem_t)]` ) so they can never drift.

54. **The full screen 800x480 framebuffer lives at fixed addresses in
    AXISRAM3..6** ( 4 x 448 KB contiguous from 0x34200000 secure ). Those SRAMs
    boot powered down: enable the RAMCFG plus AXISRAMx MEM clocks and clear
    `RAMCFG_SRAMx_AXI->CR` SRAMSD directly ( the HAL_RAMCFG module is not
    linked ). LTDC DMA reads the region with no extra RIF work.

55. **Redrawing the live framebuffer flickers; double buffer instead.** Two
    768 KB buffers ( 0x34200000 / 0x342C0000 ), draw into the back one, then
    `HAL_LTDC_SetAddress` + `HAL_LTDC_Reload( LTDC_RELOAD_VERTICAL_BLANKING )`
    and swap. Guard the next redraw on `LTDC->SRCR` reload bits being clear.

56. **Anti aliased panel fonts: tools\gen_font.py** renders Bahnschrift
    SemiBold into 2 bit per pixel monospace tables at the exact 6*s cell sizes
    the layout assumes ( 12x16, 18x24, 36x48 ), ASCII 32..126 plus a degree
    glyph at index 95 reached with the 0x7F sentinel. draw_char blends glyph
    edges against the live framebuffer pixel. build_lcd.bat stages font_panel.h.

57. **J-Link on the N6: debug yes, flash no ( for this board ).** SEGGER
    supports the N657X0 ( V8.x ) but lists only a quad SPI external flash pin
    map; no octal MX66UW1G45G loader and no .stldr support, so CubeProgrammer
    plus STLINK stays the flash path. The BOOT1 dance and FSBL signing are boot
    ROM properties no probe removes. A J-Link Ultra+ would add RTT, Ozone live
    watch and real breakpoints ( the app runs from SRAM ); the DK debug
    connector and solder bridge details still need UM3300 ( st.com was down ).

58. **Cloud reconnected on the new PC via ADC, no service account key needed
    ( 2026 07 02 ).** `gcloud auth application-default login` as the user plus
    `set-quota-project muller-motor-controller` gives firebase_admin and
    google-cloud-bigquery working credentials, completely sidestepping the org
    policy that blocks SA key minting. `dashboard\cloud_forwarder.py` ( behind
    `web_console.py --cloud` ) streams ~5 Hz decimated v3 telemetry into the
    auto created time partitioned `muller_motor.telemetry_v3` table and
    refreshes the Firestore `live/n6` doc about once a second; verified live
    against the board, 262 rows in 55 s. The forwarder is a pure downstream
    tap: non blocking enqueue on the BrainCore RX callback, worker thread,
    batching, backoff, a 50k row outage buffer. Caveat for the headless future:
    ADC is tied to the interactive user login; an unattended brain service
    should eventually get the SA key or workload identity.

59. **The Claude control surface is live ( 2026 07 02 ).** web_console.py
    serves GET /health, GET /telemetry and a bearer token POST /command
    ( token at dashboard\console_token.txt, gitignored; localhost only ) that
    reuses the WebSocket command encoder. Verified on hardware: a set_trim
    posted over HTTP was applied by the N6 and confirmed by telemetry readback.
    Remote commands appear in the operator journal tagged remote. History reads
    go straight to BigQuery ( muller_motor.telemetry_v3, ADC creds ) or the
    console /runs and /run endpoints; live state is /telemetry or the Firestore
    live/n6 doc. See docs\claude_interface.md. PowerShell gotcha: variables are
    case insensitive, $h and $H collide.

60. **The 5 inch panel renders PORTRAIT now ( 2026 07 14 ).** The RK050HR18 is
    mounted turned 90 degrees COUNTER CLOCKWISE, so the UI draws in a logical
    480x800 space and fb_idx transposes every pixel write into the UNCHANGED
    800x480 landscape scanout buffer ( PANEL_TURNED_CW 0; LTDC timing, layer
    window and pitch never change ). fb_rect and draw_glyph2bpp clip against
    LOG_W / LOG_H and write through fb_idx; draw_dashboard was relaid two lines
    per channel. A CLOCKWISE build read upside down, which fixed the mount
    direction. Set PANEL_TURNED_CW 1 for a clockwise mount.

61. **The Hall double counting is FIRING INDUCED, not sensor side ( 2026 07 14,
    decisive bench test ).** With only CH1 firing, the disabled CH2..CH5 read
    1.5x to 2x the true edge rate; the instant firing stops ( DISABLE ALL,
    rotor coasting ) ALL five channels snap to identical clean readings
    ( ratio 1.00 ). So the drive flyback transient couples into the OTHER
    channels' sensor inputs and injects a spurious edge; the sensors and the
    Hall pull ups are fine. Real fix is drive side: flyback snubber at the coil
    / gate driver, sensor wire routing / shielding away from the coil wiring,
    CD40106B input RC ( 1k + 10nF ). CH1 ( the firing channel ) stays clean, so
    5 channel operation likely makes it worse ( every channel an aggressor ).
    The chain is a 5 V active low open collector Hall into a CD40106B Schmitt
    INVERTER at 3.3 V: two inversions CANCEL, so the N6 pins are active high
    ( HALL_CHAIN_INVERTED 0, rising edge, pulldown ); a single inversion
    assumption showed all five inputs asserted at once on a parked rotor.

62. **CPU is 600 MHz not 800, and the TIM6 output timer was silently dead
    ( 2026 07 14 ).** SystemClock_Config ( from the LTDC example ) gives CPUCLK
    = IC1 = PLL1/2 = 600 MHz, so the DWT cycle counter wraps every 7.16 s, not
    5.37 s. The 20 kHz TIM6 output service ( drive_outputs + M4 watchdog, off
    the main loop's mercy ) never ran: HAL_RCCEx_GetPeriphCLKFreq(RCC_PERIPHCLK
    _TIM) has NO case for the TIM group and silently returns 0, tripping drive
    _timer_init's guard. Fix: HAL_RCCEx_GetTIMGFreq() = SYSCLK ( IC2 = PLL1/3 =
    400 MHz ) >> TIMPRE = 400 MHz; PSC 399 gives a 1 MHz count, ARR 49 gives
    20 kHz. svc_per_s telemetry went 0 to 20000, overlap_forces dropped ~85
    percent, and the watchdog false trips ( red channels on the panel ) stopped.

63. **Firmware hardened against noise induced phantom firing ( 2026 07 14 ).**
    on_rotor_edge rejects edges < 5 ms apart ( glitch gate, checked in CYCLES
    so tick quantization can not leak sub 2 ms doublets ), reseeds ( no period,
    no pulse ) on gaps > 2 s, and fires only when the new period sits inside a
    FACTOR OF TWO band of the previous ( a tight agreement gate STARVED spin up,
    bench proven, because a strong pulse near stall legitimately steps the
    period hard ). Absolute 250 ms pulse ceiling in any mode. The M4 stuck high
    watchdog judges each pulse against ITS OWN latched allowance ( scheduled
    width + 10 ms service margin ), not the live g_period_cyc a mid pulse noise
    edge can shrink. The fault latch now BLOCKS scheduling and clears only on an
    explicit channel re enable. Channels boot DISARMED and the NOR restore never
    re arms ( the proven Giga contract ).

64. **RPM estimator: median of ENABLED live channels, robust to double counting
    ( 2026 07 14 ).** compute_telemetry ages liveness in MILLISECONDS
    ( HAL_GetTick ) not cycles ( the 7.16 s cycle wrap resurrected stale
    channels 1 s out of every wrap ). RPM comes from the ENABLED channels only
    ( a disabled channel may be disabled BECAUSE its sensor is bad; with only
    CH1 armed, the disabled CH4/CH5 double count dragged the reading to ~2x ),
    falling back to all live sensors when nothing is armed. Within the set it
    takes the LONGEST period at least two channels agree on within 20 percent:
    flyback only ADDS edges ( shortens periods ), so a clean channel reports
    truth and doublers are outvoted; a plain median failed once 3+ channels
    doubled. The old two sided 2x plausibility gate LATCHED the reading high ( a
    true drop looked implausible and was rejected too ) and was removed.

65. **Telemetry carries a build id so the running firmware is PROVABLE
    ( 2026 07 14 ).** fw_build = FNV-1a of __DATE__ " " __TIME__ at boot, in the
    v4 extension. fw_build 0 = pre build id firmware; a change after a flash
    proves the new image booted; unchanged proves the flash did not take. The
    expected value is extractable from the signed .bin ( grep the compile
    timestamp string, FNV hash it ), so a flash is verifiable to the exact
    build. External XSPI read back is unreliable on this board, so the
    programmer exit code plus this id are the trust anchors. Telemetry is v4
    now ( 152 B + fw_build ); svc_per_s, loop_per_s, tim_clk_mhz, fault_bits,
    glitch_rejects, band_rejects, overlap_forces, fire_count all added.

66. **Brain side operator stack rebuilt ( 2026 07 14 ).** web_console runs
    DETACHED ( Start-Process, NOT the preview harness which killed it twice
    with no log ); logs to build\web_console.{out,err}.log. New modules:
    activity_log ( JSONL, 5 Hz + every state change / command / fault /
    reboot ), firehose ( EVERY 50 Hz frame to build\brain_logs for post run
    analysis ), autotune ( in console async gradient P/D fire point tuner,
    dynamic 1..3 s dwell, RPM SCALED step cap 0.25 deg at speed because a big
    trim move at ~670 rpm drives the pulse against the rotor and throws severe
    flyback ). The page is served no store so a plain reload always shows UI
    changes.

67. **Trims are DERIVED from the placement instrument plus geometry, and the
    motor RAN ( 2026 07 14 ).** ch_phase_c10 ( each channel's Hall edge phase
    vs CH1, same period pairs, circular smoothed ) measured the sensor
    placement map to a tenth of a degree across three spins at different
    speeds. Trim = coil sector position ( 5 coils at 72 deg gives sector
    offsets 0/27/9/36/18 ) minus measured sensor position plus CH1's anchor
    41.4; the model reproduced CH2's legacy trim to 0.15 deg. On the computed
    set the machine SUSTAINED ~610 rpm on CH1 alone ( four channels agreeing,
    155 us jitter ), the first self sustained proper direction run. Tune trims
    SMALL at speed ( large steps throw flyback / braking ); phase_base is the
    global fine knob. The RPM target slider is a NO OP ( M7 PID unbuilt );
    pulse width ( 0x01/0x02, on the dashboard ) is the real open loop throttle.

68. **Input and output power on a Rigol MSO5074 over ISOLATED Ethernet
    ( 2026 07 14 ).** Scope at 192.168.1.225 ( LAN, the mini PC second NIC; the
    192.168.10.x NIC stays the N6 link ), raw SCPI port 5555. Ethernet is
    REQUIRED not just convenient: its magnetics keep the mains powered brain
    isolated from the motor side the scope probes touch; a USB tether would bond
    those grounds ( CLAUDE.md isolation rule ). MATH1 = CH1 x CH2 = input power
    ( CH1 voltage probe, CH2 Hantek CC-65 at 10 mV/A = 100 A/V ); MATH2 =
    CH3 x CH4 = output power ( CH3 output voltage probe, CH4 Tektronix A622 at
    10 mV/A ). dashboard\rigol_ingest.py polls VAVG of the math ( the per
    acquisition average, NOT the statistics running average which never forgets
    and lags load ), applies the A per V scale in software, EMA smooths, drives
    the Input / Output / COP tiles. Scope tuned: 20 MHz bandwidth limit both
    channels ( kills flyback HF ringing that swung the average 7x ), HighRes
    acquisition, long timebase to average many whole fire cycles at low rpm.
    Rigol raw socket reliably answers only the FIRST query per connection, so
    the ingest opens one short connection per query with a pacing gap and
    retries; keep it to a few queries per cycle at ~1 Hz or reads start
    dropping.

69. **The current clamps carry large DRIFTING DC offsets that MUST be tared
    ( 2026 07 14 ).** Hall clamps ( CC-65, A622 ) output a DC offset at zero
    current that drifts with position, temperature and stray rotor field ( the
    A622 read ~34 mV in free air, ~87 mV near the machine ). At bus voltage
    that offset becomes tens to hundreds of PHANTOM watts ( CC-65 -15 mV gives
    ~35 W input phantom; A622 -34 mV gives ~169 W output phantom and a garbage
    COP of 4.9 ). A software TARE ( power_tare command, dashboard button,
    persisted to build\rigol_tare.json so a console restart does not silently
    untare ) captures the present raw as the offset and subtracts it. CRITICAL:
    tare with the bus ENERGIZED, the rotor STOPPED ( zero real current, offset
    visible ), and the reading SETTLED ( it ramps a few seconds after the bus
    energizes; taring mid ramp lands wrong, seen twice ). A dead bus tare
    captures ~0 and nulls nothing. Clamps are ISOLATED ( no grounding rules ):
    clamp the single HOT lead, away from the rotor, arrow toward the load. But
    the two VOLTAGE probes ( CH1, CH3 ) share scope ground, so their clips must
    sit on the same node or genuinely isolated sections ( input and output here
    share NO node ). A floating voltage probe clip reads plausible garbage
    ( CH3 read 21.5 V unclipped vs 25 V clipped: the clip did not change the
    circuit, it gave the measurement a reference ). A622 needs its own battery
    powered ON ( a dead flat reading that ignores the balance knob = it is off ).
