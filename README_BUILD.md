# App Cloner (Parallel Space-style, open source) — build notes

Engine: **BlackBox** (virtual app container) — this package is the most-starred
surviving fork of the original project (github.com/FBlackBox/BlackBox). The
original author deleted his repo in 2022; forks are the only source now.
Licence: Apache-2.0 (see LICENSE). App shell module: `:app`
(applicationId `top.niunaijun.blackboxa` — rename it if you rebrand).

## Build
- JDK **11**, Gradle wrapper included (7.3.3), AGP 7.2.0
- Command: `./gradlew :app:assembleDebug`
- APK out: `app/build/outputs/apk/debug/`
- The GitHub workflow `build.yml` does exactly this and uploads the APK as
  an artifact named `cloner-app-debug-apk`.

## Known limits (read before testing)
- Codebase is from 2022: compileSdk 32 / targetSdk 30. Most apps clone and
  run, but some modern apps will fail inside the container.
- Banking apps and apps with Play Integrity / SafetyNet checks generally
  refuse to run in ANY virtual cloner (Parallel Space included). Google
  sign-in inside clones often fails too.
- The engine manifest (Bcore) declares ~185 permissions. That is the
  standard virtual-cloner design: the host app holds permissions on behalf
  of the guest apps it runs. Expect the permission list to look scary; it
  is not, by itself, data collection.
- If dependency resolution fails, delete the `jcenter()` lines from
  `build.gradle` (jcenter shut down); the dependencies live on
  mavenCentral / google / jitpack.

## Quick scan done on this copy (2026-11-14, NOT a full audit)
- No Firebase / Crashlytics / AppsFlyer / Adjust / Umeng / Bugly /
  analytics SDKs in gradle files or code.
- No telemetry endpoints in code; hardcoded URLs are licence/doc links
  plus the original dev's Telegram group link.
- App shell manifest asks only for QUERY_ALL_PACKAGES; the big
  permission pool lives in the engine (Bcore) manifest, as explained above.
