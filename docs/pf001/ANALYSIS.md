# PF-001 analysis

## 1. The register fault (runs 1-3, no Wi-Fi)
[logs/run1-3_timer_tick.md](logs/run1-3_timer_tick.md), 29 boots:

| Image | Reflash | Power-cycle | RUN button |
|---|---|---|---|
| no fix | 999.874-999.881 µs/ms (5/5) | **23.433-23.434** (5/5) | **23.434** (5/5) |
| fix | 999.873-999.882 (5/5) | 999.881 (5/5) | 999.881-999.883 (4/4) |

- 23.4375 µs/ms is exactly 12 MHz ÷ 512 per 1000 µs. A WATCHDOG TICK CYCLES
  field of 0 divides by 512, and the measured rate matches that to four digits.
- The captured headers give the same picture. Reflash boots show
  `tick_at_boot ... cycles=12`. No-fix power-cycles show `cycles=0`, and the
  register stays at 0. A fix power-cycle shows `tick_at_boot ... cycles=0`, then
  `tick_now ... cycles=12`. On a power-on boot the register starts
  unprogrammed, and the fix is what programs it.
- The RUN-pin reset behaves like a power-on, so the fault isn't limited to
  pulling the cable.

## 2. The DHCP consequence (run 4, router)
[logs/run4_router.md](logs/run4_router.md), 40 boots. Every boot's path is
taken from its own `done` line ([METHOD.md](METHOD.md#reading-the-done-line)):

| Image | Path | `done` line | Joined | IP ≤ 60 s | Ping |
|---|---|---|---|---|---|
| no fix | reflash ×10 | `at boot: cycles=12 wd_reason=0x1`, now 12 | 10/10 | 10/10 | ✅ |
| no fix | power-cycle ×10 | `at boot: cycles=0 wd_reason=0x0`, now **0** | 10/10 | **0/10** | ❌ |
| fix | reflash ×10 | `at boot: cycles=12 wd_reason=0x1`, now 12 | 10/10 | 10/10 | ✅ |
| fix | power-cycle ×10 | `at boot: cycles=0 wd_reason=0x0`, now **12** | 10/10 | **10/10** | ✅ |

### Cross-examination
- **Only the fix changes.** Each pair has the same source, profile,
  credentials, board and network. They differ only in `PF001_APPLY_FIX`, i.e. one
  register write.
- **The router was healthy throughout.** The no-fix reflash set (ending 01:11)
  and the fix reflash set (starting 01:25) both got an IP. They ran immediately
  before and after the 0/10 no-fix power-cycle set, so the failures were not a
  router outage. The same MAC got the same lease, 192.168.50.8, on every
  successful boot.
- **The board fails exactly where the hypothesis predicts.** On slow-timer
  boots, association still completes: `link UP` 10/10, and the router's client
  list refreshed the Pico's access time. Association runs in the CYW43 firmware.
  DHCP does not complete, because it runs on lwIP timers built on `sys_now()`,
  i.e. on the TIMER.
- **There is a retry for the slow timer to stretch.** On good boots, `ip=`
  arrives 6, 9 or 12 s after `link UP`. Those are multiples of the 3 s
  status-poll period (`WIFI_IDLE_MS`, timed by the kernel clock), so the first
  DHCP exchange usually needs at least one retransmit. At 23.4 µs/ms, lwIP's
  2 s DHCP retry takes ~85 s, which is past the 60 s measurement.
- **Statistics.** On the two power-cycle rows, 0/10 vs 10/10 gives a
  two-sided Fisher exact p ≈ 1.1 × 10⁻⁵.

## 3. Phone hotspot run
[logs/run4_hotspot.md](logs/run4_hotspot.md), 40 boots. On a phone hotspot
both images joined on only about every second boot, on every boot path:

| Image | Path | Joined | IP, of those that joined |
|---|---|---|---|
| no fix | reflash ×10 | 6 | 6/6 |
| no fix | power-cycle ×10 | 5 | **0/5** |
| fix | reflash ×10 | 5 | 5/5 |
| fix | power-cycle ×10 | 5 | **5/5** |

- The alternation appears with and without the fix and on reflash boots, so it
  isn't caused by the TIMER. On the router it didn't occur at all (40/40 joined
  first time). It comes from the phone's access point, not from this fault, and
  it is outside the scope of PF-001. Only boots that joined can show the DHCP
  result, so the table counts those.
- Among the boots that did join, the hotspot result matches the router:
  0/5 without the fix, 5/5 with it.
- One side observation: on slow-timer boots, `link UP` took ~12 s from the
  banner instead of ~6-9 s. This fits the CYW43 driver's own polling and
  timeouts also being TIMER-based. It doesn't affect the result.

## 4. Limits
- "No IP" means no IP within the 60 s window. At 43× slow, lwIP's retries
  could eventually succeed after several minutes. That wasn't tested, and
  either way a board that needs minutes to get an address after every power-on
  isn't usable.
- DHCP was tested on power-cycle, not on the RUN button. Runs 1-3 show that the
  RUN button leaves the register in the same `cycles=0` state.
- The hardware runs used the fix as `#if PF001_APPLY_FIX` test code. The
  commit proposed for merge makes the identical write (`(1<<9) | 12` to
  `0x4005802C`) at the identical point in `startup_clock()`, right after
  `clk_ref` is switched to the XOSC. It was built in both profiles with no
  warnings, but it was not run separately on hardware. The only other
  difference is the test code's two WATCHDOG SCRATCH writes, which don't touch
  the tick generator.
- One board and two networks were tested.
- Some console headers were lost to USB-CDC reconnect timing
  ([METHOD.md](METHOD.md#why-the-boot-values-are-repeated-on-the-done-line)).
  On the router run every boot still has a complete `done` line.

## 5. Background
This was first found while building a downstream application on this port.
There, the board joined but never got an IP after a power-on. The same pattern
(0/5 before the fix, 7/7 after) was measured on that firmware first. All the
runs in this folder were then repeated on unmodified upstream `main`, so the
result does not depend on the downstream changes.

## Conclusion
On upstream `main`, a power-on or RUN-pin boot leaves the TIMER ~43× slow,
because `startup_clock()` never programs WATCHDOG TICK. With Wi-Fi, the board
associates but does not get an IP address (0/10). Adding the one register write
that the Pico SDK also performs restores a 1 MHz TIMER on every boot path. With
it, DHCP succeeds on every boot (10/10). The network stack was not changed.
