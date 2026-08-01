# Tauri Android Workflow Reference

## Prerequisite Checks

Use these checks before deeper debugging:

```bash
npm run tauri -- --version
rustup target list --installed | rg 'android|linux-android'
adb devices
java -version
```

Expected Rust Android targets usually include:

- `aarch64-linux-android`
- `armv7-linux-androideabi`
- `i686-linux-android`
- `x86_64-linux-android`

Install missing targets with `rustup target add <target>`.

## Gradle Proxy

Symptom: Gradle cannot download Android/Kotlin dependencies, hangs, or fails with network errors.

File: `src-tauri/gen/android/gradle.properties`

Example for a local HTTP(S) proxy on port `7890`:

```properties
systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=7890
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=7890
```

Verify from the Android Gradle project:

```bash
cd src-tauri/gen/android
./gradlew tasks --info
```

Only add proxy config when the user actually uses a proxy. If the repo will be shared, explain that this is machine-local configuration.

## NDK Linker Configuration

Symptom: Rust cross-compilation fails because Android linkers cannot be found.

Preferred approach: set `ANDROID_NDK_HOME` to the installed NDK directory.

Example:

```bash
export ANDROID_NDK_HOME="$ANDROID_HOME/ndk/26.3.11579264"
```

If Cargo still cannot find linkers, create `src-tauri/.cargo/config.toml`:

```toml
[target.aarch64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"

[target.armv7-linux-androideabi]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi24-clang"

[target.i686-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/i686-linux-android24-clang"

[target.x86_64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android24-clang"
```

If environment variables are not expanded by the local Cargo version or shell context, replace `$ANDROID_NDK_HOME` with the absolute NDK path after confirming it with the user.

## Vite And Tauri Dev Server

Symptom: desktop dev works, but Android app cannot connect, hot reload fails, or the phone shows a blank page.

Important settings:

`package.json`

```json
{
  "scripts": {
    "dev": "vite --host"
  }
}
```

`vite.config.*`

```ts
const host = process.env.TAURI_DEV_HOST;

export default defineConfig({
  server: {
    port: 1420,
    strictPort: true,
    host: host || false,
    hmr: host
      ? {
          protocol: "ws",
          host,
          port: 1421,
        }
      : undefined,
  },
});
```

`src-tauri/tauri.conf.json`

```json
{
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:1420",
    "beforeBuildCommand": "npm run build"
  }
}
```

Run with a connected and authorized phone:

```bash
npm run tauri android dev -- --verbose
```

## Phone White Screen

Use Chrome remote WebView debugging:

1. Start the Android app on the phone.
2. Open Chrome on the computer.
3. Visit `chrome://inspect/#devices`.
4. Inspect the app WebView and read console/network errors.

Typical causes:

- Vite dev server bound only to localhost instead of device-reachable host.
- `TAURI_DEV_HOST` or HMR host mismatch.
- Frontend runtime exception hidden inside WebView.
- CSP or cleartext traffic mismatch for debug mode.

## Release APK Signing

Only configure release signing after debug Android builds are working.

### 1. Generate a release keystore

Run from `src-tauri/gen/android/app` (so the keystore lands in the same directory as `build.gradle.kts`, where `file(...)` resolves relative paths):

```bash
cd src-tauri/gen/android/app
keytool -genkey -v -keystore my-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-app-alias -storepass <password> -keypass <password> -dname "CN=devtoys, OU=dev, O=dev, L=dev, ST=dev, C=CN"
```

### 2. Create `src-tauri/gen/android/keystore.properties`

```properties
keyAlias=my-app-alias
password=<your-keystore-password>
storeFile=my-release-key.jks
```

`storeFile` is relative to `app/build.gradle.kts` when using `file(...)`. If you prefer to store the keystore at the root project level, change `storeFile` to a path like `app/my-release-key.jks` and use `rootProject.file(...)` in the Gradle snippet instead.

### 3. Update `src-tauri/gen/android/app/build.gradle.kts`

**Add imports** (check existing imports first — `java.util.Properties` may already be present):

```kotlin
import java.io.FileInputStream
import java.util.Properties
```

**CRITICAL: `signingConfigs` MUST be defined BEFORE `buildTypes`** inside the `android { ... }` block. If `buildTypes` references `signingConfigs.getByName("release")` before the `signingConfigs` block is parsed, Gradle will fail with `SigningConfig with name 'release' not found`.

Add `signingConfigs` BEFORE `buildTypes`:

```kotlin
android {
    // ... compileSdk, namespace, defaultConfig ...

    signingConfigs {
        create("release") {
            val keystorePropertiesFile = rootProject.file("keystore.properties")
            val keystoreProperties = Properties()
            if (keystorePropertiesFile.exists()) {
                keystoreProperties.load(FileInputStream(keystorePropertiesFile))
            }

            keyAlias = keystoreProperties["keyAlias"] as String
            keyPassword = keystoreProperties["password"] as String
            storeFile = file(keystoreProperties["storeFile"] as String)
            storePassword = keystoreProperties["password"] as String
        }
    }

    buildTypes {
        // ...
        getByName("release") {
            signingConfig = signingConfigs.getByName("release")
            // ... other release config ...
        }
    }
}
```

### 4. Update `.gitignore`

Add these entries to prevent committing signing secrets:

```gitignore
# Android signing
*.jks
keystore.properties
```

### 5. Build

Use the Android-specific build command (NOT `npm run tauri build`, which targets desktop):

```bash
npm run tauri android build
```

Output paths:

```text
APK: src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk
AAB: src-tauri/gen/android/app/build/outputs/bundle/universalRelease/app-universal-release.aab
```

### Manual fallback signing (stopgap only)

If automatic signing is not configured yet, a temporary manual signing flow is:

```bash
cd src-tauri/gen/android/app/build/outputs/apk/universal/release/
apksigner sign --ks ~/.android/debug.keystore --ks-pass pass:android --key-pass pass:android --out app-release-signed.apk app-universal-release-unsigned.apk
```

Use manual debug-keystore signing only as a stopgap for local testing, not for real release distribution.

## Quick Diagnostic Map

Gradle cannot resolve dependencies:

- Check `gradle.properties`.
- Verify proxy and run `./gradlew tasks --info`.

Rust target/linker error:

- Check installed Rust Android targets.
- Check `ANDROID_NDK_HOME`.
- Add or fix `src-tauri/.cargo/config.toml`.

No device:

- Check `adb devices`.
- Reconnect USB and accept phone authorization.

White screen:

- Inspect WebView with `chrome://inspect/#devices`.
- Check Vite host/HMR and frontend console errors.

Release APK unsigned / signing error:

- `SigningConfig with name 'release' not found`: `signingConfigs` block is defined AFTER `buildTypes`. Move `signingConfigs` before `buildTypes` inside `android { }`.
- `keystore.properties` not found: ensure the file exists at `src-tauri/gen/android/keystore.properties` and `rootProject.file(...)` can resolve it.
- Keystore file not found: check that `storeFile` is relative to the module directory (`app/`) when using `file(...)`, or use an explicit path.
- Build command wrong: use `npm run tauri android build` for Android, not `npm run tauri build` (which builds desktop bundles).

Release APK unsigned (old):

- Check `keystore.properties`.
- Check `signingConfigs` in `app/build.gradle.kts`.
- Build again with `npm run tauri android build`.
