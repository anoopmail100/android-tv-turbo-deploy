---
name: android-tv-turbo-deploy
description: Speeds up the edit-rebuild-sideload loop when developing an Android TV / Google TV app over ADB, by enabling the Gradle daemon, configuration cache, build cache and parallel execution, plus trimming unnecessary adb install flags for multi-profile TV devices. Use whenever iterating on an Android TV app (edit code, rebuild, adb install to the TV over wifi/USB, relaunch, repeat) and the loop feels slow, or the user asks to speed up builds/deploys to a TV.
---

# Android TV Turbo Deploy

Developing for Android TV means a tight loop: edit code, rebuild, `adb install` to the actual TV (often over wifi, which is already slower than a local emulator), relaunch, check the result on screen, repeat. Most of the time lost in that loop isn't compilation or the network transfer — it's Gradle restarting cold (JVM boot + re-evaluating build scripts) on every single invocation, plus `adb install` doing more work than it needs to on a TV device that may have multiple user profiles.

Measured on a small single-module Android TV app, deploying to a real Google TV device over `adb connect`: a fully cached rebuild went from **~35s** (`--no-daemon`, no config cache) to **~2s** (warm daemon + reused configuration cache). A real recompile after an actual code change still dropped from ~35s to **~17s**. `adb install` dropped from using `--user all` (installs for every TV user profile, including dormant secondary/kid profiles never actually used) to a plain `-r` install, shaving a few more seconds per deploy.

## 1. Enable the Gradle daemon — stop passing `--no-daemon`

`--no-daemon` forces a cold JVM + Gradle boot on every call. Drop it. The first build in a session pays the daemon startup cost once; every build after reuses the warm daemon.

```bash
./gradlew assembleDebug
```

not

```bash
./gradlew assembleDebug --no-daemon
```

If a project's `gradle.properties` or CI docs explicitly disabled the daemon for a reason (e.g. flaky CI runners), don't fight that — this skill is for local iterative dev loops against a real TV or emulator, not CI.

## 2. Turn on caching + parallel execution in `gradle.properties`

Add (or merge in) these lines:

```properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.vfs.watch=true
```

- `org.gradle.caching=true` — reuses task outputs across builds (build cache).
- `org.gradle.configuration-cache=true` — skips re-evaluating build scripts when nothing relevant changed. This is usually the single biggest win for small/medium TV apps. Requires a reasonably modern Gradle (7.6+) and AGP (7.4+/8.x); if the build fails with configuration-cache-related errors on an older plugin (e.g. an old Leanback or TV-specific Gradle plugin), drop this one line rather than abandoning the rest.
- `org.gradle.parallel=true` — runs independent module tasks concurrently. Low impact on a single-module TV app, meaningful if the project splits a TV module from a mobile module.
- `org.gradle.vfs.watch=true` — keeps a file-system watch so Gradle doesn't rescan the whole tree every build (default on in modern Gradle, but harmless to set explicitly).

Bump JVM heap a little if you have headroom, since daemon + config cache hold more in memory between runs:

```properties
org.gradle.jvmargs=-Xmx2560m
```

## 3. Trim `adb install` flags for TV devices specifically

Android TV / Google TV devices commonly ship with more than one Android user profile — e.g. an "Owner" profile plus dormant secondary or kid profiles that were set up once and never actually used. `--user all` installs the APK for *every* one of those profiles, which costs real time for no benefit if only one is ever active. Check first:

```bash
adb shell pm list users
adb shell dumpsys package <your.package.name> | grep -E "User [0-9]+:"
```

If only one user profile shows as actually running/used, drop `--user all`:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Only keep `--user all` if you've confirmed multiple profiles on the TV genuinely need the app (e.g. a kids' profile that also launches it).

## 4. Reconnect ADB fast after the TV sleeps or reboots

TVs often drop the ADB-over-wifi connection when they sleep, lose wifi briefly, or reboot. Don't waste time re-scanning the whole subnet each time — remember the TV's IP from the last successful `adb connect` and try that first:

```bash
adb connect <tv-ip>:5555
adb devices -l   # confirm it shows up as "device", not "unauthorized" or missing
```

Only fall back to a full network/ARP scan if the remembered IP no longer responds (the TV's DHCP lease may have changed).

## 5. Verify, don't just assume

After applying these, time a couple of real rebuild-and-deploy cycles to confirm the win on this specific project — some plugins or build scripts don't play well with configuration cache yet, and it's better to find that out with `time ./gradlew assembleDebug` than to claim a speedup that didn't actually happen:

```bash
time ./gradlew assembleDebug   # first run: cold daemon, builds config cache
time ./gradlew assembleDebug   # second run: should be dramatically faster if nothing changed
touch app/src/main/java/.../SomeFile.kt   # force a real recompile
time ./gradlew assembleDebug   # realistic incremental-change timing
time adb install -r app/build/outputs/apk/debug/app-debug.apk
```

If configuration cache causes problems (plugin incompatibility, weird errors mentioning "configuration cache"), remove just that one property and keep the daemon + caching + parallel changes — they're independent and still worth it on their own.
