---
name: tauri-android-init
description: Guide, initialize, configure, debug, and package Tauri v2 Android mobile app development, including fresh projects created with `npm create tauri-app@latest`. Use this skill whenever the user mentions Tauri Android, Tauri mobile, Android dev setup, `tauri android init`, `tauri android dev`, Gradle/NDK linker errors, phone white screen/WebView debugging, APK release signing, or asks to configure a Tauri app for Android, even if they only describe a build or device problem.
---

# Tauri Android Init

Use this skill to turn a desktop/web Tauri project into a working Android development setup, and to debug common Tauri Android traps around Gradle, NDK, WebView dev mode, and APK signing.

## First Read

This skill is self-contained for distribution. Do not assume the user's project has the original notes this skill was distilled from.

Read the bundled reference when the request involves setup details, errors, signing, or device debugging:

- `references/tauri-android-workflow.md`: setup checklist, configuration snippets, and diagnostic flow.

## Working Rules

1. Inspect the existing project before editing: `package.json`, `vite.config.*`, `src-tauri/tauri.conf.json`, `src-tauri/gen/android/gradle.properties`, `src-tauri/gen/android/app/build.gradle.kts`, `src-tauri/gen/android/keystore.properties` (if signing exists), and `src-tauri/.cargo/config.toml` if present.
2. Treat generated Android files as project-local state. Do not blindly regenerate or overwrite `src-tauri/gen/android` if the user already has signing, namespace, SDK, or Gradle fixes there.
3. Keep machine-specific paths out of reusable config when possible. Prefer environment variables such as `ANDROID_HOME`, `ANDROID_SDK_ROOT`, and `ANDROID_NDK_HOME`; only hard-code NDK paths when the user asks or the environment cannot resolve them.
4. Do not expose or commit signing secrets. If a `keystore.properties` file contains a real password, avoid printing it back. Recommend `.gitignore` coverage for `*.jks` and `keystore.properties`.
5. When the user has a concrete error, route through the diagnostic flow before making broad changes. Most failures fall into Gradle networking, NDK linker selection, device/WebView connectivity, Vite host/HMR config, or release signing.

## Default Workflow

Follow this order for Android initialization:

1. Confirm prerequisites: Android SDK, NDK, Java/JDK, Rust Android targets, Tauri CLI, connected phone/emulator, and USB debugging.
   - Check ALL of these env vars (users may use different names): `ANDROID_HOME`, `ANDROID_SDK_ROOT`, `NDK_HOME`, `ANDROID_NDK_HOME`, `JAVA_HOME`.
   - **Tauri specifically reads `ANDROID_NDK_HOME`** — if the user only has `NDK_HOME`, the NDK will not be discoverable and linkers must be hardcoded in `cargo/config.toml`.
   - Also verify these binaries are on PATH: `adb` (from platform-tools), `sdkmanager` (from cmdline-tools), `keytool` (from JDK).
2. For a fresh `npm create tauri-app@latest` project, run `npm install` first, then `npm run tauri android init`.
3. Initialize or inspect Tauri Android output: `npm run tauri android init` if Android has not been initialized; otherwise inspect existing `src-tauri/gen/android`.
4. Fix frontend dev server settings for device live reload:
   - `package.json` dev script should run Vite with host binding, usually `vite --host`.
   - `vite.config.*` should use port `1420`, `strictPort: true`, and `host: process.env.TAURI_DEV_HOST || false`.
   - If HMR is configured, align it with `TAURI_DEV_HOST`, commonly WebSocket port `1421`.
5. Fix Gradle dependency resolution when networking fails:
   - Add proxy values to `src-tauri/gen/android/gradle.properties` only when the user uses a local proxy.
   - Validate with `cd src-tauri/gen/android` then `./gradlew tasks --info`.
6. Fix Rust Android linking:
   - First check if `ANDROID_NDK_HOME` is set. If not, check `NDK_HOME` and suggest `set -gx ANDROID_NDK_HOME $NDK_HOME` (fish) or `export ANDROID_NDK_HOME=$NDK_HOME` (bash/zsh).
   - If Cargo still cannot resolve linkers (env var expansion in `config.toml` is unreliable), create `src-tauri/.cargo/config.toml` with absolute paths to the NDK linker binaries for `aarch64-linux-android`, `armv7-linux-androideabi`, `i686-linux-android`, and `x86_64-linux-android`.
   - Always verify the linker binaries actually exist at the expected NDK path before writing them into config.
7. Run on a real device:
   - Ensure the phone is connected and authorized before `npm run tauri android dev -- --verbose`.
   - If the app is white, inspect the Android WebView through Chrome at `chrome://inspect/#devices`.
8. Configure release signing only after debug builds run:
   - Generate a keystore in `src-tauri/gen/android/app/`.
   - Create `src-tauri/gen/android/keystore.properties`.
   - In `app/build.gradle.kts`: add `signingConfigs` BEFORE `buildTypes` (ordering matters — Gradle will fail with `SigningConfig not found` if `signingConfigs` comes after `buildTypes`).
   - Add `signingConfig = signingConfigs.getByName("release")` in the `release` build type.
   - Add `*.jks` and `keystore.properties` to `.gitignore`.
   - Build with `npm run tauri android build` (NOT `npm run tauri build` which targets desktop).

## Output Style

When helping a user:

- Start with a short diagnosis of which bucket the issue belongs to.
- Give exact files and commands.
- Prefer small patches over large rewrites.
- For signing, include placeholders for secrets instead of real values.
- End with the next verification command.
