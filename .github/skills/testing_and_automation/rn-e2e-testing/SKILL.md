---
name: rn-e2e-testing
description: Expert guidance on end-to-end testing React Native apps with Maestro or Detox, including CI setup and flake reduction. Use when asked about E2E, UI automation, Maestro, or Detox.
---

# React Native E2E Testing: Maestro and Detox

## Instructions

Pick **Maestro** by default: it is flow-based, requires no native instrumentation, and survives RN upgrades. Pick **Detox** when you need deep native hooks (e.g., controlling timers, intercepting system dialogs).

### 1. Maestro Flow

Install the CLI (`curl -Ls "https://get.maestro.mobile.dev" | bash`) and write flows in YAML.

`.maestro/login.yaml`:

```yaml
appId: com.example.app
---
- launchApp:
    clearState: true
- assertVisible: "Sign in"
- tapOn:
    id: "email-input"
- inputText: "qa@example.com"
- tapOn:
    id: "password-input"
- inputText: "correct-horse"
- tapOn: "Sign in"
- assertVisible:
    text: "Welcome"
    timeout: 5000
```

Run locally:

```sh
maestro test .maestro/login.yaml
```

Record an interactive run:

```sh
maestro studio
```

### 2. TestIDs for Selectors

Mark the small set of elements you need to target:

```tsx
<TextInput testID="email-input" accessibilityLabel="Email" onChangeText={setEmail} />
```

Keep IDs stable, kebab-case, and scoped (`login.email-input`).

### 3. Maestro in CI (GitHub Actions)

```yaml
# .github/workflows/e2e.yml
name: e2e
on: [pull_request]

jobs:
  android:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          arch: x86_64
          script: |
            curl -Ls "https://get.maestro.mobile.dev" | bash
            export PATH="$PATH:$HOME/.maestro/bin"
            ./gradlew :app:assembleDebug :app:assembleAndroidTest
            adb install -r android/app/build/outputs/apk/debug/app-debug.apk
            maestro test .maestro/
```

### 4. Detox (When You Need It)

`package.json`:

```json
{
  "detox": {
    "testRunner": { "args": { "config": "e2e/jest.config.js" } },
    "configurations": {
      "ios.sim.debug": {
        "device": { "type": "iPhone 15" },
        "app": "ios.debug",
        "type": "ios.simulator"
      }
    },
    "apps": {
      "ios.debug": {
        "type": "ios.app",
        "binaryPath": "ios/build/Build/Products/Debug-iphonesimulator/App.app",
        "build": "xcodebuild -workspace ios/App.xcworkspace -scheme App -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build"
      }
    }
  }
}
```

`e2e/login.e2e.ts`:

```ts
import { device, element, by, expect } from 'detox';

describe('login', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true, delete: true });
  });

  it('signs the user in', async () => {
    await expect(element(by.text('Sign in'))).toBeVisible();
    await element(by.id('email-input')).typeText('qa@example.com');
    await element(by.id('password-input')).typeText('correct-horse');
    await element(by.text('Sign in')).tap();
    await expect(element(by.text('Welcome'))).toBeVisible();
  });
});
```

### 5. Reducing Flake

- **Seed data via the app's debug menu or a dev-only deep link**, never via production API calls.
- **Disable animations in E2E builds**:

  ```ts
  // src/app/env.ts
  import { I18nManager } from 'react-native';
  if (process.env.E2E === '1') {
    // Ask Reanimated / Animated to finish instantly
    require('react-native').LogBox?.ignoreAllLogs?.(true);
  }
  ```

- **Network**: mock external services at the app boundary or route traffic through a local Mirage/MSW server so tests are deterministic.
- **Wait for the right thing**: always await a visible piece of UI, never use `sleep` / `waitFor(3000)`.
- **One flow per file**: makes retries and reporting precise.

### 6. Snapshots & Screenshots

- Maestro: `- takeScreenshot: login-success` stores a PNG per step; attach to CI artifacts.
- Detox: `device.takeScreenshot('login-success')`.
- For visual regression, pair with `react-native-owl` or ship PNGs to a service like Percy.

## Checklist

- [ ] Default stack is Maestro; Detox is justified by specific native needs.
- [ ] Selectors use stable, scoped `testID`s.
- [ ] Test data is seeded via debug menu or deep link, not prod API calls.
- [ ] Animations and long timers are disabled in the E2E build.
- [ ] CI installs a prebuilt APK/IPA; flows never build from source during the test stage.
- [ ] Every flow ends by asserting a visible post-condition, not a fixed sleep.
- [ ] Screenshots are archived on failure for triage.
