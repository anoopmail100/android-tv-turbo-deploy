---
name: turbo-gradle-build
description: Speeds up repeated Gradle/Android build-edit-deploy cycles within a session by enabling the Gradle daemon, configuration cache, build cache and parallel execution, plus trimming unnecessary adb install flags. Use whenever iterating on an Android/Gradle project (edit code, rebuild, reinstall to a device/emulator, repeat) and builds feel slow, or the user asks to speed up builds/deploys.
---

# Turbo Gradle Build

Most of the time lost in an edit → rebuild → reinstall loop isn't compilation — it's Gradle re-starting cold (JVM boot + re-evaluating build scripts) on every single invocation, plus `adb install` doing more work than it needs to. This skill fixes both.

Measured on a small single-module Android app: a fully cached rebuild went from **~35s** (`--no-daemon`, no config cache) to **~2s** (warm daemon + reused configuration cache). A real recompile after an actual code change still dropped from ~35s to **~17s**. `adb install` dropped from using `--user all` (installs for every device user profile) to a plain `-r` install when only one user profile is ever active, shaving a few more seconds per deploy.

## 1. Enable the Gradle daemon — stop passing `--no-daemon`

`--no-daemon` forces a cold JVM + Gradle boot on every call. Drop it. The first build in a session pays the daemon startup cost once; every build after reuses the warm daemon.

```bash
./gradlew assembleDebug
```

not

```bash
./gradlew assembleDebug --no-daemon
```

If a project's `gradle.properties` or CI docs explicitly disabled the daemon for a reason (e.g. flaky CI runners), don't fight that — this skill is for local iterative dev loops, not CI.

## 2. Turn on caching + parallel execution in `gradle.properties`

Add (or merge in) these lines:

```properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.vfs.watch=true
```

- `org.gradle.caching=true` — reuses task outputs across builds (build cache).
- `org.gradle.configuration-cache=true` — skips re-evaluating build scripts when nothing relevant changed. This is usually the single biggest win for small/medium projects. Requires a reasonably modern Gradle (7.6+) and AGP (7.4+/8.x); if the build fails with configuration-cache-related errors on an older plugin, drop this one line rather than abandoning the rest.
- `org.gradle.parallel=true` — runs independent module tasks concurrently. Low impact on a single-module app, meaningful on multi-module projects.
- `org.gradle.vfs.watch=true` — keeps a file-system watch so Gradle doesn't rescan the whole tree every build (default on in modern Gradle, but harmless to set explicitly).

Bump JVM heap a little if you have headroom, since daemon + config cache hold more in memory between runs:

```properties
org.gradle.jvmargs=-Xmx2560m
```

## 3. Trim `adb install` flags

`--user all` installs the APK for every user profile on the device (e.g. a dormant secondary Android TV profile that's never actually used), which costs real time for no benefit. Only use it if you've confirmed multiple profiles are genuinely active and need the app. Otherwise:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

## 4. Verify, don't just assume

After applying these, time a couple of real rebuilds to confirm the win on this specific project — some plugins or build scripts don't play well with configuration cache yet, and it's better to find that out with `time ./gradlew assembleDebug` than to claim a speedup that didn't actually happen:

```bash
time ./gradlew assembleDebug   # first run: cold daemon, builds config cache
time ./gradlew assembleDebug   # second run: should be dramatically faster if nothing changed
touch app/src/main/java/.../SomeFile.kt   # force a real recompile
time ./gradlew assembleDebug   # realistic incremental-change timing
```

If configuration cache causes problems (plugin incompatibility, weird errors mentioning "configuration cache"), remove just that one property and keep the daemon + caching + parallel changes — they're independent and still worth it on their own.
