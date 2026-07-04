# Session 8 - RTC Clock Widget

The M5StickC Plus 2 has a BM8563 RTC chip on its I2C bus that can keep time even when the device is powered off (it has a separate coin cell battery). This session adds a clock readout to the Stats screen.

## What was built

A `ClockManager` class that wraps the M5.Rtc API. `begin()` seeds the RTC with a hardcoded starting time (12:00) since there's no NTP or user input yet. `getCurrentTime()` reads the RTC registers and writes hours and minutes into out-parameters. The DisplayManager calls it once per render cycle and draws `HH:MM` in the top area of the Stats screen, between the stat bars and the mood word.

The M5.Rtc API is simple - `M5.Rtc.setTime()` sets the time, `M5.Rtc.getTime()` reads it into an `m5::rtc_time_t` struct with hours, minutes, and seconds fields. No I2C protocol details are exposed. The clock ticks forward as long as the RTC battery holds, which means the time survives reboots and power loss.

The feature is guarded by `ENABLE_MULTISCREEN` since the clock only appears on the Stats screen. The `ClockManager` is instantiated in `src/main.cpp` alongside the other managers, `begin()` is called in `setup()`, and `getCurrentTime()` is called at the top of the main loop's render path. The hours and minutes are passed down to `drawStatsScreen()` which formats them with `snprintf("%02d:%02d")`.

## Clamping

The RTC can return uninitialised data if the coin cell is dead or the chip has never been set. `getCurrentTime()` clamps the values to valid ranges (0-23 for hours, 0-59 for minutes) and falls back to 12:00 if anything looks wrong. This keeps the display from showing garbage without needing a separate initialisation check.

## Stretch ideas

- Button-set time. Long-press A on the Stats screen to enter time-set mode, then B/C cycle hours and minutes, A confirms.
- Persist to NVS. Save the last-known timestamp on every minute rollover so reboots don't reset to 12:00 even if the RTC battery fails.
- Overnight decay. Use the RTC to calculate elapsed time since the pet was last powered on and apply accumulated stat decay for that duration. Needs Unix timestamp math and a cap (a week offline shouldn't kill the pet instantly).
- NTP sync. If the WiFi mode ever supports station mode (instead of just AP), fetch the real time from an NTP server.

## New concepts

- I2C peripheral access through the M5.Rtc wrapper (one-line API, no protocol details)
- Time as a struct with hours/minutes/seconds fields
- Out-parameters (`int& hours, int& minutes`) instead of returning a struct

## Files touched

- `lib/Clock/clock_manager.h` and `lib/Clock/clock_manager.cpp` - new
- `src/main.cpp` - instantiate and wire up
- `lib/Display/display_manager.cpp` - add the time row to the Stats screen

No hardware changes needed - the RTC is already on the board.
