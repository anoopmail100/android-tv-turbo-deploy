# turbo-gradle-build

A [Claude Code](https://claude.com/claude-code) skill that speeds up the edit → rebuild → reinstall loop on Android/Gradle projects by fixing the usual cause of slow local builds: Gradle restarting cold on every invocation instead of reusing a warm daemon and its configuration cache.

Measured on a small single-module Android app in a real session: a fully-cached rebuild dropped from **~35s** to **~2s**, and a real rebuild after an actual code change dropped from ~35s to **~17s**. `adb install` also got faster by dropping a redundant `--user all` flag.

## What it does

- Stops passing `--no-daemon` so the Gradle daemon stays warm between builds
- Enables Gradle's build cache, configuration cache, and parallel execution via `gradle.properties`
- Trims unnecessary `adb install` flags when only one device user profile is active
- Tells you to verify the win with `time ./gradlew assembleDebug` rather than just trusting it

See [`turbo-gradle-build/SKILL.md`](turbo-gradle-build/SKILL.md) for the full technique and the exact properties to add.

## Installing this skill

Claude Code skills live as a folder containing a `SKILL.md` with YAML frontmatter (`name`, `description`). To use this one:

1. Copy the `turbo-gradle-build/` folder into your skills directory, e.g.:
   ```bash
   cp -r turbo-gradle-build ~/.claude/skills/turbo-gradle-build
   ```
   (or your project's `.claude/skills/` directory for a project-scoped skill)
2. Start a Claude Code session in your Android/Gradle project. Claude will pick up the skill automatically when build speed comes up, or you can ask for it directly: *"use the turbo-gradle-build skill to speed up my builds"*.

## Why this exists

This started as a one-off fix during a real Android TV app build session, where every `./gradlew assembleDebug --no-daemon` call was paying a full cold-start tax for no reason. Enabling the daemon and configuration cache turned a 35-second habit into a 2-second one. Packaged here so the same fix doesn't have to be rediscovered on the next project.

## License

MIT — do whatever you want with it.
