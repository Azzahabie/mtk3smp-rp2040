# Reproducing PF-001

There are three ways to check this, from quickest to most complete:

| | What you need | Time | Shows |
|---|---|---|---|
| [A. Prebuilt images](#a-prebuilt-images-no-build) | a Pico W and a USB cable | ~5 min | the TIMER runs 43× slow after a power-cycle without the fix |
| [B. Build the test images](#b-build-the-test-images) | toolchain + Pico SDK | ~15 min | the same, built from source; optionally the DHCP failure |
| [C. Build the proposed fix](#c-build-the-proposed-fix-on-its-own) | toolchain + Pico SDK + a Wi-Fi network | ~15 min | `main` gets no IP after a power-cycle, and the fix branch does |

## The two boot paths
Every test compares these two ways of starting the board:

| Path | How | What the boot ROM leaves in WATCHDOG TICK |
|---|---|---|
| **Reflash** | unplug USB, hold **BOOTSEL**, plug in, copy the `.uf2` onto the `RPI-RP2` drive | CYCLES = 12, TIMER correct |
| **Power-cycle** | unplug the USB cable, wait 1-2 s, plug it back in (no BOOTSEL) | CYCLES = 0, TIMER 43× slow unless the fix sets it |

Pressing a RUN/reset button, if your board has one, behaves like a power-cycle.

> [!TIP]
> The console is USB-CDC, so the port disappears on every unplug. Lines printed
> in the first ~0.5 s after the port comes back can be lost before your
> terminal reconnects. The test images print a final `[pf001] done:` line about
> 60 s after boot that repeats everything needed, so wait for that line.

## A. Prebuilt images, no build
Download `pf001_upstream_nofix.uf2` and `pf001_upstream_fix.uf2` from the
Release `pf001-evidence`, and check them against `SHA256SUMS.txt`. Both are the
base profile, with no Wi-Fi and no credentials.

1. **Reflash** `pf001_upstream_nofix.uf2` and open the board's USB serial port.
   Wait about 60 s for the last window and the `done` line:
   ```
   [pf001] window 6/6: kernel=10010 ms timer=10009962 us -> 999.996 us/ms   cumulative 999.881 us/ms
   [pf001] done: fix compiled OUT, tick cycles now=12
   ```
2. **Power-cycle** the board, reopen the port, and wait about 60 s:
   ```
   [pf001] window 6/6: kernel=10010 ms timer=234610 us -> 23.437 us/ms   cumulative 23.434 us/ms
   [pf001] done: fix compiled OUT, tick cycles now=0
   ```
   This is the fault: the TIMER advanced 0.23 s while the kernel clock advanced
   10 s.
3. **Reflash** `pf001_upstream_fix.uf2`, then **power-cycle** it. Both boots
   show ~1000 us/ms and `tick cycles now=12`.

## B. Build the test images
### Prerequisites
These are the same as the port's own docs: [GETTING_STARTED.md §1 and
§1b](../GETTING_STARTED.md).
- `arm-none-eabi-gcc` (the results here used Arm GNU Toolchain 13.2.rel1), GNU
  `make`, and a host `g++`.
- Pico SDK 2.2.0. It is needed because the USB-CDC console and Wi-Fi use it.
  Pass `PICO_SDK_PATH=...` to `make` or export it.

### Build
```sh
git clone https://github.com/Azzahabie/mtk3smp-rp2040.git
cd mtk3smp-rp2040
git checkout test/pf001-evidence
cd build_make

# Image 1: fix compiled IN (the default)
make SMP=1 CONSOLE=usb_cdc PICO_SDK_PATH=/path/to/pico-sdk -j8
cp mtk3pico_smp1_usb_cdc.uf2 pf001_fix.uf2

# Image 2: fix compiled OUT
#   edit ../kernel/sysdepend/pico_rp2040/cpu_clock.c line 26:
#       #define PF001_APPLY_FIX 1    ->    #define PF001_APPLY_FIX 0
make SMP=1 CONSOLE=usb_cdc PICO_SDK_PATH=/path/to/pico-sdk -j8
cp mtk3pico_smp1_usb_cdc.uf2 pf001_nofix.uf2
```
Both builds write to the same output name, so copy each one away before
building the next. `PF001_APPLY_FIX` is the only difference between the two
images. Then run the steps in [A](#a-prebuilt-images-no-build) with your own
images.

### Optional: the DHCP failure with the test images
1. Create the credentials file:
   ```sh
   cp config/wifi_credentials.example.h config/wifi_credentials.h
   ```
2. Edit it: set `UTK_WIFI_SSID` and `UTK_WIFI_PASSWORD`, set
   `UTK_WIFI_CREDENTIALS_SET` to `1`, and set `UTK_WIFI_AUTH` to match your
   network. These runs used `CYW43_AUTH_WPA3_WPA2_AES_PSK`.
3. Build both images as above, but with the DHCP profile. The output is
   `mtk3pico_smp1_usb_cdc_wifi_dhcp.uf2`:
   ```sh
   make SMP=1 CONSOLE=usb_cdc WIFI=cyw43 WIFI_JOIN=1 WIFI_NETIF=1 WIFI_DHCP=1 PICO_SDK_PATH=/path/to/pico-sdk -j8
   ```
4. Power-cycle each image a few times and watch for these lines:

   | Image | Expected after a power-cycle |
   |---|---|
   | fix OUT | `[wifi] link UP ...`, then no `[wifi] ip=` line; 23.437 us/ms |
   | fix IN | `[wifi] link UP ...`, then `[wifi] ip=...` 6-12 s later; ~1000 us/ms |

## C. Build the proposed fix on its own
This compares unmodified `main` with `fix/pf001-watchdog-tick`, without any test
code. The credentials file from B is gitignored, so it stays in place when you
switch branches.

```sh
cd mtk3smp-rp2040/build_make

git checkout main
make SMP=1 CONSOLE=usb_cdc WIFI=cyw43 WIFI_JOIN=1 WIFI_NETIF=1 WIFI_DHCP=1 PICO_SDK_PATH=/path/to/pico-sdk -j8
cp mtk3pico_smp1_usb_cdc_wifi_dhcp.uf2 main_dhcp.uf2

git checkout fix/pf001-watchdog-tick
make SMP=1 CONSOLE=usb_cdc WIFI=cyw43 WIFI_JOIN=1 WIFI_NETIF=1 WIFI_DHCP=1 PICO_SDK_PATH=/path/to/pico-sdk -j8
cp mtk3pico_smp1_usb_cdc_wifi_dhcp.uf2 fix_dhcp.uf2
```

For each image: reflash it and confirm it gets `[wifi] ip=`. Then power-cycle
it several times.

| Image | After a reflash | After a power-cycle |
|---|---|---|
| `main_dhcp.uf2` | `link UP`, then `ip=` | `link UP`, **no `ip=`** |
| `fix_dhcp.uf2` | `link UP`, then `ip=` | `link UP`, then **`ip=`** |

These images have no `[pf001]` lines, so the `[wifi]` lines are the result.
Allow about a minute per boot.

The diff between the two branches is a single file:
```sh
git diff main fix/pf001-watchdog-tick
```
