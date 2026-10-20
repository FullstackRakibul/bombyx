# Bombyx — Project Summary

_Snapshot as of 2026-10-20 (commit `80c2837` "Ground Control")._

## Overview

**Bombyx** is an Android app for operating a **UHF RFID reader over UART (serial)** on rugged handheld terminals. It is built from the vendor's `uhf-uart-demo` (Gradle root project name) and drives the hardware through the vendor SDK `com.rscja.deviceapi.RFIDWithUHFUART` (bundled as `DeviceAPI_ver20251103_release.aar`).

It covers the full tag lifecycle: inventory, read/write, lock, kill, locating tags, LED/flash tags, NXP-specific commands, reader configuration, Excel export, and reader firmware upgrade.

| Item | Value |
|---|---|
| App name | Bombyx |
| Package / namespace | `com.example.uhf` |
| Version | `1.4.6` (versionCode 2) |
| Language | Java 8 (no Kotlin) |
| SDK levels | min 26 · target 33 · compile 34 |
| Build | Android Gradle Plugin 8.2.2, Gradle wrapper 9.7.1 |
| APK name | `UHF-serial_v1.4.6.apk` |
| Size | ~45 Java classes, ~10.3k lines |
| Languages (UI) | English (`values/`) and Chinese (`values-zh/`) |

## Repository Layout

```
bombyx/
├── build.gradle                 # Root: declares AGP 8.2.2
├── settings.gradle              # rootProject "uhf-uart-demo", includes :app
├── gradle.properties
├── gradle/wrapper/              # Gradle 9.7.1 wrapper
└── app/
    ├── build.gradle             # Module config, signing, dependencies
    ├── proguard-rules.pro
    ├── uhf-serial_release.jks   # Signing keystore (committed)
    ├── libs/                    # Local jars/aars
    │   ├── DeviceAPI_ver20251103_release.aar   # Vendor RFID SDK
    │   ├── jxl.jar                             # Excel (.xls) writing
    │   ├── poi-3.12-android-a.jar              # Apache POI
    │   ├── poi-ooxml-schemas-3.12-...jar
    │   └── xUtils-2.5.5.jar
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   ├── java/com/example/uhf/
        │   │   ├── AppContext.java            # Application; crash handler → /sdcard/uhf/error/
        │   │   ├── BootBroadcastReceiver.java # Auto-launch on BOOT_COMPLETED
        │   │   ├── ErrorCodeManage.java       # Reader error code → message
        │   │   ├── RadarTAGInfo.java, UhfInfo.java
        │   │   ├── activity/                  # Main tab host, base activity, speed test
        │   │   ├── fragment/                  # One fragment per feature tab
        │   │   ├── adapter/                   # ViewPager + LED-tag list adapters
        │   │   ├── filebrowser/               # Simple file picker (firmware files)
        │   │   ├── tools/                     # Excel export, file, string, toast, UI helpers
        │   │   ├── view/                      # Custom views: radar, location canvas, circle seekbar
        │   │   └── widget/                    # LazyViewPager, NoScrollViewPager
        │   └── res/                           # Layouts, strings (en/zh), sounds, drawables
        ├── test/                              # Template unit test only
        └── androidTest/                       # Template instrumented test only
```

## Architecture

- **Single-activity, tab-based UI.** [UHFMainActivity](app/src/main/java/com/example/uhf/activity/UHFMainActivity.java) uses a `FragmentTabHost` with one fragment per feature.
- **Shared reader instance.** [BaseTabFragmentActivity](app/src/main/java/com/example/uhf/activity/BaseTabFragmentActivity.java) owns `public RFIDWithUHFUART mReader`. It initializes the reader in an `AsyncTask` with an "init..." progress dialog, and calls `free()` in `onDestroy` (which then kills the process).
- **Fragments call the reader directly** via `mContext.mReader.*`. All fragments extend `KeyDwonFragment`, so the active fragment receives `myOnKeyDwon()` when a hardware **trigger/scan key** is pressed (key codes 139, 280, 291, 293, 294, 311–315, 591–596).
- **Audio feedback.** `SoundPool` beeps on success/failure, plus a variable-rate beep thread used for proximity feedback while locating tags.
- **Global crash handling.** `AppContext` catches uncaught exceptions, writes a crash log to `/sdcard/uhf/error/UHF-serial-<time>.txt`, then exits.
- **No persistence layer, no networking, no DI.** State is held in memory in the activity/fragments.

## Features (tabs, in order)

| # | Tab | Fragment | What it does | Key SDK calls |
|---|---|---|---|---|
| 1 | **Scan / Inventory** | `UHFReadTagFragment` | Single-shot or continuous inventory, with a timed auto-stop. Shows a tag list (EPC/TID/USER, count, RSSI, optional phase), tag count, elapsed time, and filtering by EPC/TID/USER bank. Can toggle EPC + tamper-alarm mode. | `inventorySingleTag`, `startInventoryTag`, `setInventoryCallback`, `stopInventory`, `setFilter` |
| 2 | **Settings** | `UHFSetFragment` | Output power, frequency region and custom hop, protocol (ISO 18000-6C / GB/T 29768 / GJB 7377.1 / ISO 18000-6B), RF link profile, session/Gen2 params, inventory memory bank mode (EPC / EPC+TID / EPC+TID+USER), FastID, TagFocus, fast inventory, factory reset. | `setPower`, `setFrequencyMode`, `setFreHop`, `setProtocol`, `setRFLink`, `setGen2`, `setEPCAndTIDUserMode`, `setFastID`, `setTagFocus`, `factoryReset` |
| 3 | **Radar Location** | `UHFRadarLocationFragment` | Radar-style view that finds a target EPC, showing direction and signal. Has a landscape layout. | `startRadarLocation`, `stopRadarLocation`, `setDynamicDistance` |
| 4 | **Location** | `UHFLocationFragment` | Finds a single tag by signal-strength gauge, with beep rate scaled to proximity. | `startLocation`, `stopLocation`, `setDynamicDistance` |
| 5 | **Tag Lit (LED tags)** | `UHFTagLitFragment` | Inventories, then makes selected LED-capable tags light up (`MODE_LED_TAG`), using a per-tag EPC filter list. | `setFilter(List)`, `setEPCAndTIDUserMode`, `startInventoryTag` |
| 6 | **Tag Flash** | `UHFTagFlashFragment` | Triggers a tag's light by reading the Reserved bank, with an optional filter. | `readData(... Bank_RESERVED ...)` |
| 7 | **Protected-ShortRange** | `ProtectedModeAndShortRangeModeFragment` | Toggles NXP UCODE Protected Mode and Short-Range Mode on a tag. | `setProtectedModeAndShortRangeMode` |
| 8 | **MarginRead** | `MarginReadFragment` | NXP margin read, which checks how reliably a memory region's data is written. | `marginRead` |
| 9 | **NxpUcodeDna** | `AuthenticateNxpUcodeDnaFragment` | NXP UCODE DNA cryptographic authentication (key ID + challenge). | `authenticateNxpUcodeDna` |
| 10 | **Read / Write** | `UHFReadWriteFragment` | Reads and writes any bank (Reserved/EPC/TID/USER) by pointer and length, with an access password and optional filter. | `readData`, `writeData` |
| 11 | **Lock** | `UHFLockFragment` | Builds a lock code through a dialog (lock / unlock / permalock per bank) and applies it. Includes GB (Chinese national standard) tag lock. | `lockMem`, `uhfGBTagLock` |
| 12 | **Kill** | `UHFKillFragment` | Permanently kills a tag using the kill password, with an optional filter. | `killTag` |
| 13 | **BlockWrite** | `BlockWriteFragment` | Writes data in blocks to a bank. | `blockWriteData` |
| 14 | **BlockPermalock** | `BlockPermalockFragment` | Permanently locks individual memory blocks, using a mask. | `uhfBlockPermalock` |
| 15 | **Upgrade** | `UHFUpgradeFragment` | Flashes reader firmware (app or hardware/bootloader) from a file chosen with the built-in file browser, and shows a progress bar. | `uhfJump2Boot`, `uhfStartUpdate`, `uhfUpdating`, `uhfStopUpdate` |

### Overflow menu

- **About.** Shows the reader's software and hardware versions.
- **Export.** Writes the current inventory list to an `.xls` file at `/sdcard/UHF_exportData/<yyyyMMddHHmmss>.xls`, with columns EPC, TID, USER, COUNT, RSSI. This uses `jxl` via `ExportExcelAsyncTask`, and is blocked while a scan is running.
- **Test rate.** Hidden unless `BuildConfig.Test = true`. Opens `TestActivity`, which is a read-rate benchmark that shows tags per second.

### Other behaviour

- **Starts automatically on boot** via `BootBroadcastReceiver`.
- **Storage permissions.** On Android 11+ the app requests *All files access* (`MANAGE_EXTERNAL_STORAGE`) and finishes until it is granted. On older versions it requests read/write external storage.

## Dependencies

- AndroidX AppCompat 1.7.0, Material 1.12.0, ConstraintLayout 2.1.4
- Local `libs/` (everything in it is included via `fileTree`): vendor DeviceAPI AAR, jxl, Apache POI 3.12, xUtils 2.5.5
- Tests: JUnit 4.13.2, AndroidX Test JUnit 1.2.1, Espresso 3.6.1 (template tests only)

## Build & Run

```bash
./gradlew assembleDebug        # → app/build/outputs/apk/debug/UHF-serial_v1.4.6.apk
./gradlew assembleRelease
```

The app has to run on a supported handheld device that has the UHF UART module. On a phone or emulator, `mReader.init()` fails and shows "init fail".

## Notes & Known Issues

- **Signing config path.** `app/build.gradle` sets `storeFile` to `new File("/uhf-serial_release.jks")`, which is the filesystem root. The keystore actually lives in `app/`. The keystore and its passwords (`123456`) are also committed to the repo.
- **Toolchain versions.** The project pairs AGP 8.2.2 with the Gradle 9.7.1 wrapper. Check that this combination builds; you may need to upgrade AGP or pin an 8.x Gradle.
- **Manifest version.** The manifest's `versionCode`/`versionName` (1 / 1.0) are overridden by Gradle (2 / 1.4.6).
- **Dead code.** `BaseTabFragmentActivity.exportData()` is a stub that just toasts "导出失败" (export failed). The real export is the override in `UHFMainActivity`. Several fragments and resources are unused, such as `fragment_temperature.xml`, `fragment_deactivate.xml`, the ping and upload strings, and the hidden kill/lock/set menu items.
- **Deprecated APIs.** The code uses `AsyncTask`, `ProgressDialog`, `FragmentTabHost`, and the `SoundPool` constructor. Paths are hardcoded to `/sdcard/...`.
- **Code comments.** Many comments are in Chinese, inherited from the vendor demo.
- **Tests.** There is no real test coverage.
