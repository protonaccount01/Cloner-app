# Security Audit — BlackBox App Cloner (BlackBoxing/BlackBox fork)

- **Tree audited:** `~/workspace/blackbox_src` — the exact tree the user will build (BlackBoxing/BlackBox fork of the BlackBox virtual-app engine, Apache-2.0)
- **Date:** 2026-11-14
- **Method:** full source enumeration per `open_source_app_security_audit` skill — every network destination, every capture/input channel, every prebuilt binary, all manifests, fork-provenance diff against two independent forks, signing check.
- **Static source audit only.** The app was NOT built or run (user builds on GitHub Actions only). Runtime behaviour was not observed.

## Verdict: LOW risk — no malware, no trackers, no exfiltration code found in the source

Across all 855 files (~575 Java/Kotlin sources, 28 C++ sources, all manifests and Gradle files):

- **Zero** HTTP client usage (no OkHttp, Retrofit, Volley, HttpURLConnection, URLConnection, sockets) in host or engine code.
- **Zero** POST requests, **zero** `Base64.` calls, **zero** analytics/tracker SDKs or endpoints (Firebase, Crashlytics, AppsFlyer, Adjust, Umeng, Bugly, Mixpanel, Amplitude, Sentry — searched by name across the whole tree; the only hits are template comments in `.gitignore` files).
- The host app's own code contains **no** microphone, camera, contacts, SMS, call-log, or device-ID collection code at all.
- The only network traffic the app itself can generate is listed in Finding N1–N2 below. Everything else network-shaped in the tree is proxy/hook infrastructure that serves *guest* (cloned) apps, or javadoc comment links that are never executed.

The caveats that matter are in Findings F1–F4 and the Blind Spots section. None of them is a tracker or malware; F1 is a real privacy wart inherited from upstream, and F2 is a repo-hygiene action the user should take before pushing the tree to GitHub.

---

## Findings (things that deserve attention)

### F1 — MEDIUM (privacy wart, local only): the engine continuously records logcat to a world-readable public folder, automatically, from first launch

- **File:** `Bcore/src/main/java/top/niunaijun/blackbox/BlackBoxCore.java:138` (call), `:420–427` (implementation), using `Bcore/src/main/java/top/niunaijun/blackbox/utils/ShellUtils.java:117` (`Runtime.exec` into `sh`).
- **Trigger:** AUTOMATIC. `startLogcat()` runs unconditionally inside `doAttachBaseContext()` whenever the main process starts. There is no settings toggle and no configuration flag (searched — none exists). It runs `logcat -c` then `logcat -f <file>`, i.e. a never-ending log capture for as long as the process lives.
- **Data:** the Android log stream of the host app's own UID. Because cloned (guest) apps run *inside* the host's processes, their log lines are captured too — guest apps that log tokens, URLs or user data would write them into this file.
- **Destination:** a local file only — `Downloads/<host package name>_logcat.txt` in the **public** Downloads directory (`Environment.getExternalStoragePublicDirectory(DIRECTORY_DOWNLOADS)`). Any app with storage access, and anyone holding the phone, can read it. **No network destination exists for this data** — there is no upload code anywhere in the tree.
- **Provenance:** NOT a fork addition. The identical call and method exist in both independent reference forks (warren-bank mirror `BlackBoxCore.java:133,413`; AmySama fork `:133,413`). This is upstream BlackBox behaviour.
- **Note:** a related crash library, `xcrash.XCrash.init(context)` (`BlackBoxCore.java:164`, dependency `Bcore/build.gradle:81`), is initialised with defaults only — no callback and no uploader is configured anywhere in the tree, so crash tombstones are written locally by default. `PineConfig.debug = true` is also hard-coded (`BlackBoxCore.java:162`), increasing local log verbosity.

### F2 — Repo hygiene (act before pushing to GitHub): the tree ships a GitHub Actions workflow with a push trigger that calls the GitHub API with a repo secret

- **File:** `.github/workflows/upstream.yml` (ships inside the source tree).
- **What it does, factually:** on every push to `master` it would run in the user's own repo and make an authenticated GitHub API call (using a secret named `TOKEN`, if one exists) asking GitHub to merge an upstream branch from a third-party repository (`AutoBlackBox/BlackBox`) into the fork.
- **Risk:** it is not app code and cannot affect the built APK. The risk is to the user's repo: an unexpected automatic action on push, using the user's own secret if present, pulling third-party code into his tree without review.
- **Recommendation:** delete `.github/workflows/upstream.yml` from the repo before the first push. Left in the audited zip deliberately (audit changes no code except proven malware/trackers — see Removal Log), but flagged here as the first thing to remove.

### F3 — LOW: app-wide cleartext HTTP is permitted by config

- **File:** `app/src/main/res/xml/network_security_config.xml` — `<base-config cleartextTrafficPermitted="true" />`, referenced from `app/src/main/AndroidManifest.xml:13`.
- **Effect:** the host app (and WebViews inside cloned apps) are *allowed* to use unencrypted HTTP. The host itself makes no HTTP requests (see N-findings), so this is a permissive setting rather than an observed behaviour — but any guest app/WebView content inherits the allowance.

### F4 — LOW / by-design, stated plainly: what a virtual cloner inherently is

- Cloned apps run **inside the host app's UID and processes**. A cloned app with bad behaviour can use every permission the host holds and its own network access exactly as if it were installed normally. The engine adds no collection of its own, but it also cannot make a malicious guest app safe. Clone only apps you would install anyway.
- The engine contains extensive **guest-facing spoofing** (this is a feature, not collection): it intercepts guest apps' calls for IMEI/device-id/subscriber-id (`Bcore/.../fake/service/ITelephonyManagerProxy.java:62–117`, `IPhoneSubInfoProxy.java:63`), returns a hard-coded fake Wi-Fi MAC `ac:62:5a:82:65:c4` (`IWifiManagerProxy.java:62`), rewrites `Build.*` fields to pose as a Xiaomi Mi 10 for guests (`Bcore/.../fake/service/BuildProxy.kt`, fork-added file, read in full — spoof values only), hooks `__system_property_get` natively (`Bcore/src/main/cpp/Hook/SystemPropertiesHook.cpp:32`), and provides user-configured fake GPS location. Data flows: values flow *to* guest apps; nothing is read for the host's own use and nothing leaves the device.
- The app can install a **GMS (Google services) stack and Xposed modules** into the container (`GmsManagerActivity`, `XpActivity`) — from packages the user supplies/selects locally; no download URL exists in the code for either.

---

## Network destinations — complete enumeration

### N1 — Map tiles from Wikimedia (the only in-app network fetch)

- **File:** `app/src/main/java/top/niunaijun/blackboxa/view/fake/FollowMyLocationOverlay.kt:88` — `binding.map.setTileSource(TileSourceFactory.WIKIMEDIA)` (osmdroid, dependency `app/build.gradle:163`).
- **Trigger:** USER ACTION ONLY — fires when the user opens the Fake Location map picker (menu → Fake Location) and pans/zooms the map.
- **Data sent:** ordinary HTTPS map-tile GET requests to Wikimedia's tile servers. As with any tile fetch, the viewed map region and the device IP are visible to the tile server. No account data, no identifiers are added by the app code (osmdroid's default user-agent is sent by the library).
- **Destination:** maps.wikimedia.org tile service.

### N2 — Two browser links (no data sent by the app)

- **Files:** `app/src/main/java/top/niunaijun/blackboxa/view/main/MainActivity.kt:160` (menu item → `https://github.com/BlackBoxing/BlackBox`) and `:170` (menu item → `https://t.me/blackboxing`).
- **Trigger:** explicit menu taps only. The app fires an `ACTION_VIEW` intent; the browser opens the page. No payload, no referrer data beyond what the browser itself sends.

### N3 — Everything else searched and found EMPTY (clean findings, same precision)

- HTTP clients (`OkHttpClient`, `Retrofit`, `Volley`, `HttpURLConnection`, `URLConnection`, `openConnection`, `Socket(`, `WebView.loadUrl`, Cronet): **0 hits** in app + engine code. The only `WebView` reference is `BActivityThread.java:335` setting a per-guest data-directory suffix (no loading), plus reflection stubs in `android-mirror`.
- `"POST"`, `setRequestMethod`, `Request.Builder`, `RequestBody`: **0 hits**.
- `Base64.`: **0 hits** in the entire tree.
- `DownloadManager`: **0 hits**; package installation is from local files/installed apps only (`BlackBoxCore.java:228–251`, `app/.../AppsRepository.kt:162–164`).
- Native code (`Bcore/src/main/cpp`, `pine-core/src/main/cpp`): **0** socket/connect/curl code; all URL strings in native files are source-attribution comments.
- URL literals across all code: every remaining hit is a javadoc/comment link (developer.android.com, androidxref, Apache licence headers, a blog reference at `Bcore/.../entity/location/BCell.java:15`, an author credit at `ShellUtils.java:169`) — none is executed.
- Build-time only: Gradle resolves dependencies from `google()`, `mavenCentral()`, `jitpack.io` (`build.gradle:7–11, 21–25`), and a *commented-out* Umeng SDK repository line (`build.gradle:11, 25`) — commented, so Umeng is **not** included. These are build-machine downloads, not app traffic.

---

## Capture / input channels — complete enumeration

- **Screenshot / media projection:** NONE in host code. `MediaProjection`, `createScreenCaptureIntent`, `ImageReader`: 0 hits. The single `createVirtualDisplay` hit (`Bcore/.../fake/service/IDisplayManagerProxy.java:46–51`) is a hook that intercepts *guest* apps' display calls.
- **Accessibility:** NO accessibility service is declared or implemented by the host. The only reference (`IAccessibilityManagerProxy.java:46`) intercepts guest apps' queries about accessibility services.
- **Microphone / camera:** NO host code uses `MediaRecorder`, `AudioRecord`, or any Camera API (0 hits in the app module; engine hooks exist only to proxy guest apps' framework calls).
- **Contacts / SMS / call log:** NO host code reads them (0 hits for `ContactsContract`, `Telephony.Sms`, `CallLog`, `content://sms` in the app module). The engine proxies the corresponding system services for guests, as a container must.
- **Clipboard:** the host declares legacy clipboard-related permissions (see Manifests) and the engine proxies `IClipboard` for guests (`android-mirror` stubs only — reflection definitions, no reading code). No host feature reads the user's clipboard.
- **Shell execution:** `Runtime.exec`/`ProcessBuilder` call sites, all local and enumerated:
  - `BlackBoxCore.java:424–425` — `logcat -c` and `logcat -f <file>` (Finding F1).
  - `Bcore/.../utils/FileUtils.java:109` — `chmod` on the app's own files; `:121` — `ln -s` symlink creation inside the app's sandbox.
  - `ShellUtils.java:117` — the generic `sh`/`su` runner behind the above. Its `checkRootPermission()` helper has **no callers** anywhere in the tree (dead code, searched). BlackBox also advertises *hiding* root from guests (`isHideRoot`), the opposite direction.
- **Dynamic code loading:** all local, all expected for a container, none remote:
  - `System.loadLibrary("blackbox")` (`NativeCore.java:34`) and `System.loadLibrary("pine")` (`PineConfig.java:43`) — the engine's own native libraries, built from the C++ sources in this tree.
  - `DexFile` on the bundled `empty.jar` asset (`NativeCore.java:96`) — a classloader warm-up trick; see Blind Spots for the jar itself.
  - `ModuleClassLoader extends PathClassLoader` (`pine-xposed/.../ModuleClassLoader.java:24–27`) — loads Xposed modules the user installs into the container.
  - No code path loads dex/code from the network (there is no network code to fetch it with — see N3).
- **Hidden features / unusual constants:** the fork-added files were each read (see Provenance). `BuildProxy.kt` hard-codes a fake device identity (Xiaomi Mi 10 / "umi") served to guests. `IWifiManagerProxy.java:62` hard-codes the fake MAC above. No hidden keys, no encoded payload strings (searched for long base64-like string literals: 0 hits), no undocumented UI triggers found; the app's feature surface (clone, fake location, GMS manager, Xposed manager, settings) matches its menus.

---

## Manifests & permissions

- **App module** (`app/src/main/AndroidManifest.xml`): exactly **1** permission — `QUERY_ALL_PACKAGES` (needed to list installed apps for cloning). Activities are the app's own screens; `MainActivity` is the only exported entry. No receivers, no boot handling, no accessibility/media-projection declarations.
- **Engine module** (`Bcore/src/main/AndroidManifest.xml`): **185** `uses-permission` entries, 151 stub activities, plus `DaemonService` (foreground keep-alive for cloned apps; **off by default** — `BlackBoxLoader.kt:25`, default `false`, user-toggleable in Settings), `ProxyVpnService` (so a *guest* app can use VpnService), `SystemCallProvider`, and `ProxyBroadcastReceiver` (exported, guest-broadcast plumbing). All other module manifests declare **0** permissions.
- The 185-entry pool is the standard virtual-cloner pattern: the host must declare every permission a guest app might check for. Most entries are normal; a large block are **dead legacy permissions** Android no longer has (`READ_SOCIAL_STREAM`, `WRITE_OWNER_DATA`, `READ_PROFILE`, `PERSISTENT_ACTIVITY`, `RESTART_PACKAGES`, `GET_TASKS`, etc.) — inert. Several are **signature-level and cannot be granted** to a normal install (`INSTALL_PACKAGES`, `DELETE_PACKAGES`, `INTERACT_ACROSS_USERS_FULL`, `CLEAR_APP_USER_DATA`, `READ_LOGS`) — declared but Android will refuse them.
- **Flagged as odd even for a cloner:**
  - `READ_LOGS` — declared, and the engine *does* effectively exercise log access via the shell `logcat` in F1 (an app can read its own UID's logs without the permission; the declaration plus F1 together are the real story).
  - `MANAGE_EXTERNAL_STORAGE` — all-files access; broader than cloning strictly needs, present so guests can share the host's file access.
  - `GET_CLIPS` / `READ_CLIPS` / `WRITE_CLIPS` — non-standard OEM clipboard permissions, legacy leftovers.
  - `RECEIVE_BOOT_COMPLETED` — declared, but **no boot receiver exists** in any manifest (searched) and no boot-handling code was found; appears to be inert legacy. The marketing-relevant correction: despite the permission's presence, this build does not auto-start on boot (the DaemonService is a foreground service started while the app runs, default-off).

---

## Prebuilt binaries — complete inventory (audit blind spots)

Only three binary archives exist in the tree. **No `.so` files ship** — all native libraries are compiled at build time from the C++ sources in the tree (CMake, `Bcore/src/main/cpp`, `Bcore/pine-core/src/main/cpp`), which *were* read. **No `libs/` folder exists** in any module, so the `fileTree(dir:"libs")` declarations (`app/build.gradle:124`, `Bcore/build.gradle:75`) load nothing.

| File | Size | What it is | Audited? |
|---|---|---|---|
| `Bcore/src/main/assets/junit.jar` | 87,259 B | A jar containing a prebuilt `classes.dex` (192,572 B) of the JUnit test framework. Copied out at first run (`BlackBoxSystem.java:92–93`) and registered inside the sandbox as a fake system package so guest apps that probe for it see it. | **NO — binary blind spot.** Contents enumerated (file listing only); dex bytecode not decompiled. |
| `Bcore/src/main/assets/empty.jar` | 1,483 B | Near-empty jar (manifest + 1,872 B `classes.dex`); used for the classloader warm-up at `NativeCore.java:96` and as a placeholder package. | Listing only; bytecode not decompiled. |
| `gradle/wrapper/gradle-wrapper.jar` | 54,329 B | The standard Gradle wrapper bootstrap (build machine only, never enters the APK). | Not audited; standard component. |

**Remote binary dependencies (also blind spots — pulled as compiled artifacts at build time, not source-audited here):** the meaningful ones are `org.osmdroid:osmdroid-android:6.1.14` (map tiles, Finding N1), `com.iqiyi.xcrash:xcrash-android-lib:3.0.0` (local crash capture, F1), `com.bytedance.android:shadowhook:1.0.3` (native hooking), `org.lsposed.hiddenapibypass:hiddenapibypass:4.3`, `com.github.CodingGay.BlackReflection:core` (reflection), plus androidx/material and small UI libraries (`app/build.gradle`, `Bcore/build.gradle` dependency blocks — full list read; none is an ad/analytics library, and one, `com.gitee.cbfg5210:RVAdapter`, comes from a less-mainstream repo).

---

## Fork provenance diff

References cloned fresh for this audit: `warren-bank/mirror-Android-BlackBox` and `AmySama/BlackBox` (two independent forks of the original FBlackBox project).

- Both references agree with each other and are **older snapshots** than the target fork; the target (BlackBoxing) is a later continuation of upstream development, so a large number of files differ by version drift (Kotlin conversions such as `Resolution.java → Resolution.kt`, `ActivityRecord.java → .kt`, and many proxy refinements). Resource/translation noise ignored per method.
- **Files existing ONLY in the target fork** were each inspected: `Bcore/src/main/java/top/niunaijun/blackbox/fake/hook/ProxyVersion.java` (a version-gating annotation, header-dated 2022/5/24 by the fork author), `Bcore/.../fake/service/BuildProxy.kt` (guest device spoofing, Finding F4), `Bcore/.../fake/service/IBluetoothManagerProxy.java` (standard binder proxy for guests), `Bcore/src/main/cpp/Hook/SystemPropertiesHook.cpp/.h` (native `__system_property_get` hook serving guests). **None contains network, collection, or exfiltration code** — and all were inside the scope of the whole-tree greps in N3, which found nothing anywhere.
- The two user-visible links differ by fork branding only: references point to `github.com/FBlackBox/BlackBox` / `t.me/fvblackbox`; the target points to its own repo/channel (`MainActivity.kt:160,170`).
- F1 (logcat) and `XCrash.init` exist identically in both references → upstream behaviours, not fork injections.
- **Conclusion:** no evidence of code injected by this fork. Differences are consistent with continued upstream-style development and rebranding.

## Signing / keystores

- **No keystore, signing key, `.jks`, `.keystore`, `.pem`, `.p12` or `.pk8` file exists anywhere in the tree** (searched by name), and no `signingConfig`/`storeFile` reference exists in any Gradle file (searched). Builds from this tree will be signed by whatever key the user's own pipeline supplies (the GitHub Actions debug key by default) — the safe arrangement the audit skill recommends: install only builds from your own pipeline.

## Removal log

- **Nothing was removed.** No tracker, malware, or exfiltration code was found, so per the audit brief no app code was changed. The only addition to the tree is this report (`AUDIT.md` at the tree root).
- Pre-delivery re-scan after adding the report: the whole-tree greps for HTTP clients, POST, Base64, and tracker names were re-run against the final tree; results unchanged (0 hits beyond the documented comment/template mentions and this report's own text).

## Audit limits

- **Version checked:** the BlackBoxing/BlackBox tree exactly as delivered in `~/workspace/blackbox_src` on 2026-11-14 (fork snapshot; no VCS history in the tree to date it more precisely). A different download of the "same" fork is a different tree.
- **Channels searched:** all URL literals; HTTP client/socket/WebView APIs; POST construction; Base64; download managers; shell execution; dynamic code loading; screenshot/media-projection, accessibility, clipboard, mic, camera, contacts/SMS/call-log APIs; device-ID APIs; manifests of all 8 modules; all Gradle files; native C++ sources; fork-only files individually.
- **NOT audited:** the two asset jars' dex bytecode and all remote Maven/JitPack binary dependencies (listed above) — behaviour claims for those rest on their public reputations, not on this audit. Runtime/dynamic behaviour was not observed (no build, no device run) — e.g. what guest apps do inside the container is the guest apps' own behaviour, out of scope by definition.
- **Scoped verdict:** "no exfiltration" applies to the host app + engine source in this tree, over the channels above. It is not a claim about cloned apps, about the binary dependencies, or about any other build of BlackBox obtained elsewhere.

---

## Post-audit removals (2026-11-14)

At the owner's request ("remove if not needed — I want directly risk-free"), the following non-essential items identified by this audit were removed from the tree after the audit above was completed. The findings above describe the tree **as audited**; this section describes the tree **as delivered**. A short summary also ships in the tree as `REMOVED.md`.

### Removed

1. **Finding N1 — Fake Location feature + osmdroid (the tree's only network fetch).** Deleted files: `app/src/main/java/top/niunaijun/blackboxa/view/fake/` (all 5 files: `FollowMyLocationOverlay.kt`, `FakeManagerActivity.kt`, `FakeLocationAdapter.kt`, `FakeLocationFactory.kt`, `FakeLocationViewModel.kt`), `app/src/main/java/top/niunaijun/blackboxa/bean/FakeLocationBean.kt`, `app/src/main/java/top/niunaijun/blackboxa/data/FakeLocationRepository.kt`, `app/src/main/res/layout/activity_osmdroid.xml`, `app/src/main/res/layout/item_fake.xml`. Edited: `app/src/main/AndroidManifest.xml` (both activity declarations removed), `app/src/main/java/top/niunaijun/blackboxa/view/main/MainActivity.kt` (fake-location menu handler + import removed), `app/src/main/res/menu/menu_main.xml` (`fake_location` item removed), `app/src/main/java/top/niunaijun/blackboxa/util/InjectionUtil.kt` (repository field, factory method and both imports removed), `app/build.gradle` (osmdroid dependency removed). The shared `activity_list.xml` layout was kept (also used by `ListActivity`). **Pass-through verified in the kept engine code:** `BLocationManagerService.getOrCreateConfig()` initialises every package's pattern to `BLocationManager.CLOSE_MODE`; `BLocationManager.isFakeLocationEnable()` is therefore false when nothing was ever set; and every hook in `ILocationManagerProxy.java` first invokes the real system method (`method.invoke(who, args)`) and only alters the result when fake location is enabled. With the UI gone, nothing in the app module can set a pattern (the only remaining `BLocationManager` caller, `RockerManager.kt`, only offsets a location when one is already enabled).
2. **Finding N2 — browser links.** The `main_git` and `main_tg` handlers (and the now-unused `android.net.Uri` import) were removed from `MainActivity.kt`, and both items were removed from `res/menu/menu_main.xml`. The main menu now contains only Settings. (The repo `README.md`/`README_ZH.md` still mention the Telegram channel as documentation text — not app code.)
3. **xcrash (part of Finding F1's note).** Removed `xcrash.XCrash.init(context);` from `BlackBoxCore.java` and `implementation 'com.iqiyi.xcrash:xcrash-android-lib:3.0.0'` from `Bcore/build.gradle`. No other xcrash references existed (no manifest entries).
4. **Finding F1 — automatic logcat dump.** Removed the unconditional `startLogcat();` call in `doAttachBaseContext()` and the entire `startLogcat()` method from `BlackBoxCore.java`, plus the imports that only it used (`android.os.Environment`, `utils.ShellUtils`, `utils.FileUtils` in that file). `ShellUtils` itself was kept untouched (it has no remaining callers after this removal; the `chmod`/`ln -s` sandbox calls live in `FileUtils` via `Runtime.exec` directly and are unaffected).
5. **`junit.jar` asset (Blind Spots table).** Deleted `Bcore/src/main/assets/junit.jar`; removed the copy lines and the `JUNIT_JAR` static import from `BlackBoxSystem.java` (`initJarEnv()` now copies only `empty.jar`), the `JUNIT_JAR` field from `BEnvironment.java`, and the commented-out `sharedLibraryFileList.add(BEnvironment.JUNIT_JAR...)` line from `PackageManagerCompat.java`. Removal was safe because that commented-out line was the jar's only consumer — the copied `junit.apk` was never read by any live code path.

### Kept, with reason

- **`empty.jar` + `NativeCore.loadEmptyDex()` — load-bearing, removal stopped per instructions.** The native engine resolves `loadEmptyDex` by name via JNI during native init (`Bcore/src/main/cpp/BoxCore.cpp:98`, `GetStaticMethodID`) and calls it (`BoxCore.cpp:59–61`) to obtain dex cookies from `new DexFile(EMPTY_JAR)` (`NativeCore.java:96`) for the guest dex-loading path. Deleting the jar or its copy code would leave the engine calling into a missing file on that path. It remains the tree's one unaudited binary (1,483 B, near-empty by design).
- Engine dependencies (Pine, shadowhook, hiddenapibypass, BlackReflection, AndroidX/UI libraries, `gradle-wrapper.jar`) — required for the cloner to run; untouched.
- Finding F2 was already resolved before this surgery (`.github/workflows/upstream.yml` deleted; the tree now contains no `.github` directory at all). Finding F3 (cleartext config) was out of scope for this surgery and is unchanged.

### Post-removal verification (static re-scan of the final tree)

- `osmdroid`, `wikimedia`, `xcrash`, `startLogcat`, `_logcat.txt`: **0 hits** in code, Gradle files, manifests and layouts (remaining hits are this report's historical text and `REMOVED.md`).
- Deleted names (`FollowMyLocationOverlay`, `FakeManagerActivity`, `FakeLocationAdapter`, `FakeLocationFactory`, `FakeLocationViewModel`, `FakeLocationBean`, `FakeLocationRepository`, `getFakeLocationFactory`, `ActivityOsmdroidBinding`, `ItemFakeBinding`, `JUNIT_JAR`, `activity_osmdroid`, `item_fake`, `R.id.main_git`, `R.id.main_tg`, `R.id.fake_location`): **0 dangling references** anywhere in code or XML.
- `junit`: remaining hits are test-only and unrelated to the removed asset — `testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"` strings in four module `build.gradle` files and `android-mirror/src/test/.../ExampleUnitTest.java` imports. Test sources never ship in the APK.
- The two link URLs: **0 hits** in app code (README documentation mentions remain, noted above).
- **Not compiled.** The tree was not built here (owner builds on GitHub Actions only); no claim is made that the build passes. Every edited file is listed above for eyeball review.
