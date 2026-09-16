# Android Emulator WebView Updater

An [Agent Skill](https://www.anthropic.com/news/agent-skills) that updates the system WebView on an old Android emulator (AVD) whose built-in version is too outdated to render modern sites — common on API 21-28 images.

It lists your AVDs, asks which one to update, validates the WebView APK you provide against that image (provider allowlist, ABI, SDK range, signature), installs it via `adb install` or a `/system` swap under `-writable-system` depending on what the image allows, and verifies the result by loading a real page and reading the User-Agent.

## Install

Copy `skills/android-emulator-webview-updater` into your agent's skills directory, or tell your agent to add it from this repo.

## Usage

Ask your agent to update the WebView on an emulator, e.g. "the emulator's webview is outdated" / "update webview on my API 25 emulator". You'll be asked which AVD, and for the path to a WebView APK.

## License

[Apache 2.0](LICENSE)
