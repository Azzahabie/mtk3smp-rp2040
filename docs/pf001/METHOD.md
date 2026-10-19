# PF-001 test method

## Test code
Two commits on this branch, on top of upstream `main` @ `2e8e6ad`. Neither is
meant for merging.

| Commit | Used for | Change |
|---|---|---|
| `test(pf001): measure the TIMER rate and boot path at start-up` | runs 1-3, run 4 (hotspot) | see below |
| `test(pf001): repeat the boot-time values on the final "done" line` | run 4 (router) | the last line also prints the boot-time values |

**`kernel/sysdepend/pico_rp2040/cpu_clock.c`, in `startup_clock()`, right after
`clk_ref` is switched to the XOSC:**
1. Copy WATCHDOG TICK (`0x4005802C`) into WATCHDOG SCRATCH0 (`0x4005800C`).
   This preserves the value the boot path left, before anything changes it. RAM
   is not initialised this early, so a scratch register is used.
2. Store `PF001_APPLY_FIX` in SCRATCH1 (`0x40058010`), so the image reports
   which build it is.
3. If `PF001_APPLY_FIX` is 1, write `(1<<9) | XOSC_MHz` to WATCHDOG TICK. This
   is the same register and value as the fix commit.

**`app_program/app_main.c`, `pf001_report()`:** this runs once in the monitor
task, after the demo banner.

```
[pf001] upstream main, fix compiled IN|OUT                    ← SCRATCH1
[pf001] tick_at_boot=… (enable cycles running)  tick_now=…    ← SCRATCH0, live TICK
[pf001] wd_reason=…  chip_reset=… (por run psm_restart)       ← WATCHDOG REASON, CHIP_RESET
[pf001] window n/6: kernel=10010 ms timer=… us -> … us/ms     ← 6 × 10 s
[pf001] done: fix compiled …, tick cycles now=… | at boot: cycles=… wd_reason=… por=… run=…
```

Each window reads the TIMER (`TIMERAWL`, `0x40054028`) against the kernel clock
(`tk_get_otm`, driven by SysTick from the processor clock, not by the watchdog
tick). A healthy TIMER reads ~1000 µs per kernel ms. The kernel's 10 ms tick
granularity accounts for the small offset (`kernel=10010`).

### Reading the `done` line
| `at boot: cycles` | `wd_reason` | `tick cycles now` | Meaning |
|---|---|---|---|
| 12 | 0x1 | 12 | BOOTSEL reflash: the boot ROM programmed the tick, and its watchdog reboot sets REASON |
| 0 | 0x0 | 0 | power-on / RUN reset, **no fix**: the TIMER runs at 23.4 µs/ms |
| 0 | 0x0 | 12 | power-on / RUN reset, **fix**: `startup_clock()` programmed the tick |

`por=1` appears on reflash boots too, because the BOOTSEL procedure begins with
an unplug. `wd_reason` and `at boot: cycles` are what tell the paths apart.

### Why the boot values are repeated on the `done` line
The header lines are printed about 0.5 s after USB enumeration. The firmware
waits for enumeration, not for the host terminal to reopen the port, and bytes
sent before Serial Monitor reconnects are lost. On many boots in runs 1-3 and
the hotspot run, the header was lost, and for those boots the boot path relied
on the operator's notes. The `done` line comes ~60 s later and is always
captured, so it was extended before the router run. The values are the ones
read at the top of `pf001_report()`, so printing them late does not change
them.

## The two images
Every pair of images was built from the same tree with one difference:

```c
#define PF001_APPLY_FIX 0   /* "nofix" / "fix0" image */
#define PF001_APPLY_FIX 1   /* "fix"   / "fix1" image */
```

## Builds
All builds used the upstream makefiles in `build_make/` with Arm GNU Toolchain
13.2.rel1 and Pico SDK 2.2.0, in a Docker container. There were no warnings.

| Run | Images | Command (in `build_make/`) | Output |
|---|---|---|---|
| 1-3 | `pf001_upstream_nofix.uf2`, `pf001_upstream_fix.uf2` | `make SMP=1 CONSOLE=usb_cdc -j8` | `mtk3pico_smp1_usb_cdc.uf2` |
| 4 hotspot | `pf001_run4_dhcp_fix0.uf2`, `…_fix1.uf2` | `make SMP=1 CONSOLE=usb_cdc WIFI=cyw43 WIFI_JOIN=1 WIFI_NETIF=1 WIFI_DHCP=1 -j8` | `mtk3pico_smp1_usb_cdc_wifi_dhcp.uf2` |
| 4 router | `pf001_run4_router_fix0.uf2`, `…_fix1.uf2` | same as above, with the second test commit | same |
| – | (build check only) | base and DHCP profiles on `fix/pf001-watchdog-tick`, the proposed commit alone | both built, no warnings |

Wi-Fi builds need `config/wifi_credentials.h`, which is gitignored. Run 4 used
`UTK_WIFI_AUTH = CYW43_AUTH_WPA3_WPA2_AES_PSK`.

The Wi-Fi images are not published because they contain the network password.
The two base test images are on the Release, with SHA-256 checksums:

| Image | SHA-256 |
|---|---|
| `pf001_upstream_nofix.uf2` | `aa80c1261002ff3ee9dcfa9314246941d4f8dbb835cfcf4a8bee7b17206ba7eb` |
| `pf001_upstream_fix.uf2` | `7721ec2d01877508377f5b71f06dd70ba6d364451074e9d573709189ecf43df6` |

## Boot procedures
Console: USB-CDC, opened in Serial Monitor at 115200 baud (the baud rate
has no effect on CDC).

| Path | Procedure | Expected registers |
|---|---|---|
| Reflash | unplug USB, hold BOOTSEL, plug in, copy the `.uf2` to the drive | `wd_reason=0x1`, tick cycles 12 |
| Power-cycle | unplug the USB cable at the PC end, wait 1-2 s, plug back in | `wd_reason=0x0`, tick cycles 0 |
| RUN reset (runs 1-3) | press the RUN/reset button on the maker board | `wd_reason=0x0`, tick cycles 0 |

The first power-cycle of each set follows the reflash. The rest follow one
another with no reflash in between.

## Networks
| Run | Network | Notes |
|---|---|---|
| 4 hotspot | phone hotspot | rejects every second join, with or without the fix. See [ANALYSIS.md](ANALYSIS.md#3-phone-hotspot-run) |
| 4 router | ASUS router, 2.4 GHz only, 802.11ax, WPA2/WPA3-Personal, AES | the PC was on the same network for the ping checks |

## Reproducing
Step-by-step build and test instructions are in [REPRODUCE.md](REPRODUCE.md).
