# android-tv-turbo-deploy

A [Claude Code](https://claude.com/claude-code) skill that speeds up the edit → rebuild → sideload loop when developing an **Android TV / Google TV app over ADB**, by fixing the usual cause of slow local builds: Gradle restarting cold on every invocation instead of reusing a warm daemon and its configuration cache, plus trimming `adb install` flags that waste time on multi-profile TV devices.

Measured on a small single-module Android TV app, deploying to a real Google TV device over `adb connect`: a fully-cached rebuild dropped from **~35s** to **~2s**, and a real rebuild after an actual code change dropped from ~35s to **~17s**. `adb install` also got faster by dropping a redundant `--user all` flag that was reinstalling for a dormant secondary TV profile nobody used.

## What it does

- Stops passing `--no-daemon` so the Gradle daemon stays warm between builds
- Enables Gradle's build cache, configuration cache, and parallel execution via `gradle.properties`
- Trims unnecessary `adb install --user all` on TV devices once you've confirmed only one profile is active
- Remembers the TV's last-known IP for fast `adb connect` after it sleeps/reboots, instead of re-scanning the subnet
- Tells you to verify the win with `time ./gradlew assembleDebug` rather than just trusting it

See [`android-tv-turbo-deploy/SKILL.md`](android-tv-turbo-deploy/SKILL.md) for the full technique and the exact properties to add.

## Installing this skill

Claude Code skills live as a folder containing a `SKILL.md` with YAML frontmatter (`name`, `description`). To use this one:

1. Copy the `android-tv-turbo-deploy/` folder into your skills directory, e.g.:
   ```bash
   cp -r android-tv-turbo-deploy ~/.claude/skills/android-tv-turbo-deploy
   ```
   (or your project's `.claude/skills/` directory for a project-scoped skill)
2. Start a Claude Code session in your Android TV project. Claude will pick up the skill automatically when the build/deploy loop comes up, or you can ask for it directly: *"use the android-tv-turbo-deploy skill to speed up my builds"*.

## Why this exists

This started as a one-off fix during a real Android TV app build session, where every `./gradlew assembleDebug --no-daemon` call was paying a full cold-start tax for no reason, and every deploy was reinstalling the app for a secondary TV user profile that was never actually used. Enabling the daemon and configuration cache turned a 35-second habit into a 2-second one. Packaged here so the same fix doesn't have to be rediscovered on the next Android TV project.

## License

MIT — do whatever you want with it.
