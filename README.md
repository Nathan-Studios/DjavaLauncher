# DjavaLauncher 2.0 — Complete Technical Reference

> **Project**: DjavaLauncher 2.0 (GTA SA Reversed Android)  
> **Root name**: `gtareversed`  
> **Package**: `com.nathan.djavarp`  
> **Version**: 1.5 (code 135)  
> **Min SDK**: 26 | **Target SDK**: 36 | **Compile SDK**: 36  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Build System](#3-build-system)
4. [ABI & Native Libraries](#4-abi--native-libraries)
5. [Native (C++) Layer](#5-native-c-layer)
6. [Java/Kotlin Layer](#6-javakotlin-layer)
7. [Package Reference](#7-package-reference)
8. [Android Resources](#8-android-resources)
9. [Dependencies](#9-dependencies)
10. [Permissions & Manifest](#10-permissions--manifest)
11. [ProGuard & Obfuscation](#11-proguard--obfuscation)
12. [Assets](#12-assets)
13. [Networking & API](#13-networking--api)
14. [Firebase Services](#14-firebase-services)
15. [Game Data & Downloader](#15-game-data--downloader)
16. [Crash Analysis](#16-crash-analysis)
17. [Development Workflow](#17-development-workflow)

---

## 1. Overview

DjavaLauncher 2.0 is a **reverse-engineered SA-MP (San Andreas Multiplayer) Mobile launcher/client** for GTA: San Andreas on Android. It patches the original GTASA game process to inject multiplayer capabilities using native code hooking (Shadowhook), custom UI overlays, and an Android-native launcher UI built with Material Design 3.

### Key Capabilities
- Launch GTASA with injected SA-MP client
- Browse and connect to multiplayer servers
- Download game data assets
- Chat overlay with custom keyboard
- Dialog/menu system for server interactions
- Server favorites management

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Android Application                    │
│                    (:app module)                         │
│                    com.nathan.djavarp                    │
├─────────────────────┬───────────────────────────────────┤
│   Launcher Layer     │         Game Layer                │
│  (Java/Android UI)   │       (Java Bridge)               │
│                      │                                   │
│  SplashScreenActivity│  SAMP.java (extends GTASA)        │
│  MainActivity        │  Game UI Overlays                 │
│  Fragments           │  ChatWindow / DialogManager       │
│  Settings/Server UI  │  CustomKeyboard / LoadingScreen   │
├─────────────────────┼───────────────────────────────────┤
│               Native Layer (C++/CMake)                   │
│                                                          │
│  samp/  ← SA-MP Client Implementation                   │
│  opus/  ← Audio Codec (Opus)                            │
│  Shadowhook ← ByteDance Function Hooking                 │
│  BASS Audio Library                                      │
│  GlossHook / EGL / GLESv3                                │
├─────────────────────┼───────────────────────────────────┤
│               Pre-built JNI Libraries                    │
│                                                          │
│  libGTASA.so      ← Patched game library                 │
│  libSCAnd.so      ← Social Club native                   │
│  libOpenAL*.so    ← OpenAL audio                         │
│  libImmEmulatorJ.so ← Input method (armeabi-v7a only)    │
└─────────────────────┴───────────────────────────────────┘
```

### Activity Flow
```
SplashScreenActivity (portrait, LAUNCHER)
        │
        ▼
  MainActivity (portrait, bottom-nav host)
        │  ┌── HomeFragment
        │  ├── ServerFragment
        │  ├── DownloadFragment
        │  └── SettingsFragment
        │
        ▼
  SAMP (extends GTASA) (landscape, game activity)
        │
        ├── ChatWindow overlay
        ├── CustomKeyboard overlay
        ├── DialogManager (server dialogs)
        ├── TabManager (player list, etc.)
        └── LoadingScreen overlay
```

---

## 3. Build System

### Gradle Setup

| Property | Value |
|----------|-------|
| **Android Gradle Plugin** | 8.13.2 (settings) / 8.2.1 (actual) |
| **Gradle Wrapper** | 8.13 |
| **NDK** | 26.2.11394342 |
| **Kotlin** | No — pure Java + C++ |
| **Build Features** | prefab, viewBinding, buildConfig |

### Build Commands

```bash
./gradlew assembleDebug     # Debug APK (minify=false, release signing)
./gradlew assembleRelease   # Release APK (minify=false, release signing)
```

Both build types use the **same release signing config** (`djavalauncher.jks`, alias `key0`, password `Nathan#1234`).

### Post-Build
Gradle registers a task `copy{Variant}ApksToLaragon` that automatically copies all APK outputs to `C:\laragon\www` after every successful build.

### ABI Splits

```kotlin
splits {
    abi {
        isEnable = true
        reset()
        include("armeabi-v7a", "arm64-v8a")
        isUniversalApk = false
    }
}
```

Only two ABIs are built — no x86/x86_64, no universal APK.

### Gradle Properties

```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.nonTransitiveRClass=true
android.suppressUnsupportedCompileSdk=36
```

### Version Catalog (`gradle/libs.versions.toml`)

Full dependency management via TOML catalog. Key versions:

| Library | Version |
|---------|---------|
| appcompat | 1.6.1 |
| material | 1.12.0 |
| constraintlayout | 2.2.0 |
| navigation | 2.7.7 |
| fragment | 1.6.2 |
| lifecycle | 2.7.0 |
| firebase BOM | 33.6.0 |
| shadowhook | 1.0.9 |
| smoothbottombar | 1.7.9 |

### Repositories
- Google, Maven Central, JitPack, Gradle Plugin Portal
- Aliyun mirrors (public, google, jcenter)
- AppLovin artifacts
- Sonatype snapshots

---

## 4. ABI & Native Libraries

### Pre-built JNI Libraries (`app/src/main/jniLibs/`)

| ABI | Files |
|-----|-------|
| `arm64-v8a/` | `libGTASA.so` (205KB), `libOpenAL64.so`, `libSCAnd.so` |
| `armeabi-v7a/` | `libGTASA.so` (165KB), `libImmEmulatorJ.so`, `libOpenAL32.so`, `libSCAnd.so` |

These are **pre-compiled binaries** — not built by the CMake project. They include the patched GTASA library that enables SA-MP functionality.

### Java Native Libraries (`app/libs/`)
- `com.bda.controller.jar` — NVIDIA Shield controller support

---

## 5. Native (C++) Layer

### CMake Structure

```
app/src/main/cpp/
├── CMakeLists.txt          ← Root: links shadowhook, adds subdirectories
├── opus/                   ← Opus audio codec (45 files, optimized with -O3)
│   └── include/
└── samp/                   ← SA-MP client implementation (26 source files)
    └── CMakeLists.txt      ← Main build config (66 lines)
```

### SA-MP Submodule (`app/src/main/cpp/samp/`)

**Key build details:**
- **Language**: C++20
- **Arch detection**: `armeabi-v7a` → `VER_x32=true`, `arm64-v8a` → `VER_x32=false`
- **Sources**: Recursive `GLOB_RECURSE` over `.c*` files (up to 9 levels deep)
- **Linking**:
  - `log`, `android`, `EGL`, `GLESv3`
  - `opus` (audio codec)
  - `shadowhook::shadowhook` (function hooking)
  - `libbass.so`, `libbass_ssl.so` (BASS audio library)
  - `libGlossHook.a`, `libGlossHook.so`

**Visibility:**
- Debug: `-fvisibility=default`
- Release: `-fvisibility=hidden`

**Linker flag:** `-Wl,-z,max-page-size=16384`

### Dependencies
- **Shadowhook** (ByteDance) — Android PLT hook library, required via `find_package(shadowhook REQUIRED CONFIG)` and prefab
- **BASS** — audio library for streaming and playback
- **BASS_SSL** — SSL support for BASS
- **GlossHook** — additional hooking library
- **Opus** — low-latency audio codec for VoIP

---

## 6. Java/Kotlin Layer

### Entry Points

| Class | Role | Screen Orientation | Exported |
|-------|------|-------------------|----------|
| `SplashScreenActivity` | Launcher activity, app entry point | portrait | Yes |
| `MainActivity` | Bottom-navigation host container | portrait | No |
| `UpdateActivity` | In-app updater screen | portrait | No |
| `SAMP` (extends `GTASA`) | Game activity, native bridge | landscape | Yes |
| `DjavaApplication` | Application subclass, initialization | — | — |

### Activity Details

#### `SplashScreenActivity`
- **Path**: `com.nathan.djavarp.launcher.activity.SplashScreenActivity`
- **Intent filter**: `MAIN`/`LAUNCHER`
- **Theme**: `Theme.DjavaLauncher`
- Transitions to `MainActivity` after initialization

#### `MainActivity`
- **Path**: `com.nathan.djavarp.launcher.activity.MainActivity`
- **Theme**: `Theme.DjavaLauncher`
- Uses `SmoothBottomBar` for navigation
- Hosts 4 fragments: Home, Server, Download, Settings
- Not exported (internal activity)

#### `SAMP`
- **Path**: `com.nathan.djavarp.game.SAMP`
- **Extends**: `com.nathan.djavarp.game.GTASA`
- **Orientation**: landscape, no action bar
- **Launch mode**: `singleTask`
- **Config changes**: keyboard, orientation, screenSize, uiMode
- **Window soft input**: `adjustPan`
- **Exported**: true (launchable from game launcher)
- **Obfuscated**: via Paranoid `@Obfuscate` annotation

#### `DjavaApplication`
- **Path**: `com.nathan.djavarp.launcher.other.DjavaApplication`
- Initializes PRDownloader library
- App-level initialization

---

## 7. Package Reference

### `com.nathan.djavarp.launcher.activity`
| Class | Description |
|-------|-------------|
| `SplashScreenActivity` | Splash screen / launcher entry |
| `MainActivity` | Main bottom-nav host |
| `UpdateActivity` | App update screen |

### `com.nathan.djavarp.launcher.fragment`
| Class | Description |
|-------|-------------|
| `HomeFragment` | Home screen with announcements |
| `ServerFragment` | Server browser/list |
| `DownloadFragment` | Game data download manager |
| `SettingsFragment` | App settings |

### `com.nathan.djavarp.launcher.adapter`
| Class | Description |
|-------|-------------|
| `AnnouncementAdapter` | RecyclerView adapter for announcements |
| `ChangelogAdapter` | RecyclerView adapter for changelog |
| `ServerAdapter` | RecyclerView adapter for server list |

### `com.nathan.djavarp.launcher.data`
| Class | Description |
|-------|-------------|
| `FavoritesInfo` | Favorited server data model |

### `com.nathan.djavarp.launcher.model`
| Class | Description |
|-------|-------------|
| `Announcement` | Announcement data model |
| `Changelog` | Changelog data model |
| `ServerConfig` | Server configuration model |
| `ServerInfo` | Server info model (IP, port, players, etc.) |

### `com.nathan.djavarp.launcher.other`
| Class | Description |
|-------|-------------|
| `CustomEditText` | Custom styled EditText |
| `CustomRecyclerView` | Custom styled RecyclerView |
| `DjavaApplication` | Application init |
| `DownloadAdapter` | Download queue adapter |
| `DownloadModel` | Download item model |
| `Lists` | Static data / list utilities |
| `UnZipCallback` | ZIP extraction callback interface |
| `Util` | General utilities |

### `com.nathan.djavarp.launcher.service`
| Class | Description |
|-------|-------------|
| `DownloadService` | Foreground service for file downloads |

### `com.nathan.djavarp.launcher.util`
| Class | Description |
|-------|-------------|
| `ApiConfig` | API endpoints configuration |
| `AppUpdateManager` | Check and apply app updates |
| `AppVersion` | Version info utility |
| `ClickEffectUtil` | Touch feedback effects |
| `ConfigValidator` | Server settings validation |
| `DownloadStore` | Download state persistence |
| `GPUUtil` | GPU info detection |
| `SampQuery` | SA-MP server query protocol |
| `ServerConnector` | Server connection management |
| `ServerStore` | Server list persistence |
| `SettingsIO` | Settings file I/O (settings.ini) |
| `SignatureChecker` | APK signature verification |
| `StorageUtil` | Storage path utilities |

### `com.nathan.djavarp.game`
| Class | Description |
|-------|-------------|
| `GTASA` | Base game activity class |
| `SAMP` | Main game bridge (obfuscated) |
| `HeightProvider` | Screen height utility for overlays |

### `com.nathan.djavarp.game.ui`
| Class | Description |
|-------|-------------|
| `AttachEdit` | Attached edit text for chat input |
| `ChatWindow` | SA-MP chat overlay |
| `CustomKeyboard` | Custom on-screen keyboard |
| `FadingEdgeLayout` | Layout with fading edge effect |
| `LoadingScreen` | Loading overlay |
| `StrokedTextView` | Text view with stroke |

### `com.nathan.djavarp.game.ui.dialog`
| Class | Description |
|-------|-------------|
| `DialogAdapter` | Server dialog list adapter |
| `DialogManager` | Dialog display manager |

### `com.nathan.djavarp.game.ui.tab`
| Class | Description |
|-------|-------------|
| `PlayerData` | Player info data model |
| `TabAdapter` | Tab list adapter |
| `TabManager` | Tab UI manager |

### `com.nvidia.devtech.*`
NVIDIA Shield TV compatibility layer:
- `NvEventQueueActivity` — Base activity with event queue
- Other NVIDIA-specific helpers

### `com.wardrumstudios.utils.*`
Wardrum Studios utilities:
- Billing, gamepad input, HTTP networking, media playback
- Used by the original GTASA engine

---

## 8. Android Resources

### Layouts (`res/layout/`) — 28 files

| Layout | Purpose |
|--------|---------|
| `activity_main.xml` | Main bottom-nav container |
| `activity_splash.xml` | Splash screen |
| `activity_update.xml` | Update activity |
| `fragment_home.xml` | Home screen |
| `fragment_server.xml` | Server browser |
| `fragment_download.xml` | Download manager |
| `fragment_settings.xml` | Settings screen |
| `samp_launcher.xml` | SAMP game activity |
| `chat_dialog.xml` | Chat window overlay |
| `layout_loading.xml` | Loading screen |
| `dialog_list.xml` | Server dialog list |
| 16+ additional layouts for cards, items, dialogs |

### Drawables (`res/drawable/`) — 68 files

Includes:
- M3 icon vectors: `ic_home_m3.xml`, `ic_server_m3.xml`, `ic_download_m3.xml`, `ic_settings_m3.xml`
- Brand icons: Discord, GitHub, etc.
- Chat/UI backgrounds, dialog styles, button states
- Loading screen graphics, HUD elements, tab backgrounds
- Splash/logo images

### Themes & Styles (`res/values/`)

**`themes.xml`** — Material 3 dark theme:
- `Theme.DjavaLauncher` — DayNight, NoActionBar, Material 3
- Legacy `Theme.SAMPLauncher` — for backward compat
- Bottom nav styling, button/dialog/card styles

**`style.xml`** — Material 3 type scale:
- Display/Label/Title/Body/Headline — all sizes (Large/Medium/Small)
- Uses `@font/djava_material` (PT Root UI)
- Card styles: Elevated, Filled, Outlined
- Chip styles, settings section headers

**`colors.xml`** — 214 color resources:
- Cyber Green `#00E676` (#00E676)
- OLED Black `#000000` (#000000)
- Material 3 baseline palette
- Game-specific colors (chat, HUD, dialog)
- Bottom nav active/inactive colors

**`strings.xml`** — 118 string resources:
- Indonesian (id) locale
- UI labels for home, server, download, settings
- Loading screen messages, about section

**`dimens.xml`** — 42 dimension resources:
- M3 spacing scale (4dp to 64dp)
- Corner sizes (small, medium, large, full)
- Icon sizes (16dp to 120dp)
- Bottom nav dimensions
- Card elevation levels (1–5)

### Fonts (`res/font/`) — 45 files

| Font Family | Styles |
|-------------|--------|
| Montserrat | Regular, Medium, SemiBold, Bold, ExtraBold |
| PT Root UI | Regular, Medium, Bold (`djava_material`) |
| DIN Pro | Regular, Medium, Bold, Black |
| Roboto | Regular, Medium, Bold |
| Akrobat | Regular, SemiBold, Bold, ExtraBold |
| Bebas Neue | Regular |
| Plus Jakarta Sans | Regular, Medium, Bold, ExtraBold |

### Menu (`res/menu/`)
- `bottom_nav_menu.xml` — 4 items: Home, Server, Download, Settings

### Raw Resources (`res/raw/`) — 8 files
- MP3 audio files
- JSON HUD configuration files
- TTF font file

### XML Config (`res/xml/`)
- `network_security_config.xml` — Allows cleartext traffic
- `provider_paths.xml` — FileProvider paths

---

## 9. Dependencies

### AndroidX
| Dependency | Version |
|------------|---------|
| appcompat | 1.6.1 |
| material (MDC) | 1.12.0 |
| constraintlayout | 2.2.0 |
| gridlayout | 1.0.0 |
| recyclerview | 1.3.2 |
| viewpager2 | 1.1.0 |
| fragment | 1.6.2 |
| swiperefreshlayout | 1.1.0 |
| navigation-fragment | 2.7.7 |
| navigation-ui | 2.7.7 |
| lifecycle-runtime | 2.7.0 |
| lifecycle-viewmodel | 2.7.0 |
| lifecycle-process | 2.7.0 |
| multiDexEnabled | true |

### Firebase (BOM 33.6.0)
| Dependency | Version (in BOM) |
|------------|-----------------|
| firebase-analytics | 21.2.0 |
| firebase-crashlytics-ndk | 19.2.0 |
| firebase-messaging | 23.1.1 |

### Third-Party
| Dependency | Version | Purpose |
|------------|---------|---------|
| PRDownloader | 0.6.0 | File download manager |
| Volley | 1.2.1 | HTTP networking |
| SDP (intuit) | 1.1.0 | Screen-size adaptive dimensions |
| ini4j | 0.5.4 | INI file parsing (settings.ini) |
| Glide | 4.13.0 | Image loading |
| Shadowhook | 1.0.9 | Native function hooking (prefab) |
| SmoothBottomBar | 1.7.9 | Bottom navigation bar |
| Material Icon Library | 1.1.5 | Material Design icons |
| Retrofit2 + Gson | 2.1.0 | REST API client |
| AutoImageSlider | 1.4.0 | Image carousel |
| AndroidP7zip | 1.7.2 | 7z extraction |
| un7zip | 1.7.0 | 7z extraction utility |
| Paranoid | 0.3.14 | Obfuscation plugin |

### Local Libraries
| File | Purpose |
|------|---------|
| `libs/com.bda.controller.jar` | NVIDIA Shield controller API |

---

## 10. Permissions & Manifest

### Permissions (22 total)

```xml
<uses-permission android:name="INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.VIBRATE" />
<uses-permission android:name="android.permission.ACCESS_ALL_DOWNLOADS" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.android.vending.CHECK_LICENSE" />
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="com.google.android.finsky.permission.BIND_GET_INSTALL_REFERRER_SERVICE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE" />
```

### Features
```xml
<uses-feature android:glEsVersion="0x00020000" android:required="true"/>
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />
<uses-feature android:name="android.software.leanback" android:required="false" />
<uses-feature android:name="android.hardware.bluetooth" android:required="false" />
<uses-feature android:name="android.hardware.microphone" android:required="false" />
```

OpenGL ES 2.0 is **required**. Touchscreen, leanback, Bluetooth, microphone are optional.

### Application Flags
- `gwpAsanMode="always"` — Guard Page heap detection
- `requestLegacyExternalStorage="true"` — Legacy storage access
- `isGame="true"` — Game category
- `largeHeap="true"` — Large heap for game
- `usesCleartextTraffic="true"` — Allow HTTP
- `hardwareAccelerated="true"` — Hardware acceleration
- `networkSecurityConfig` — Custom network config

### Components
- **4 Activities**: SplashScreenActivity (LAUNCHER), MainActivity, UpdateActivity, SAMP (game, exported)
- **1 Service**: DownloadService (foreground, dataSync type)
- **1 Provider**: FileProvider (APK sharing)

---

## 11. ProGuard & Obfuscation

### ProGuard Rules (`app/proguard-rules.pro`)

```pro
-keep class com.nvidia.devtech.* { *; }
-keep class com.wardrumstudios.utils.* { *; }
-keep class com.nathan.djavarp.game.* { *; }
-dontwarn javax.servlet.**
-dontwarn org.conscrypt.**
-dontwarn org.bouncycastle.**
-dontwarn org.openjsse.**
```

Kept classes:
- NVIDIA Shield helpers
- Wardrum utilities (billing, gamepad, HTTP, media)
- Game bridge package (GTASA, SAMP, etc.)

### Paranoid Obfuscation

The Paranoid Gradle plugin (`com.joom.paranoid:paranoid-gradle-plugin:0.3.14`) is applied at the project level. `SAMP.java` is annotated with `@Obfuscate`, meaning its bytecode is additionally obfuscated at the instruction level.

> **Note**: Stack traces from obfuscated code will NOT match source line numbers.

---

## 12. Assets

Located at `app/src/main/assets/` — 16 entries:

| Path | Description |
|------|-------------|
| `assetfile.txt` | Asset manifest |
| `audio/` | Audio assets directory |
| `data/` | Game data assets |
| `Fonts/` | Font files |
| `images/` | Image assets |
| `json/` | JSON configuration files |
| `scache_small_low.txt` | Sound cache (low quality) |
| `scache_small.txt` | Sound cache |
| `scache.txt` | Sound cache |
| `social club/` | Social Club assets |
| `socialclub/` | Social Club assets (duplicate) |
| `stream.ini` | Streaming configuration |
| `Text/` | Text assets directory |
| `Textures/` | Texture assets directory |
| `version.txt` | Version info |
| `xml/` | XML assets |

---

## 13. Networking & API

### Server Query (`SampQuery`)
- Implements the SA-MP query protocol
- Used to fetch server info (name, players, gamemode, ping)
- Queries servers in `ServerFragment`

### Server Connection (`ServerConnector`)
- Manages connection pipeline to SA-MP servers
- Saves/loads server list via `ServerStore`
- Config file: `settings.ini` (parsed by `ini4j`)

### API Configuration (`ApiConfig`)
- Defines API endpoints for:
  - Server list fetching
  - Announcement retrieval
  - Changelog data
  - App update checking

### Network Stack
- **Volley** — primary HTTP request queue
- **Retrofit2 + Gson** — REST API + JSON parsing
- **PRDownloader** — file downloads with resume support
- **Glide** — image loading/caching

---

## 14. Firebase Services

### Firebase Project
- **Project name**: `djavalauncher`
- **Project ID**: `djavalauncher`
- **Project number**: `344444244793`
- **Package**: `com.nathan.djavarp`
- **API key**: `AIzaSyACnQ13obP1jz39GOZqhDYk0LqwQMjxrD8`
- **google-services.json**: committed to repository

### Enabled Services
- **Firebase Analytics** — Usage tracking
- **Firebase Crashlytics NDK** — Native crash reporting (includes C++ crash detection)
- **Firebase Cloud Messaging** — Push notifications

---

## 15. Game Data & Downloader

### Game Data System
- Game assets (maps, textures, audio) are downloaded separately
- Stored in `gamedata/` directory
- Downloaded from remote server via `DownloadFragment` and `DownloadService`

### Download Service
- **Class**: `com.nathan.djavarp.launcher.service.DownloadService`
- **Type**: Foreground service with `dataSync` type
- Uses PRDownloader under the hood
- Supports download queue management via `DownloadStore` and `DownloadAdapter`

### External Downloader Script
A Python script (`downloader.py`, external to Gradle build) fetches assets from:
```
https://samp-mobile.shop/files.json
```
- Downloads to `gamedata/` directory
- Resumes by file-size matching

---

## 16. Crash Analysis

### Known Issue: RLEDecompress SIGSEGV (Android 14+)

A documented crash occurs on Android 14+ in the `RLEDecompress` function. The crash is a `SIGSEGV` (signal 11, code 1, fault addr `0x10`) at:

```
#00  libGTASA.so (RLEDecompress+0x28)
#01  libGTASA.so (txd_rwFoldFindChunk+0x60)
```

**Root cause**: `RLEDecompress` dereferences a null/invalid pointer when reading the next chunk in GPU texture decompression. The 4th register (`X3`/`R3`) is zero, so `[x3, #0x10]` accesses address `0x10`.

**Factors**: GWP-ASan (`gwpAsanMode="always"`), Android 14+ memory management changes, potential GPU driver differences.

**Workaround**: None yet — under investigation.

### Other Crash Patterns
- `libGTASA.so` in `txd_rwFoldFindChunk` context
- Potential GPU/driver-specific crashes on newer Android versions

---

## 17. Development Workflow

### Setup Requirements
1. Android Studio (latest stable)
2. Android SDK 34+
3. NDK 26.2.11394342
4. JDK 11+
5. CMake 3.12+

### Build Steps

```bash
# Clone with submodules
git clone --recursive https://github.com/NathanKanaeru/DjavaLauncher-2.0.git

# Debug build
./gradlew assembleDebug

# Release build
./gradlew assembleRelease
```

### Signing
- Keystore: `djavalauncher.jks`
- Password: `Nathan#1234`
- Alias: `key0`
- Used for both debug and release builds

### CI/GitHub Actions

The repository includes 6 workflow files under `.github/workflows/`:
- `push.yml` — triggered on push
- `pull.yml` — triggered on PR events
- `issue.yml` — triggered on issue events
- `misc.yml` — stars, forks, wiki edits
- `release.yml` — release events
- `run.yml` — workflow run completion

All are **webhook notifiers** (Discord notifications), not CI build pipelines.

### Git Conventions

**Commit format**: `Scope: Description`
- `UI: Redesign bottom navbar, fix Add Server hint overlap bug`
- `Fix: Resolve RLEDecompress crash on Android 14`
- `Feature: Add server favorite management`
- `Update: Bump dependency versions`

**Author**: `NathanKanaeru` / `akumaumabar222@gmail.com`

### Submodules
- `peak-reference` — external reference repository (empty/incomplete)

---

## Appendix A: File Map

```
DjavaLauncher-2.0/
├── AGENTS.md                      # AI agent reference
├── ANALISIS_KONEKSI.md            # Connection analysis (ID)
├── ANALISIS_UI_ASSETS.md          # UI asset analysis (ID)
├── CRASH_ANALYSIS_RLE.md          # RLEDecompress crash analysis
├── GEMINI.md                      # Gemini project context
├── STRATEGI_MODLOADER.md          # Modloader strategy (ID)
├── README.md                      # Project README
├── CLAUDE.md                      # Claude project context
├── build.gradle.kts               # Root build script
├── settings.gradle.kts            # Project settings
├── gradle.properties              # Gradle properties
├── gradlew / gradlew.bat          # Gradle wrappers
├── google-services.json           # Firebase config
├── djavalauncher.jks              # Release keystore
├── keystore_playmarket2.jks       # Play Store keystore (alt)
├── hooks_fix.cpp                  # Hook fix source
├── downloader.py                  # Game data downloader (external)
│
├── app/
│   ├── build.gradle.kts           # App module build
│   ├── proguard-rules.pro         # ProGuard rules
│   ├── google-services.json       # Firebase config (copy)
│   ├── libs/
│   │   └── com.bda.controller.jar # NVIDIA controller API
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── assets/                # Bundled game assets
│       ├── cpp/                   # Native C++ source
│       │   ├── CMakeLists.txt     # Root CMake
│       │   ├── opus/              # Opus audio codec
│       │   └── samp/              # SA-MP client implementation
│       ├── java/com/nathan/djavarp/
│       │   ├── game/              # Game bridge + UI overlays
│       │   └── launcher/          # Android launcher UI
│       ├── jniLibs/               # Pre-built .so libraries
│       │   ├── arm64-v8a/
│       │   └── armeabi-v7a/
│       └── res/                   # Android resources
│
├── gradle/
│   ├── libs.versions.toml         # Version catalog
│   └── wrapper/                   # Gradle wrapper files
│
├── docs/
│   └── superpowers/
│       ├── plans/                 # Planning documents
│       └── specs/                 # Design specifications
│
└── .github/workflows/             # Discord notification webhooks
```

---

## Appendix B: Key Configuration Values

| Parameter | Value |
|-----------|-------|
| applicationId | `com.nathan.djavarp` |
| minSdk | 26 (Android 8.0) |
| targetSdk | 36 (Android 14/15) |
| compileSdk | 36 |
| versionCode | 135 |
| versionName | "1.5" |
| NDK version | 26.2.11394342 |
| Gradle version | 8.13 |
| AGP version | 8.13.2 (settings) |
| CMake min | 3.12 |
| C++ standard | C++20 |
| OpenGL ES | 2.0+ |
| Keystore password | Nathan#1234 |
| Firebase project | djavalauncher |

---

*Generated from project source — DjavaLauncher 2.0*
