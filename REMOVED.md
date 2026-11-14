# Post-audit removals — what was taken out of this tree and why

After the security audit (see `AUDIT.md`), everything that is **not strictly needed**
to run cloned apps was removed at the owner's request ("direct risk-free").
The audit found no malware and no trackers; these removals eliminate the remaining
non-essential surface: the only network fetch, a crash library, an automatic log
dump, a prebuilt binary with no live consumer, and two outbound browser links.

## Removed

1. **Fake Location feature (app module) + osmdroid** — the tree's only network code.
   The map screen fetched map tiles from Wikimedia (audit finding N1). Deleted:
   the whole `view/fake/` package (`FollowMyLocationOverlay.kt` map screen,
   `FakeManagerActivity.kt`, `FakeLocationAdapter.kt`, `FakeLocationFactory.kt`,
   `FakeLocationViewModel.kt`), `bean/FakeLocationBean.kt`,
   `data/FakeLocationRepository.kt`, layouts `activity_osmdroid.xml` and
   `item_fake.xml`, both manifest activity declarations, the main-menu entry,
   the fake-location wiring in `InjectionUtil.kt`, and the
   `org.osmdroid:osmdroid-android` dependency in `app/build.gradle`.
   The engine's (Bcore) location proxies were **kept**: with no fake location ever
   set, a package's location pattern defaults to `CLOSE_MODE` and every proxy hook
   passes the guest's request through to the real system location service
   (verified in `BLocationManagerService.java` and `ILocationManagerProxy.java`).

2. **Browser links in the main menu** — the GitHub and Telegram menu items and
   their handlers in `MainActivity.kt` (audit finding N2), plus the menu items
   in `res/menu/menu_main.xml`. The menu now contains only Settings.

3. **xcrash crash library** — the `xcrash.XCrash.init(context)` call in
   `BlackBoxCore.java` and the `com.iqiyi.xcrash:xcrash-android-lib` dependency
   in `Bcore/build.gradle`. Gone entirely; nothing replaces it.

4. **Automatic logcat dump (audit finding F1)** — the engine no longer writes a
   continuous logcat file to the public Downloads folder. The unconditional
   `startLogcat()` call and the method itself were removed from
   `BlackBoxCore.java`. `ShellUtils` was kept untouched (it now has no
   remaining callers; the engine's `chmod`/`ln -s` calls on its own sandbox
   files live in `FileUtils` via `Runtime.exec` and are unaffected).

5. **`junit.jar` prebuilt asset** — an unaudited binary whose only would-be
   consumer (registering it as a shared library for guests) was already
   commented out upstream, so the copied file was never read by live code.
   Deleted the asset and all wiring: the copy code in `BlackBoxSystem.java`
   (`initJarEnv()`), the `JUNIT_JAR` field in `BEnvironment.java`, and the
   commented-out reference in `PackageManagerCompat.java`.

## Deliberately kept

- **`empty.jar` asset + `NativeCore.loadEmptyDex()`** — load-bearing, **not**
  removable: the native engine resolves `loadEmptyDex` by name through JNI at
  init (`Bcore/src/main/cpp/BoxCore.cpp`) and calls it to obtain the empty-dex
  cookies used on the guest dex-loading path. Removing the jar or the method
  risks breaking cloned-app loading. The jar is a 1,483-byte near-empty
  placeholder (manifest + a 1,872-byte `classes.dex`) — see `AUDIT.md`,
  "Prebuilt binaries".
- **Engine dependencies** — Pine, shadowhook, hiddenapibypass,
  BlackReflection/freeReflection, AndroidX/Material and the small UI libraries,
  and `gradle/wrapper/gradle-wrapper.jar`. The cloner does not run without them.
- **Bcore location/GPS proxies and the `BLocation` plumbing** — pass-through
  when no fake location is set (see above). The game-rocker overlay
  (`RockerManager.kt`) can only *move* a fake location that is already enabled;
  after removal #1 nothing in the app can enable one.
- **Inert leftovers** — a few now-unreferenced string resources
  (`fake_location`, `open_source_path`, `tg_group`, …) remain in `strings.xml`,
  and the repo READMEs still link to the fork's Telegram channel as plain
  documentation text. Neither is code; neither runs. Unit-test sources that
  import JUnit (`android-mirror/src/test`) are test-only and never ship in the APK.

## Verification

After the surgery the whole tree was re-grepped: zero live references to
`osmdroid`, `wikimedia`, `xcrash`, `startLogcat`, the Downloads logcat path,
the two link URLs, or any deleted class/layout/field name (remaining hits are
this file, `AUDIT.md`'s historical text, README documentation, and test-only
sources). The tree was **not** compiled here — the owner builds on GitHub
Actions; `AUDIT.md` ("Post-audit removals") lists every edited file for review.
