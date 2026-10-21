# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Duo Open: an Android app (Kotlin, Jetpack Compose, single `:app` module) that plays a frosted-glass "fold" effect system-wide on book-style foldables, driven by the hinge angle sensor, no root. The effect is an AGSL shader (`app/src/main/res/raw/duo_unfold.agsl`) applied to a screenshot of the display, drawn in a touch-transparent accessibility overlay. A live-wallpaper mode does the same effect on the wallpaper only.

`README.md` is the user-facing spec. It covers device behaviour (OnePlus Open, Galaxy Z Fold 8), known platform limits, and the Shizuku mode. Read it before changing fold timing or panel logic. `SURFACE_DUO_PORT.md` (in Russian) holds the in-progress notes for porting to Surface Duo 1 (API 32). That port has to lower `minSdk` from 33 and replace AGSL/`RuntimeShader` with GLSL ES.

## Build / run

```
./gradlew assembleFullDebug     # full edition (accessibility overlay + wallpaper + Shizuku)
./gradlew assembleLiteRelease   # lite: wallpaper only, app id com.duoopen.lite
./gradlew assembleRelease       # both; signed with keystore.properties if present, else debug key
./gradlew installFullDebug
./gradlew lintFullDebug
```

There are no unit or instrumented tests. You verify changes on a device:

```
adb shell settings put secure enabled_accessibility_services com.duoopen/com.duoopen.overlay.FoldOverlayService
adb shell am broadcast -a com.duoopen.DEMO     # replay the effect without folding
adb logcat -s DuoOverlay
adb shell settings put global duoopen_test_displays 1   # accept emulator "simulated secondary display" as a panel
```

The toolchain is JDK 17, AGP 8.9, Kotlin 2.2, compileSdk/targetSdk 35, and minSdk 33 (because AGSL `RuntimeShader` requires API 33).

## Architecture

**Flavors (`edition` dimension).** `src/main` is shared. `src/full` adds the accessibility service, overlay engines, Shizuku shell helper, and setup guide. `src/lite` contains stubs only. Shared code reaches flavor-specific features only through `com.duoopen.overlay.OverlayFeature`, which is defined in both `src/full` and `src/lite` with the same API (`AVAILABLE`, `playDemo()`, Shizuku status…). If you add a capability that shared UI needs, add it to both `OverlayFeature` objects. Full-only dependencies use `"fullImplementation"`.

**Fold core (`src/main/.../fold/`)** is shared by three renderers: the in-app preview (`ui/FoldEffect.kt`, `ui/HomePreview.kt`), the live wallpaper (`wallpaper/DuoWallpaperService.kt`), and the overlay (`src/full/.../overlay/FoldOverlayView.kt`).
- `DuoShader`: the glue for the AGSL file. It handles uniforms, maps hinge angle to pane tilt, and places the fold line (`FoldLine`).
- `HingeAngleSource`: picks the hinge sensor (vendor fallbacks), detects "stops-only" coarse sensors (Samsung 0/90/180), and produces the "Copy sensor report".
- `TiltFollower`: a per-vsync ease that smooths over 1° sensor steps. On coarse sensors it plays timed eases instead.
- `Panels.kt`: decides inner vs cover panel from the display mode, and which displays count as live panels.

**Settings.** `settings/DuoSettings.kt` stores one `DuoConfig` in SharedPreferences, exposed as a `StateFlow`. The Tune sheet (`ui/ControlSheet.kt`), the wallpaper, and the overlay all read it live. A new tuning knob goes there and gets documented in the `DuoConfig` KDoc.

**Overlay (full).** `FoldOverlayService` (AccessibilityService) watches `DisplayManager` and runs one `PanelEngine` per lit built-in display. `PanelEngine` is the largest file and holds the per-display state machine: hinge tracking, fold phases (cover frosts in, then panel swap, then inner clears), capture timing around the ~0.4 s panel-reveal blackout and the ~333 ms screenshot rate limit, and the tent-mode fade-out. `FoldSurface` is either a snapshot surface drawn through the shader (`FoldOverlayView`) or the "Live blur" strip windows that use SurfaceFlinger cross-window blur. `OverlayState.running` tells the wallpaper and the preview to skip their own pass while the overlay is live, so the effect isn't drawn twice.

**Shizuku mode (full, optional).** `ShizukuBridge` (app side) binds `DuoShellService`, a Shizuku user service that runs with ADB shell privileges in a separate process. They talk over a hand-rolled Binder protocol: transaction codes in `ShellProtocol`, Bundles in and out. The shell service provides fast display capture that excludes our own overlay layer (the layer id is read via hidden API, hence `HiddenApiBypass` in `OverlayFeature.initProcess`), and it tails logcat for Samsung's fold wallpaper angle lines. `WallpaperAngleFeed` pings that wallpaper about 30×/s to make it log. Everything must fall back cleanly to the accessibility screenshot and the public sensor when Shizuku is absent.

**`DualScreen.kt`** uses the Jetpack Window Area API to request concurrent (both-panels-on) mode, so the both-screens engine path can be exercised.

## Conventions

- Logging tag for the overlay is `DuoOverlay`. Bug reports in the README depend on that tag, so keep using it.
- The app has no network permission, and the README's Privacy section promises that captures stay in memory only. Don't break either.
- Release signing reads an untracked `keystore.properties` (`storeFile`, `storePassword`, `keyAlias`, `keyPassword`).
- Techniques borrowed from other projects are credited in the README "Credits" section and in `THIRD_PARTY_NOTICES.md`. Keep those in sync when you add a dependency or a borrowed technique.
