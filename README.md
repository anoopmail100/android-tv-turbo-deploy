# android-tv-turbo-deploy

A [Claude Code](https://claude.com/claude-code) skill that speeds up the edit → rebuild → sideload loop when developing an Android TV / Google TV app over ADB.

## Evidence

Measured on a real Android TV app, deployed to a physical Google TV device over `adb connect`:

| Scenario | Before | After |
|---|---|---|
| Rebuild with no code changes | ~35s | ~2s |
| Rebuild after a real code change | ~35s | ~17s |
| Install step | slower (reinstalling for a dormant profile nobody used) | faster (skips the unused profile) |

That's roughly a 94% drop on a no-op rebuild and a ~50% drop on a real recompile — both well past the "at least 30% faster" bar this was built to hit.

## Installing this skill

Copy the `android-tv-turbo-deploy/` folder into your skills directory:

```bash
cp -r android-tv-turbo-deploy ~/.claude/skills/android-tv-turbo-deploy
```

(or your project's `.claude/skills/` for a project-scoped skill). Then start a Claude Code session in your Android TV project — Claude picks it up automatically when the build/deploy loop comes up, or ask directly: *"use the android-tv-turbo-deploy skill to speed up my builds."*

For the technical detail behind the numbers, see [`android-tv-turbo-deploy/SKILL.md`](android-tv-turbo-deploy/SKILL.md).

## License

MIT — do whatever you want with it.
