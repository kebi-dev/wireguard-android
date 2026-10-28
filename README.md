# Android GUI for [WireGuard](https://www.wireguard.com/)

**[Download from the Play Store](https://play.google.com/store/apps/details?id=com.wireguard.android)**

This is an Android GUI for [WireGuard](https://www.wireguard.com/). It [opportunistically uses the kernel implementation](https://git.zx2c4.com/android_kernel_wireguard/about/), and falls back to using the non-root [userspace implementation](https://git.zx2c4.com/wireguard-go/about/).

## Building and installing on your phone

The quickest path is a debug build installed over USB. Debug builds use the package name `com.wireguard.android.debug`, so they install alongside the Play Store version instead of replacing it.

### 1. Check your computer

The native part of the app (wireguard-go) is built by a Makefile that downloads a pinned Go toolchain, so the build only works on:

- **Linux x86_64**
- **macOS** (Intel or Apple Silicon)
- **Windows via [WSL 2](https://learn.microsoft.com/windows/wsl/install)** (run everything, including the SDK, inside the Linux side). Native Windows is not supported: there is no `gradlew.bat`, and the Go download only has checksums for the platforms above.

Linux ARM64 hosts are not supported for the same reason.

### 2. Install the tools

| Tool | Version | Notes |
| --- | --- | --- |
| JDK | 17 or newer | Required by Gradle 9.3.1 and Android Gradle Plugin 9.1.0. Android Studio's bundled JDK works. |
| Android SDK Platform | API 36 | `compileSdk = 36` in `ui/build.gradle.kts` and `tunnel/build.gradle.kts`. |
| Android NDK (Side by side) | Default for AGP 9.1.0 | No `ndkVersion` is pinned, so the build uses the plugin's default. If Gradle reports a missing NDK, install the exact version it names. |
| CMake | From the SDK Manager | Builds `libwg.so` and `libwg-quick.so` from `tunnel/tools/CMakeLists.txt`. |
| Go | Nothing to install | The build downloads Go 1.24.3 into `~/.gradle/caches/golang/` on first run. |
| Host tools | `git`, `curl`, `tar`, `patch`, `sha256sum`, `flock`, a C compiler (`cc`) | Usually already present on Linux. On Debian/Ubuntu: `sudo apt install build-essential curl git`. On macOS, install the Xcode command line tools (`xcode-select --install`) and [flock(1)](https://github.com/discoteq/flock) (`brew install flock`). |

Gradle itself is downloaded by `./gradlew`, so you don't need to install it.

### 3. Get the source

Clone with submodules. The native tools (`wireguard-tools`, `elf-cleaner`) live in submodules and the build fails without them.

```
$ git clone --recurse-submodules https://github.com/kebi-dev/wireguard-android
$ cd wireguard-android
```

If you already cloned without `--recurse-submodules`, run:

```
$ git submodule update --init --recursive
```

### 4. Prepare your phone

The app runs on Android 7.0 (API 24) or newer.

1. Open **Settings → About phone** and tap **Build number** seven times to enable Developer options. (On some phones it is under **Software information**.)
2. Open **Settings → System → Developer options** and turn on **USB debugging**.
3. Connect the phone with a USB cable that supports data, and accept the **Allow USB debugging?** prompt on the phone. Tick **Always allow from this computer**.

Wireless debugging (Android 11+) also works: pair the phone in Android Studio's **Device Manager** or with `adb pair`.

### Option A: Android Studio

1. Install a recent [Android Studio](https://developer.android.com/studio). It must support Android Gradle Plugin 9.1.0; if it is too old, the Gradle sync tells you so.
2. **File → Open** and choose the cloned `wireguard-android` folder. Wait for the Gradle sync to finish.
3. Open **Tools → SDK Manager**:
   - **SDK Platforms**: tick **Android 16 (API 36)**.
   - **SDK Tools**: tick **NDK (Side by side)**, **CMake**, and **Android SDK Platform-Tools**.
   - Click **Apply**.
4. Pick the **ui** run configuration and your phone in the device dropdown at the top.
5. Click **Run ▶**. The first build takes several minutes because it downloads Go and compiles wireguard-go for every CPU architecture.

### Option B: Command line

1. Install the Android command line tools, or reuse the SDK that Android Studio installed. Point `ANDROID_HOME` at it, for example in `~/.bashrc` or `~/.zshrc`:

   ```
   # Linux / WSL
   export ANDROID_HOME="$HOME/Android/Sdk"
   # macOS
   export ANDROID_HOME="$HOME/Library/Android/sdk"

   export PATH="$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin"
   ```

   Alternatively, create `local.properties` in the repo root with `sdk.dir=/path/to/sdk`. It is ignored by git.

2. Install the SDK packages and accept the licenses:

   ```
   $ sdkmanager "platforms;android-36" "platform-tools" "cmake;3.22.1"
   $ sdkmanager --licenses
   ```

   If the build asks for a different CMake version, install that one instead. The build installs the default NDK on first run once licenses are accepted. If it doesn't, list versions with `sdkmanager --list | grep ndk` and install the one named in the error with `sdkmanager "ndk;<version>"`.

3. Make sure the phone is visible:

   ```
   $ adb devices
   ```

   It should show your device as `device`, not `unauthorized`.

4. Build and install:

   ```
   $ ./gradlew installDebug
   ```

   Or build the APK without installing and copy it over yourself:

   ```
   $ ./gradlew assembleDebug
   $ adb install -r ui/build/outputs/apk/debug/ui-debug.apk
   ```

The app appears on the phone as **WireGuard**.

### Release builds

`./gradlew assembleRelease` produces an **unsigned** APK, which Android refuses to install. To use a release build, sign it with your own key (for example with `apksigner` from the SDK build-tools, or via **Build → Generate Signed App Bundle / APK** in Android Studio). A release build uses the package name `com.wireguard.android`, so it cannot be installed over the Play Store version because the signatures differ; uninstall that first.

### Common problems

- **CMake errors about missing files under `wireguard-tools` or `elf-cleaner`**: submodules are missing. Run `git submodule update --init --recursive`.
- **`SDK location not found`**: set `ANDROID_HOME` or create `local.properties` with `sdk.dir=...`.
- **`NDK not configured` / `CMake not found`**: install them from the SDK Manager (Option A, step 3) or with `sdkmanager`.
- **`Unsupported class file major version` or "requires Java 17"**: `java -version` must report 17 or newer. In Android Studio, check **Settings → Build, Execution, Deployment → Build Tools → Gradle → Gradle JDK**.
- **Checksum errors from `sha256sum` or `flock: command not found`** while building wireguard-go: you're on an unsupported host (see step 1) or missing a host tool (see the host tools row above). On macOS, install `flock`.
- **Go download fails behind a proxy or firewall**: the build fetches `https://dl.google.com/go/go1.24.3.<platform>.tar.gz`. Download it by hand into `~/.gradle/caches/golang/` if needed.
- **`adb devices` shows `unauthorized`**: unlock the phone and accept the USB debugging prompt. If it doesn't appear, revoke USB debugging authorizations in Developer options and reconnect.
- **`INSTALL_FAILED_UPDATE_INCOMPATIBLE`**: an older build with the same package name but a different signing key is installed. Uninstall it with `adb uninstall com.wireguard.android.debug` and try again.
- **Gradle runs out of memory**: raise `org.gradle.jvmargs` in `gradle.properties` (for example to `-Xmx4g`).
- **Stale native build after switching branches**: run `./gradlew clean` and delete `tunnel/.cxx/`.

## Embedding

The tunnel library is [on Maven Central](https://search.maven.org/artifact/com.wireguard.android/tunnel), alongside [extensive class library documentation](https://javadoc.io/doc/com.wireguard.android/tunnel).

```
implementation 'com.wireguard.android:tunnel:$wireguardTunnelVersion'
```

The library makes use of Java 8 features, so be sure to support those in your gradle configuration with [desugaring](https://developer.android.com/studio/write/java8-support#library-desugaring):

```
compileOptions {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
    coreLibraryDesugaringEnabled = true
}
dependencies {
    coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:2.0.3"
}
```

## Translating

Please help us translate the app into several languages on [our translation platform](https://crowdin.com/project/WireGuard).
