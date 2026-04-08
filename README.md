# PerfOptimizer — Lightweight Android Performance Monitor

A minimal, battery-friendly system monitor for Android 8–11 devices.  
Optimised for low-end phones (Nokia 2.3, Redmi 9A, etc.) with 2 GB RAM.

---

## Features

| Screen | What it does |
|---|---|
| **Dashboard** | Live RAM, CPU, and Storage gauges — auto-refreshes every 4 s |
| **Running Apps** | Lists background processes sorted by memory; lets you stop them |
| **Storage** | Internal storage breakdown with progress bar |
| **Cache Cleaner** | Clears this app's own cache; deep-links to system storage settings |

---

## Project Structure

```
AndroidPerfOptimizer/
├── app/
│   ├── src/main/
│   │   ├── AndroidManifest.xml
│   │   ├── java/com/perfoptimizer/
│   │   │   ├── MainActivity.kt               ← Single-activity host
│   │   │   ├── data/
│   │   │   │   ├── SystemInfoRepository.kt   ← RAM / CPU / Storage reads
│   │   │   │   └── AppProcessRepository.kt   ← Process listing & kill
│   │   │   ├── utils/
│   │   │   │   └── FormatUtils.kt            ← Byte → human string
│   │   │   ├── viewmodel/
│   │   │   │   ├── MainViewModel.kt          ← Polling loop for metrics
│   │   │   │   └── AppsViewModel.kt          ← Running-apps ViewModel
│   │   │   └── ui/
│   │   │       ├── dashboard/DashboardFragment.kt
│   │   │       ├── apps/RunningAppsFragment.kt
│   │   │       ├── apps/AppProcessAdapter.kt
│   │   │       ├── storage/StorageFragment.kt
│   │   │       └── cache/CacheFragment.kt
│   │   └── res/
│   │       ├── layout/          ← XML layouts
│   │       ├── navigation/      ← nav_graph.xml
│   │       ├── menu/            ← bottom_nav_menu.xml
│   │       ├── values/          ← colors, strings, themes
│   │       ├── drawable/        ← vector icons
│   │       └── color/           ← nav icon selector
│   ├── build.gradle
│   └── proguard-rules.pro
├── build.gradle
├── settings.gradle
└── gradle.properties
```

---

## Requirements

- **Android Studio** Arctic Fox (2020.3.1) or newer  
  Download: https://developer.android.com/studio
- **JDK 11** (bundled with Android Studio)
- **Android SDK** with:
  - Build Tools 30.0.3
  - Platform SDK 30 (Android 11)
  - Minimum platform SDK 26 (Android 8)

---

## Building in Android Studio

### Step 1 — Open the project

1. Launch Android Studio.
2. Click **File → Open**.
3. Navigate to the `AndroidPerfOptimizer/` folder and click **OK**.
4. Wait for Gradle sync to finish (first sync downloads ~200 MB of dependencies).

### Step 2 — Sync Gradle

If Android Studio shows a "Gradle sync failed" banner:
- Click **File → Sync Project with Gradle Files**.
- If it asks to install missing SDK components, click **Install** and accept licences.

### Step 3 — Build a debug APK

```
Build → Build Bundle(s) / APK(s) → Build APK(s)
```

The APK will appear at:
```
app/build/outputs/apk/debug/app-debug.apk
```

### Step 4 — Build a release APK (smaller, optimised)

1. **Build → Generate Signed Bundle / APK → APK → Next**
2. Create or choose a keystore (keep the keystore file safe — you need it for updates).
3. Select **release** build variant → **Finish**.

Output: `app/build/outputs/apk/release/app-release.apk`

Release APK will be ~1.5–2 MB after minification and resource shrinking.

---

## Running on a Device / Emulator

### Option A — Direct from Android Studio

1. Enable **Developer Options** on your phone:  
   *Settings → About Phone → tap "Build number" 7 times*
2. Enable **USB Debugging** in Developer Options.
3. Connect phone via USB and accept the RSA key prompt.
4. In Android Studio select your device from the device dropdown.
5. Click the ▶ **Run** button.

### Option B — Install APK manually (sideload)

#### On the phone

1. Go to **Settings → Apps** (or Security) → enable **Install unknown apps** for your file manager.

#### Transfer the APK

- Via USB: copy `app-debug.apk` to phone storage.
- Via ADB:
  ```bash
  adb install app/build/outputs/apk/debug/app-debug.apk
  ```

#### Install

Open the APK file from your file manager and tap **Install**.

---

## Android 11 (API 30) — Special Notes

### Package Visibility

Android 11 requires apps to declare which packages they query.  
This is handled in `AndroidManifest.xml` with the `<queries>` block — no extra steps needed.

### KILL_BACKGROUND_PROCESSES limitation

On Android 8+, `killBackgroundProcesses()` only affects apps that are truly in the background (cached state). Apps running a foreground service or actively visible cannot be killed by a third-party app — this is an OS-level restriction and is by design for stability.

### PACKAGE_USAGE_STATS (optional)

For richer process monitoring, this permission requires manual user grant:

1. Go to **Settings → Apps → Special app access → Usage access**.
2. Find **PerfOptimizer** and toggle it on.

The app works fine without it — this permission only unlocks additional CPU usage detail on some OEM ROMs.

---

## Granting Permissions via ADB (faster for testing)

```bash
# Core kill permission — granted automatically on install
adb shell pm grant com.perfoptimizer android.permission.KILL_BACKGROUND_PROCESSES

# Optional usage stats (must be granted manually on device OR via ADB)
adb shell appops set com.perfoptimizer PACKAGE_USAGE_STATS allow
```

---

## Performance Design Decisions

| Decision | Reason |
|---|---|
| Single Activity + Navigation Component | Avoids Activity stack overhead; fragments share one ViewModel |
| Coroutine polling (no Service) | No persistent background thread; polling stops when app is backgrounded |
| 4-second refresh interval | Balances freshness vs CPU/battery on low-end SoCs |
| Dark theme (#121212 background) | Reduces battery draw on OLED screens common in budget phones |
| ListAdapter + DiffUtil | Only re-binds changed RecyclerView rows; no full redraws |
| No third-party image/chart libraries | Keeps APK under 3 MB; uses ProgressBar instead |
| `android:windowAnimationStyle="@null"` | Removes transition animations — saves 1–2 frames of GPU work |
| ViewBinding (no Kotlin synthetics) | Null-safe, no reflection, compile-time checked |

---

## Troubleshooting

**Gradle sync fails with "SDK location not found"**  
→ Create `local.properties` in the project root:
```
sdk.dir=/path/to/your/Android/Sdk
```
On macOS: `sdk.dir=/Users/YOUR_NAME/Library/Android/sdk`  
On Windows: `sdk.dir=C\:\\Users\\YOUR_NAME\\AppData\\Local\\Android\\Sdk`

**"No such module 'app'" error**  
→ Delete `.gradle/` and `build/` folders, then re-sync.

**App shows 0% CPU on some devices**  
→ Some OEM kernels (Qualcomm, MediaTek) restrict `/proc/stat` reads for non-root apps. CPU monitoring will silently return 0 on those devices — all other features remain fully functional.

**Running Apps list is empty**  
→ On Android 11, `getRunningAppProcesses()` only returns processes for apps that share the same UID, unless the caller holds `PACKAGE_USAGE_STATS`. Grant the permission as described above for the full list.

---

## Minimum Device Requirements

| Spec | Minimum |
|---|---|
| Android version | 8.0 Oreo (API 26) |
| RAM | 1 GB (tested on 2 GB) |
| Storage free | 5 MB for APK install |
| CPU | Any ARMv7 or ARM64 |

Tested on: Nokia 2.3 (Android 11, 2 GB RAM), Redmi 9A (Android 10, 2 GB RAM),  
Pixel 4a emulator (Android 11, AVD default).

---

## License

MIT — free to use, modify, and distribute.
