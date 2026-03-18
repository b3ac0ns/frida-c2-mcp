# Changelog

All notable changes to this project are documented here.

## [Unreleased]

## [0.0.2] — 2026-03-14

### Added

- **iOS support** (`ios.py`): Frida MCP server for rootless jailbroken iOS (palera1n, `/var/jb/`). Uses `frida-compile` and `frida-objc-bridge`; precompiled Python 3.12 `.deb` and native wheels in `ios_precompiled/`.
- **Documentation**: `setup.md` (Android + iOS), `MANIFEST.md`, Gemini CLI MCP registration, requirements files, known issues in README, FastMCP 2.x pinning notes.

### Fixed

- **`get_session_messages`**: No longer returned empty — fixed `on_message` handling of non-dict payloads, added `console.log` capture for one-shot results and `agent:log` routing for persistent scripts (Android + iOS).
- **One-shot script execution (Android)**: Resolved futures on `agent:result` exposed a latent bug — `Java.perform({ onComplete, onError })` is incompatible with esbuild-bundled `frida-java-bridge`; switched to `Java.perform(function() { ... })`.
- **One-shot / agent flow (iOS)**: Same message-pipeline fixes so success vs error and console output surface correctly.

### Notes

- Android workflow unchanged in spirit: Termux, esbuild, manual Frida Python build.
- FastMCP remains pinned to **below 3.0** on both platforms until migration.
