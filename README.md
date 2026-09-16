# Android Emulator WebView Updater

An [Agent Skill](https://www.anthropic.com/news/agent-skills) that updates the system WebView on an old Android emulator (AVD) whose built-in version is too outdated to render modern sites — common on API 21-28 images.

It lists your AVDs, asks which one to update, validates the WebView APK you provide against that image (provider allowlist, ABI, SDK range, signature), installs it via `adb install` or a `/system` swap under `-writable-system` depending on what the image allows, and verifies the result by loading a real page and reading the User-Agent.

## Install

In [Claude Code](https://claude.com/claude-code), copy the skill into your skills directory:

```bash
git clone git@github.com:akexorcist/android-emulator-webview-updater-agent-skill.git
cp -r android-emulator-webview-updater-agent-skill/skills/android-emulator-webview-updater ~/.claude/skills/
```

For other Agent Skills-compatible tools, place `skills/android-emulator-webview-updater/SKILL.md` wherever that tool loads skills from.

## Usage

In Claude Code:

```
/android-emulator-webview-updater
```

Or just ask, e.g. "the emulator's webview is outdated" / "update webview on my API 25 emulator". You'll be asked which AVD, and for the path to a WebView APK.

## License

[Apache 2.0](LICENSE)
