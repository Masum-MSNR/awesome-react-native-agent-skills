---
name: rn-ci-cd
description: Expert guidance on CI/CD for React Native with EAS Build, GitHub Actions, Fastlane, and EAS Update / CodePush for OTA delivery. Use when asked about builds, store submission, or OTA updates.
---

# React Native CI/CD

## Instructions

Automate three pipelines: **PR checks** (lint, type, unit, E2E smoke), **internal builds** (per-commit on `main`), and **release** (tagged, signed, submitted).

### 1. PR Checks (GitHub Actions)

```yaml
# .github/workflows/pr.yml
name: pr
on:
  pull_request:
    branches: [main]

concurrency:
  group: pr-${{ github.ref }}
  cancel-in-progress: true

jobs:
  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test --ci --coverage

  android-build:
    runs-on: ubuntu-latest
    needs: static
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: cd android && ./gradlew :app:assembleDevDebug --no-daemon
```

### 2. EAS Build for Release

`eas.json`:

```json
{
  "cli": { "version": ">= 10.0.0" },
  "build": {
    "dev":     { "developmentClient": true, "distribution": "internal", "channel": "dev" },
    "staging": { "distribution": "internal", "channel": "staging", "env": { "APP_ENV": "staging" } },
    "prod":    { "channel": "prod", "env": { "APP_ENV": "prod" }, "autoIncrement": true }
  },
  "submit": {
    "prod": {
      "ios": { "appleId": "releases@example.com", "ascAppId": "1234567890", "appleTeamId": "ABCDE12345" },
      "android": { "serviceAccountKeyPath": "./secrets/play-sa.json", "track": "production" }
    }
  }
}
```

Trigger from a tag:

```yaml
# .github/workflows/release.yml
name: release
on:
  push:
    tags: ['v*']

jobs:
  build-and-submit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - uses: expo/expo-github-action@v8
        with: { eas-version: latest, token: ${{ secrets.EXPO_TOKEN }} }
      - run: pnpm install --frozen-lockfile
      - run: eas build --profile prod --platform all --non-interactive
      - run: eas submit --profile prod --platform all --non-interactive
```

### 3. Fastlane for Bare Projects

`ios/fastlane/Fastfile`:

```ruby
default_platform(:ios)

platform :ios do
  desc "Build and upload to TestFlight"
  lane :beta do
    setup_ci if ENV['CI']
    match(type: "appstore", readonly: true)
    increment_build_number(xcodeproj: "MyApp.xcodeproj")
    build_app(scheme: "MyApp-Prod", configuration: "Release", export_method: "app-store")
    upload_to_testflight(skip_waiting_for_build_processing: true)
  end
end
```

`android/fastlane/Fastfile`:

```ruby
default_platform(:android)

platform :android do
  desc "Ship to internal track"
  lane :internal do
    gradle(task: "clean bundleProdRelease")
    upload_to_play_store(track: "internal", aab: "app/build/outputs/bundle/prodRelease/app-prod-release.aab")
  end
end
```

### 4. OTA Updates

Use **EAS Update** (preferred) or **CodePush** to ship JS-only fixes without a store review.

```sh
eas update --branch prod --message "Fix checkout null ref"
```

Channel to branch mapping lives in `eas.json` (`"channel": "prod"`). On app start, `expo-updates` will download and apply on the next launch.

Rules for OTA:

- Never ship native changes via OTA; a mismatched JS bundle will crash.
- Include the bundle version in the update message so rollbacks are traceable.
- Roll out gradually (e.g., branch `prod-canary` first) and promote after bake time.

### 5. Secrets

- Store signing keys and service accounts in the CI secret store (GitHub Encrypted Secrets, EAS Secrets).
- Never check in `.env.prod`, `*.keystore`, `*.p8`, or `play-sa.json`.
- Rotate Apple API keys and Play service accounts at least yearly.

### 6. Versioning

- `version` in `app.json` (Expo) or `Info.plist` / `build.gradle` (bare) is the **user-visible** version.
- `buildNumber` / `versionCode` is incremented automatically by EAS (`"autoIncrement": true`) or Fastlane.
- Tag releases as `vX.Y.Z` and let CI pick the tag name up for release notes.

## Checklist

- [ ] PR CI runs lint, typecheck, unit tests, and a cheap platform build on every PR.
- [ ] Release builds are triggered only by signed tags, never manual dispatches.
- [ ] EAS Build (or Fastlane) handles signing; no signing material is in the repo.
- [ ] `eas submit` (or Fastlane `deliver`) handles store upload; no manual drag-and-drop.
- [ ] OTA updates are JS-only, matched to native binary versions via channels.
- [ ] Secrets are stored in the CI secret manager and rotated on a schedule.
- [ ] Build numbers auto-increment; user-visible version bumps are deliberate.
