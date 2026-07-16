# Color Tuner

A system-wide display color correction app for **rooted Android 14** devices (developed and tested on Pixel 4a 5G / Snapdragon 765G). Adjusts color temperature, saturation, gamma, and per-channel RGB correction across the entire display — not just within apps, but at the compositor level — without an overlay, without polling, and without root-level library injection.

---

## Features

- **Color temperature** — 1000 K (candlelight) to 10 000 K (cool daylight), log-scale slider centered at 6500 K neutral
- **Saturation** — 0 % (grayscale) to 100 % (full), via Android's `ColorDisplayManager`
- **Gamma** — 4.0 (very dark) to 0.25 (very bright), with a midtone-correcting linear approximation
- **RGB correction** — independent red, green, blue channel scaling (0–100 %)
- **Profiles** — named presets with one-tap loading; built-in profiles for common scenarios
- **Persistent** — survives app close, screen off/on, and reboots via a foreground service and boot receiver
- **No overlay** — not a coloured sheet on top of the screen; actual compositor-level transform
- **No polling loop** — zero CPU when idle; reapplies only on screen-on events

---

## How It Works

### SurfaceFlinger color matrix (temperature, gamma, RGB)

The core of the app is a single `service call SurfaceFlinger 1015` binder transaction that injects a 3 × 4 floating-point color matrix directly into Android's display compositor. The matrix is applied at the SurfaceFlinger level — below all apps, at the point where rendered frames are composited before being sent to the display hardware.

The correct invocation format, discovered through iterative testing on Android 14, is:

```
service call SurfaceFlinger 1015 i32 1 f m0 f m1 f m2 f m3 f m4 f m5 f m6 f m7 f m8 f m9 f m10 f m11
```

The `i32 1` prefix is required on Android 12+; sending 16 raw floats (the format documented for older Android versions) produces incorrect results. The 12 floats form a column-major 3 × 4 matrix:

```
R' = m0·R + m3·G + m6·B + m9
G' = m1·R + m4·G + m7·B + m10
B' = m2·R + m5·G + m8·B + m11
```

This transform is applied once and persists indefinitely — surviving screen off/on — with no further intervention needed until the device reboots. A `BroadcastReceiver` for `ACTION_SCREEN_ON` reapplies the matrix on wake in case SurfaceFlinger resets it.

All four adjustments (temperature, gamma, RGB, saturation) are composed into a single matrix multiplication per update, so they are applied atomically in one pass with no stacking overhead.

### Color temperature

Temperature uses the **Tanner Helland blackbody approximation** for the warm side (1000–6500 K), normalized so that 6500 K (D65) produces an exact identity matrix. For the cool side (6500–10 000 K), a symmetric linear model reduces the red channel from 1.0 → 0.0, matching the visual impact of the warm side where green and blue reduce to zero at the extreme. The temperature slider uses a log scale in both directions from the 6500 K center, giving finer control near neutral and coarser steps at extremes.

Note: the Pixel 4a 5G in `ColorMode::NATIVE` bypasses standard color management, exposing the raw OLED panel's native white point (~5 500 K). The app's mathematical neutral is D65 (6 500 K); users may wish to save a personal "neutral" profile at a slightly warmer value to taste.

### Saturation

Saturation is encoded directly into the SurfaceFlinger color matrix alongside temperature, gamma, and RGB — no separate shell command, no polling, no watchdog. The desaturation blend is computed using equal (1/3, 1/3, 1/3) luminance weights per channel rather than the standard Rec. 709 weights (0.2126, 0.7152, 0.0722). The Rec. 709 weights are calibrated for sRGB-managed displays; in `ColorMode::NATIVE` they produce a visible green cast when desaturating, because the panel's actual perceptual weights differ from the standard.

During development, `cmd color_display set-saturation` was explored as a separate path and confirmed working on this device, but it requires a persistent watchdog because the `android` system package holds a competing saturation registration (`{android=0}` visible in `dumpsys color_display`) that intermittently reasserts and resets the value within about one second. Folding saturation into the SurfaceFlinger matrix sidesteps this entirely — the matrix persists until reboot with no intervention.

### Gamma

True gamma correction (`output = input^γ`) is a power curve — inherently non-linear — and cannot be represented exactly by a linear 3 × 4 color matrix. The app uses a linear approximation that correctly handles the midpoint:

- **Brightening (γ < 1):** scale = 2^(1−γ), offset = 0. Black is fixed at 0; midtone at 0.5 maps to exactly 0.5^γ; bright values clip naturally to 1.0.
- **Darkening (γ > 1):** scale = 2·(1 − 0.5^γ), offset = 2·0.5^γ − 1 (negative). White is fixed at 1.0; midtone maps to exactly 0.5^γ; shadows clip to 0.

The result is a piecewise linear function through (0, 0), (0.5, 0.5^γ), and (1, 1) — correct at all three reference points, with a linear interpolation between them rather than a true power curve. Equal slider travel near the neutral (γ = 1.0) produces smaller visual changes than at the extremes, which matches perceptual expectations.

### What we can't do (and why)

**cf.lumen's root driver** injects a shared library into the SurfaceFlinger process using ptrace, hooks the GPU compositing function at runtime, and redirects it through a custom color matrix. This happens at the fragment shader level — every rendered frame is intercepted. The result is pixel-perfect, per-frame color correction with no polling and no reset issues. Replicating this requires compiling native C++ ARM code, implementing ptrace-based library injection, and resolving SurfaceFlinger symbol addresses at runtime — far beyond what can be built with a standard Android Gradle toolchain on GitHub Actions.

**Kernel-level RGB/gamma** (KCAL, PCC nodes) — Qualcomm's SDE display driver exposes `/sys/class/drm/` sysfs nodes for polynomial color correction on some devices. These are absent on stock Pixel kernels; Google disables them. No accessible path to hardware LUT or PCC on this device without a custom kernel.

**`CONTROL_DISPLAY_COLOR_TRANSFORMS` permission** — the Java API `ColorDisplayManager.setSaturationLevel()` can register a persistent saturation owner, preventing Android's own system registration from overriding it. This permission is signature-level (granted only to apps signed with the platform key) and cannot be granted at runtime via `pm grant`, even with root.

---

## Architecture

```
MainActivity          — UI, slider ↔ EditText two-way binding, profile management
ColorService          — Foreground service; applies SurfaceFlinger matrix on start,
                        on ACTION_UPDATE intent, and on ACTION_SCREEN_ON broadcast
BootReceiver          — Restarts ColorService on BOOT_COMPLETED if it was running
```

The service sends a single `su -c 'service call SurfaceFlinger 1015 ...'` shell command per update. No JVM is spawned for the matrix call itself; the saturation watchdog spawns one root shell per interval for `cmd color_display set-saturation`.

---

## Requirements

- Rooted Android device (tested: Pixel 4a 5G, Android 14, Kitsune Mask / Magisk fork)
- Root access via `su` (Magisk or compatible) — **root is required**
- Android 12+ (SurfaceFlinger transaction 1015 with `i32 1` prefix)

Note: Shizuku alone is not sufficient. The `service call SurfaceFlinger 1015` transaction requires root (uid 0). The shell user (uid 2000) that Shizuku provides gets "Operation not permitted" when attempting to set the color transform. Shizuku support is included in the app for status reporting and future use, but the core color matrix function requires root.

## Building

Upload `ColorTuner.zip` to the repository root. GitHub Actions will unzip, build a debug APK, and attach it as a workflow artifact. No local Android Studio installation required.

```
Actions → Build APK → (green checkmark) → Artifacts → ColorTuner-debug → download → adb install
```

## Persistence across reboots

Enable the Color Service from the app. On reboot, `BootReceiver` restarts the service automatically (allow ~30 seconds for Magisk to fully initialize before `su` is available). The service reapplies the saved profile silently.
