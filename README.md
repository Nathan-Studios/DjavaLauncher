<div align="center">

# ⚡ DjavaLauncher 2.0

### *GTA SA Reversed Android — SA-MP Mobile Client*

<br>

![Version](https://img.shields.io/badge/Version-1.5-00E676?style=for-the-badge&labelColor=1a1a2e)
![SDK](https://img.shields.io/badge/Min%20SDK-26-00E676?style=for-the-badge&labelColor=1a1a2e)
![Target](https://img.shields.io/badge/Target%20SDK-36-00E676?style=for-the-badge&labelColor=1a1a2e)
![NDK](https://img.shields.io/badge/NDK-26.2-00E676?style=for-the-badge&labelColor=1a1a2e)
![C++](https://img.shields.io/badge/C++-20-00E676?style=for-the-badge&labelColor=1a1a2e)

<br>

</div>

---

<div align="center">

## 📋 Table of Contents

[Overview](#-overview) • [Architecture](#-architecture) • [Build System](#-build-system) • [ABI & Native Libraries](#-abi--native-libraries) • [Native Layer](#-native-c-layer) • [Java Layer](#-javakotlin-layer) • [Package Reference](#-package-reference) • [Resources](#-android-resources) • [Dependencies](#-dependencies) • [Permissions](#-permissions--manifest) • [ProGuard](#-proguard--obfuscation) • [Assets](#-assets) • [Networking](#-networking--api) • [Firebase](#-firebase-services) • [Game Data](#-game-data--downloader) • [Crash Analysis](#-crash-analysis) • [Development](#-development-workflow)

</div>

---

<br>

<div align="center">

## 🎯 Overview

**DjavaLauncher 2.0** is a reverse-engineered **SA-MP (San Andreas Multiplayer) Mobile launcher/client** for GTA: San Andreas on Android. It patches the original GTASA game process to inject multiplayer capabilities using native code hooking (Shadowhook), custom UI overlays, and an Android-native launcher UI built with **Material Design 3**.

### 🚀 Key Capabilities

`🎮 Launch GTASA` &nbsp;`🌐 Server Browser` &nbsp;`📥 Data Downloader` &nbsp;`💬 Chat Overlay` &nbsp;`📋 Dialog System` &nbsp;`⭐ Favorites`

</div>

<br>

---

<br>

<div align="center">

## 🏗 Architecture

</div>

```
┌─────────────────────────────────────────────────────────────┐
│                   Android Application                        │
│                    (:app module)                             │
│                    com.nathan.djavarp                        │
├──────────────────────┬──────────────────────────────────────┤
│   Launcher Layer     │         Game Layer                    │
│  (Java/Android UI)   │       (Java Bridge)                   │
│                      │                                       │
│  SplashScreenActivity│  SAMP.java (extends GTASA)            │
│  MainActivity        │  Game UI Overlays                     │
│  Fragments           │  ChatWindow / DialogManager            │
│  Settings/Server UI  │  CustomKeyboard / LoadingScreen        │
├──────────────────────┼──────────────────────────────────────┤
│               Native Layer (C++/CMake)                       │
│                                                              │
│  samp/       ← SA-MP Client Implementation                   │
│  opus/       ← Audio Codec (Opus)                            │
│  Shadowhook  ← ByteDance Function Hooking                     │
│  BASS Audio Library                                          │
│  GlossHook / EGL / GLESv3                                    │
├──────────────────────┼──────────────────────────────────────┤
│               Pre-built JNI Libraries                        │
│                                                              │
│  libGTASA.so      ← Patched game library                     │
│  libSCAnd.so      ← Social Club native                       │
│  libOpenAL*.so    ← OpenAL audio                             │
│  libImmEmulatorJ.so ← Input method (armeabi-v7a only)        │
└──────────────────────┴──────────────────────────────────────┘
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

<br>

---

<br>

<div align="center">

## 🔧 Build System

### Gradle Configuration

| Property | Value |
|----------|-------|
| **Android Gradle Plugin** | 8.13.2 |
| **Gradle Wrapper** | 8.13 |
| **NDK** | 26.2.11394342 |
| **Kotlin** | No — pure Java + C++ |
| **Build Features** | prefab, viewBinding, buildConfig |

### Build Commands

```bash
./gradlew assembleDebug     # Debug APK
./gradlew assembleRelease   # Release APK
```

> Both build types use the same release signing configuration.

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

### Version Catalog Highlights

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

</div>

<br>

---

<br>

<div align="center">

## 📦 ABI & Native Libraries

### Pre-built JNI Libraries

| ABI | Files |
|-----|-------|
| `arm64-v8a/` | `libGTASA.so` (205KB), `libOpenAL64.so`, `libSCAnd.so` |
| `armeabi-v7a/` | `libGTASA.so` (165KB), `libImmEmulatorJ.so`, `libOpenAL32.so`, `libSCAnd.so` |

### Java Native Libraries

- `com.bda.controller.jar` — NVIDIA Shield controller support

</div>

<br>

---

<br>

<div align="center">

## ⚙️ Native (C++) Layer

### CMake Structure

```
app/src/main/cpp/
├── CMakeLists.txt          ← Root: links shadowhook, adds subdirectories
├── opus/                   ← Opus audio codec (45 files, optimized with -O3)
│   └── include/
└── samp/                   ← SA-MP client implementation (26 source files)
    └── CMakeLists.txt      ← Main build config (66 lines)
```

### SA-MP Submodule

| Detail | Value |
|--------|-------|
| **Language** | C++20 |
| **Arch detection** | `armeabi-v7a` → `VER_x32=true`, `arm64-v8a` → `VER_x32=false` |
| **Linking** | `log`, `android`, `EGL`, `GLESv3`, `opus`, `shadowhook`, BASS, GlossHook |

### Native Dependencies

- **Shadowhook** (ByteDance) — Android PLT hook library
- **BASS** — audio library for streaming and playback
- **BASS_SSL** — SSL support for BASS
- **GlossHook** — additional hooking library
- **Opus** — low-latency audio codec for VoIP

</div>

<br>

---

<br>

<div align="center">

## ☕ Java/Kotlin Layer

### Entry Points

| Class | Role | Orientation | Exported |
|-------|------|-------------|----------|
| `SplashScreenActivity` | Launcher activity | portrait | Yes |
| `MainActivity` | Bottom-navigation host | portrait | No |
| `UpdateActivity` | In-app updater | portrait | No |
| `SAMP` (extends `GTASA`) | Game activity, native bridge | landscape | Yes |
| `DjavaApplication` | Application subclass | — | — |

### Activity Details

- **`SplashScreenActivity`** — LAUNCHER activity, transitions to `MainActivity` after initialization
- **`MainActivity`** — Hosts 4 fragments: Home, Server, Download, Settings
- **`SAMP`** — Game activity, `singleTask` launch mode, obfuscated via Paranoid
- **`DjavaApplication`** — Initializes PRDownloader library

</div>

<br>

---

<br>

<div align="center">

## 📁 Package Reference

</div>

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

### `com.nathan.djavarp.launcher.model`
| Class | Description |
|-------|-------------|
| `Announcement` | Announcement data model |
| `Changelog` | Changelog data model |
| `ServerConfig` | Server configuration model |
| `ServerInfo` | Server info model (IP, port, players, etc.) |

### `com.nathan.djavarp.launcher.util`
| Class | Description |
|-------|-------------|
| `ApiConfig` | API endpoints configuration |
| `AppUpdateManager` | Check and apply app updates |
| `SampQuery` | SA-MP server query protocol |
| `ServerConnector` | Server connection management |
| `SettingsIO` | Settings file I/O (settings.ini) |
| `SignatureChecker` | APK signature verification |
| `StorageUtil` | Storage path utilities |

### `com.nathan.djavarp.game`
| Class | Description |
|-------|-------------|
| `GTASA` | Base game activity class |
| `SAMP` | Main game bridge (obfuscated) |

### `com.nathan.djavarp.game.ui`
| Class | Description |
|-------|-------------|
| `ChatWindow` | SA-MP chat overlay |
| `CustomKeyboard` | Custom on-screen keyboard |
| `LoadingScreen` | Loading overlay |
| `DialogManager` | Dialog display manager |
| `TabManager` | Tab UI manager |

### Third-Party Packages
- `com.nvidia.devtech.*` — NVIDIA Shield TV compatibility layer
- `com.wardrumstudios.utils.*` — Wardrum Studios utilities (billing, gamepad, HTTP, media)

<br>

---

<br>

<div align="center">

## 🎨 Android Resources

### Layouts — 28 files

| Layout | Purpose |
|--------|---------|
| `activity_main.xml` | Main bottom-nav container |
| `activity_splash.xml` | Splash screen |
| `fragment_home.xml` | Home screen |
| `fragment_server.xml` | Server browser |
| `fragment_download.xml` | Download manager |
| `fragment_settings.xml` | Settings screen |
| `samp_launcher.xml` | SAMP game activity |
| `chat_dialog.xml` | Chat window overlay |

### Themes & Styles

- **Material 3 Dark Theme** — `Theme.DjavaLauncher` (DayNight, NoActionBar)
- **214 color resources** — Cyber Green `#00E676`, OLED Black `#000000`, M3 palette
- **118 string resources** — Indonesian (id) locale
- **42 dimension resources** — M3 spacing scale (4dp to 64dp)

### Fonts — 45 files

| Family | Styles |
|--------|--------|
| Montserrat | Regular, Medium, SemiBold, Bold, ExtraBold |
| PT Root UI | Regular, Medium, Bold |
| DIN Pro | Regular, Medium, Bold, Black |
| Roboto | Regular, Medium, Bold |
| Akrobat | Regular, SemiBold, Bold, ExtraBold |
| Bebas Neue | Regular |
| Plus Jakarta Sans | Regular, Medium, Bold, ExtraBold |

</div>

<br>

---

<br>

<div align="center">

## 📚 Dependencies

### AndroidX

| Dependency | Version |
|------------|---------|
| appcompat | 1.6.1 |
| material (MDC) | 1.12.0 |
| constraintlayout | 2.2.0 |
| navigation | 2.7.7 |
| fragment | 1.6.2 |
| lifecycle | 2.7.0 |

### Firebase (BOM 33.6.0)

| Dependency | Version |
|------------|---------|
| firebase-analytics | 21.2.0 |
| firebase-crashlytics-ndk | 19.2.0 |
| firebase-messaging | 23.1.1 |

### Third-Party

| Dependency | Version | Purpose |
|------------|---------|---------|
| PRDownloader | 0.6.0 | File download manager |
| Volley | 1.2.1 | HTTP networking |
| Shadowhook | 1.0.9 | Native function hooking |
| SmoothBottomBar | 1.7.9 | Bottom navigation bar |
| Retrofit2 + Gson | 2.1.0 | REST API client |
| Glide | 4.13.0 | Image loading |
| Paranoid | 0.3.14 | Obfuscation plugin |

</div>

<br>

---

<br>

<div align="center">

## 🔐 Permissions & Manifest

### Permissions (22 total)

```
INTERNET  •  ACCESS_NETWORK_STATE  •  WAKE_LOCK  •  VIBRATE
FOREGROUND_SERVICE  •  POST_NOTIFICATIONS  •  RECORD_AUDIO
READ_EXTERNAL_STORAGE  •  WRITE_EXTERNAL_STORAGE
MANAGE_EXTERNAL_STORAGE  •  REQUEST_INSTALL_PACKAGES
BLUETOOTH  •  BLUETOOTH_CONNECT  •  ACCESS_WIFI_STATE
RECEIVE_BOOT_COMPLETED  •  ACCESS_ALL_DOWNLOADS
INSTALL_PACKAGES  •  CHECK_LICENSE  •  BIND_GET_INSTALL_REFERRER_SERVICE
FOREGROUND_SERVICE_DATA_SYNC  •  C2D_MESSAGE  •  RECEIVE
```

### Features

```xml
<uses-feature android:glEsVersion="0x00020000" android:required="true"/>
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />
<uses-feature android:name="android.software.leanback" android:required="false" />
<uses-feature android:name="android.hardware.bluetooth" android:required="false" />
<uses-feature android:name="android.hardware.microphone" android:required="false" />
```

### App Flags

`gwpAsanMode="always"` • `requestLegacyExternalStorage="true"` • `isGame="true"` • `largeHeap="true"` • `usesCleartextTraffic="true"` • `hardwareAccelerated="true"`

</div>

<br>

---

<br>

<div align="center">

## 🛡 ProGuard & Obfuscation

### ProGuard Rules

```pro
-keep class com.nvidia.devtech.* { *; }
-keep class com.wardrumstudios.utils.* { *; }
-keep class com.nathan.djavarp.game.* { *; }
```

### Paranoid Obfuscation

The Paranoid Gradle plugin (`0.3.14`) applies instruction-level bytecode obfuscation to `SAMP.java` via the `@Obfuscate` annotation.

> **Note**: Stack traces from obfuscated code will NOT match source line numbers.

</div>

<br>

---

<br>

<div align="center">

## 📂 Assets

| Path | Description |
|------|-------------|
| `audio/` | Audio assets directory |
| `data/` | Game data assets |
| `Fonts/` | Font files |
| `images/` | Image assets |
| `json/` | JSON configuration files |
| `social club/` | Social Club assets |
| `Text/` | Text assets directory |
| `Textures/` | Texture assets directory |

</div>

<br>

---

<br>

<div align="center">

## 🌐 Networking & API

### Components

| Component | Function |
|-----------|----------|
| **SampQuery** | SA-MP query protocol implementation |
| **ServerConnector** | Server connection pipeline management |
| **ApiConfig** | API endpoints for server list, announcements, changelog, updates |

### Network Stack

`Volley` → HTTP requests • `Retrofit2 + Gson` → REST API • `PRDownloader` → File downloads • `Glide` → Image loading

</div>

<br>

---

<br>

<div align="center">

## 🔥 Firebase Services

| Service | Purpose |
|---------|---------|
| **Firebase Analytics** | Usage tracking |
| **Firebase Crashlytics NDK** | Native crash reporting (C++ included) |
| **Firebase Cloud Messaging** | Push notifications |

</div>

<br>

---

<br>

<div align="center">

## 📥 Game Data & Downloader

### Game Data System

Game assets (maps, textures, audio) are downloaded separately and stored in the `gamedata/` directory via `DownloadFragment` and `DownloadService`.

### Download Service

- **Class**: `com.nathan.djavarp.launcher.service.DownloadService`
- **Type**: Foreground service with `dataSync` type
- Uses PRDownloader with queue management

</div>

<br>

---

<br>

<div align="center">

## 💥 Crash Analysis

### Known Issue: RLEDecompress SIGSEGV (Android 14+)

A documented crash occurs on Android 14+ in the `RLEDecompress` function — `SIGSEGV` (signal 11, code 1, fault addr `0x10`):

```
#00  libGTASA.so (RLEDecompress+0x28)
#01  libGTASA.so (txd_rwFoldFindChunk+0x60)
```

**Root cause**: `RLEDecompress` dereferences a null/invalid pointer during GPU texture decompression.

**Status**: Under investigation.

</div>

<br>

---

<br>

<div align="center">

## 🛠 Development Workflow

### Requirements

```
Android Studio (latest)  •  Android SDK 34+  •  NDK 26.2.11394342  •  JDK 11+  •  CMake 3.12+
```

### Build Commands

```bash
./gradlew assembleDebug   # Debug APK
./gradlew assembleRelease # Release APK
```

### Git Conventions

**Format**: `Scope: Description`

```
UI: Redesign bottom navbar, fix Add Server hint overlap bug
Fix: Resolve RLEDecompress crash on Android 14
Feature: Add server favorite management
Update: Bump dependency versions
```

</div>

<br>

---

<br>

<div align="center">

## 📁 Project Structure

```
DjavaLauncher-2.0/
├── app/
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   ├── google-services.json
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── assets/
│       ├── cpp/                   # Native C++ source
│       │   ├── CMakeLists.txt
│       │   ├── opus/              # Opus audio codec
│       │   └── samp/              # SA-MP client implementation
│       ├── java/com/nathan/djavarp/
│       │   ├── game/              # Game bridge + UI overlays
│       │   └── launcher/          # Android launcher UI
│       ├── jniLibs/               # Pre-built .so libraries
│       └── res/                   # Android resources
├── gradle/
│   ├── libs.versions.toml
│   └── wrapper/
├── docs/
│   └── superpowers/
│       ├── plans/
│       └── specs/
└── *.md                          # Project documentation
```

</div>

<br>

---

<br>

<div align="center">

## ⚙️ Key Configuration

| Parameter | Value |
|-----------|-------|
| applicationId | `com.nathan.djavarp` |
| minSdk | 26 (Android 8.0) |
| targetSdk | 36 (Android 14/15) |
| compileSdk | 36 |
| versionCode | 135 |
| versionName | "1.5" |
| NDK | 26.2.11394342 |
| Gradle | 8.13 |
| C++ Standard | C++20 |
| OpenGL ES | 2.0+ |

</div>

---

<div align="center">

<br>

**DjavaLauncher 2.0** — *Confidential & Proprietary*

<br>

</div>
