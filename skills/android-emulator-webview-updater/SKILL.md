---
name: android-emulator-webview-updater
description: Update the Android System WebView on a local emulator/AVD whose built-in WebView is too old to render modern sites (typically API 21-28 images). Lists the available AVDs and asks which one to update, takes a WebView APK path from the user, validates that APK against the image (package allowlist, ABI, minSdk/maxSdk, signature), then installs it — plain `adb install -r` when signatures allow, otherwise by replacing the system WebView APK under `-writable-system`. Verifies the result by loading a real page and reading the UA. Use when the user says "update webview on the emulator", "the emulator's webview is outdated", "install com.android.webview.apk on <AVD>", "web pages don't work on my Android 7 emulator", or similar.
---

# Update Android System WebView on an emulator

Old system images ship an ancient WebView (e.g. API 25 Google APIs ships Chrome 55) and many sites break on it. This skill replaces it with a modern build.

## Hard rules

1. **Never pick the AVD for the user.** Always list AVDs and ask, even if only one exists — a wrong guess reboots/modifies the wrong emulator.
2. **The user supplies the APK.** Never download a WebView APK from a mirror on their behalf. If the file they gave is unusable, say exactly what to download instead (see Phase 3) and stop.
3. **Back up the original system APK before overwriting it**, to a path outside the scratchpad (e.g. `~/Downloads/WebViewGoogle-<oldVersion>-original.apk`).
4. **Ask before the `/system` route.** Writing into `/system` needs a reboot with `-writable-system`; confirm with the user first. Never `-wipe-data` or delete an AVD to "fix" anything without explicit approval.
5. **Verify by rendering a real page**, not by `dumpsys` alone — `dumpsys` shows what is installed, the UA string shows what actually loads.
6. **Never verify persistence against a quick-boot snapshot.** A restored snapshot reports whatever `/system` looked like when it was saved, in *either* direction — it will happily show you a success that isn't real, or a revert that isn't real. Always delete `snapshots/default_boot` before a persistence check.
7. Never stream long logs to the console; grep what you need.

## Phase 1 — Pick the AVD

```bash
~/Library/Android/sdk/emulator/emulator -list-avds
adb devices -l                     # is one already running?
```

Ask the user which AVD to update (`AskUserQuestion`, one option per AVD) — unless they already named it. If their target is already running, reuse it; otherwise boot it:

```bash
~/Library/Android/sdk/emulator/emulator -avd <AVD> > /tmp/emu.log 2>&1 &
adb wait-for-device
# poll until: adb shell getprop sys.boot_completed == 1
```

**Several emulators are often running at once.** Never assume a serial. Map serials to AVD names and pin the target for every later command:

```bash
for d in $(adb devices | awk '/^emulator-/{print $1}'); do
  echo "$d -> $(adb -s "$d" emu avd name | head -1 | tr -d '\r')"
done
export ANDROID_SERIAL=emulator-XXXX     # use this, not -s, so every adb call is pinned
```

The port can change across restarts, so re-resolve the serial after every emulator relaunch rather than reusing the old one.

## Phase 2 — Profile the image (decides everything downstream)

```bash
adb shell getprop ro.build.version.release   # Android version
adb shell getprop ro.build.version.sdk       # API level
adb shell getprop ro.product.cpu.abilist     # arm64-v8a only? 64+32?
adb shell getprop ro.build.type              # userdebug => root available
adb shell getprop ro.debuggable              # 1 => WebViewUpdateService skips signature checks
adb shell settings get global webview_provider
adb shell dumpsys package com.google.android.webview | grep -E "versionName|codePath"
adb shell pm list packages -f | grep -i webview
```

**Allowed WebView providers** — an APK whose package is not on this list can never become the provider, no matter how it is installed:

```bash
adb pull /system/framework/framework-res.apk <scratch>/
~/Library/Android/sdk/build-tools/<ver>/aapt2 dump xmltree <scratch>/framework-res.apk \
  --file res/xml/config_webview_packages.xml | grep packageName
```

Typical Google APIs image: `com.android.chrome`, `com.google.android.webview` (fallback), `com.chrome.beta/dev/canary`, `com.google.android.apps.chrome`. A plain AOSP image lists `com.android.webview` instead.

Signer of the current system WebView (needed to predict whether `adb install -r` can work):

```bash
adb pull /system/app/WebViewGoogle/WebViewGoogle.apk <scratch>/
<build-tools>/apksigner verify --print-certs <scratch>/WebViewGoogle.apk | grep -E "DN:|SHA-256"
```

## Phase 3 — Validate the user's APK before touching anything

```bash
<build-tools>/aapt2 dump badging <apk> | grep -E "^package|SdkVersion|native-code"
<build-tools>/apksigner verify --print-certs <apk> | grep -E "DN:|SHA-256"
```

Reject and tell the user what to get instead if any of these fail:

| Check | Requirement | If it fails |
|---|---|---|
| Package name | Must appear in the allowlist from Phase 2 | An AOSP `com.android.webview` APK is **inert** on a Google APIs image (installs fine, `set-webview-implementation` refuses it). Ask for the Google `com.google.android.webview` build — or vice versa on an AOSP image. |
| ABI | `native-code`/`lib/` must contain an ABI in `ro.product.cpu.abilist` | Modern emulator images are often **arm64-v8a only** — an `armeabi-v7a` APK cannot run at all. Ask for the `arm64-v8a` variant. |
| `minSdkVersion` | ≤ device API | Version too new for this image. |
| `maxSdkVersion` | ≥ device API (when present) | WebView builds are capped: e.g. **119.0.6045.x is the last WebView for Android 7** (`minSdk 24`, `maxSdk 28`). Tell the user the ceiling for their API level. |
| Signer SHA-256 | Only matters for the fast path | Mismatch vs the system APK just means Phase 4 must use the `/system` route. Modern WebView is Play-signed, 2016-era system APKs are not — a mismatch is normal. |

APKMirror guidance to give the user: app **Android System WebView** (Google LLC), the version allowed by the table above, variant **arm64-v8a**, **nodpi**, file type **APK** (not APKM/XAPK bundle).

## Phase 4 — Install

### Fast path — try this first

```bash
adb install -r <apk>
```

Succeeds only when the APK's signature matches the installed system package (e.g. Play-image AVDs, or same-key APKs). On success go to Phase 5.

`INSTALL_FAILED_UPDATE_INCOMPATIBLE: signatures do not match` → the fast path is impossible for *any* modern APK on that image. Use the system route.

### System route (requires root + `-writable-system`)

Precondition: `ro.build.type` is `userdebug`/`eng`. **A Google Play AVD (`user` build) cannot do this** — `adb root` is refused; there the only option is updating WebView through the Play Store, or recreating the AVD from a non-Play "Google APIs"/AOSP image. Say so and stop rather than improvising.

Confirm with the user, then:

```bash
adb emu kill                 # wait for the qemu process to actually exit
~/Library/Android/sdk/emulator/emulator -avd <AVD> -writable-system -no-snapshot-load > /tmp/emu-ws.log 2>&1 &
# wait for sys.boot_completed == 1
adb root && adb wait-for-device && adb remount
adb shell df /system         # need free space >= APK size
```

Back up the original (Phase 1 pull already has it) to `~/Downloads/`, then swap:

```bash
adb push <apk> /data/local/tmp/wv.apk
adb shell 'rm -rf /system/app/WebViewGoogle/oat && \
  cp /data/local/tmp/wv.apk /system/app/WebViewGoogle/WebViewGoogle.apk && \
  chmod 644 /system/app/WebViewGoogle/WebViewGoogle.apk && \
  chown root:root /system/app/WebViewGoogle/WebViewGoogle.apk && \
  restorecon /system/app/WebViewGoogle/WebViewGoogle.apk; rm /data/local/tmp/wv.apk'
adb reboot
```

Why this works: a `/system` app's signature is compared against nothing at scan time, and on a debuggable build `WebViewUpdateService` skips provider signature validation. Deleting `oat/` drops the stale odex for the old APK. Keep the *system* path/filename — the directory name is irrelevant, but the package name inside the APK must still be the allowlisted one.

If the provider does not switch by itself after reboot:

```bash
adb shell cmd webviewupdate set-webview-implementation <package>
```

### The `/system` change only exists under `-writable-system`

This is the part that surprises people, so state it to the user explicitly:

- The write lands in `~/.android/avd/<AVD>.avd/system.img.qcow2`. That overlay is mounted **only when the emulator is launched with `-writable-system`**. A plain launch (Android Studio's play button, `emulator -avd <AVD>`) mounts the pristine SDK system image and the old WebView is back — the overlay is not lost, just not used.
- Quick-boot snapshots mask this in both directions: a snapshot saved from a `-writable-system` session restores the *new* WebView on a plain launch, and a snapshot saved from a plain launch pins the *old* one. The emulator also silently invalidates a snapshot when launch flags differ from when it was saved, so the same command can give different answers on different days.
- So after the swap: **delete `~/.android/avd/<AVD>.avd/snapshots/default_boot`** (with the user's OK), then confirm the version with one launch *with* the flag and, if the user cares, one *without*, so they see the real behaviour rather than a snapshot artifact.
- Practical follow-up to offer: an alias/script so the AVD is always started correctly, e.g.
  `alias <avd>='~/Library/Android/sdk/emulator/emulator -avd <AVD> -writable-system &'`

## Phase 5 — Verify

```bash
adb shell settings get global webview_provider
adb shell dumpsys package <package> | grep -E "versionName|codePath"

adb logcat -c
adb shell am start -n org.chromium.webview_shell/.WebViewBrowserActivity \
  -a android.intent.action.VIEW -d "https://httpbin.org/user-agent"
sleep 15
adb logcat -d | grep -E "WebViewFactory|FATAL"      # expect: Loading <pkg> version <new>
adb exec-out screencap -p > <scratch>/wv.png        # read it: UA must say Chrome/<new version>
```

`org.chromium.webview_shell` (Browser2) ships on most AOSP/Google APIs images. If it is absent, launch any installed app with a WebView instead and read the `WebViewFactory: Loading …` logcat line.

## Phase 6 — Report

Tell the user:
- Old → new version, and which route was used.
- **How to launch it from now on**: `-writable-system` every time, or the old WebView comes back (see the section above). This is the single most important line of the report.
- Where the original APK backup is, for reverting.
- The version ceiling for that API level (no point retrying with a newer APK later).
- Offer to uninstall any inert WebView APK left in `/data/app` from a failed attempt (`adb uninstall <pkg>`) — these are large (~200 MB).

Delete `/tmp/emu*.log` and pulled APKs from the scratchpad when done.

## Last resort — patching the provider allowlist (avoid)

If the only usable APK has a package name that is not allowlisted, the allowlist lives in binary XML inside `/system/framework/framework-res.apk`. Editing it means rewriting an AXML string pool and repacking a platform-key-signed APK that shares `android.uid.system` — a bad edit bootloops the AVD. Do not do this silently: explain the risk and get explicit approval, and prefer telling the user to download the correctly-named APK instead.
