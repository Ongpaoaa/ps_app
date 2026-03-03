# AGENTS.md

Instructions for AI coding agents working in this repository.

## Versioning Rule

- On every shipped app update, bump `expo.version` in `app.json`.
- Do not hardcode version text in UI. Read it from Expo config (`Constants.expoConfig.version`).
- Keep the Profile screen footer version label present and accurate.
