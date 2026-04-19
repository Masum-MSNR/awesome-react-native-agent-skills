---
name: rn-flavors
description: Expert guidance on Android product flavors and iOS schemes for multi-environment React Native builds, with a runtime env selector. Use when asked about flavors, schemes, dev/staging/prod, or configuration.
---

# React Native Flavors, Schemes, and Environments

## Instructions

Ship three environments: `dev`, `staging`, `prod`. Each has its own bundle ID / application ID, icon, display name, and API base URL. Never branch behavior on `__DEV__` for environment selection.

### 1. Runtime Env Module

```ts
// src/app/env.ts
const envs = {
  dev: { apiBaseUrl: 'https://dev.api.example.com', sentryDsn: '' },
  staging: { apiBaseUrl: 'https://staging.api.example.com', sentryDsn: 'https://staging@...' },
  prod: { apiBaseUrl: 'https://api.example.com', sentryDsn: 'https://prod@...' },
} as const;

type EnvName = keyof typeof envs;

const name = (process.env.EXPO_PUBLIC_APP_ENV ?? process.env.APP_ENV ?? 'dev') as EnvName;

export const env = { name, ...envs[name] } as const;
```

Inject the value at build time via `.env.<env>` with `react-native-dotenv` / `dotenv`, or via Expo `extra` fields.

### 2. Android Product Flavors

`android/app/build.gradle`:

```gradle
android {
  flavorDimensions "env"
  productFlavors {
    dev {
      dimension "env"
      applicationIdSuffix ".dev"
      resValue "string", "app_name", "MyApp (Dev)"
    }
    staging {
      dimension "env"
      applicationIdSuffix ".staging"
      resValue "string", "app_name", "MyApp (Staging)"
    }
    prod {
      dimension "env"
      resValue "string", "app_name", "MyApp"
    }
  }
}
```

Build:

```sh
./gradlew :app:assembleDevDebug
./gradlew :app:assembleStagingRelease
./gradlew :app:assembleProdRelease
```

Flavor-specific `google-services.json` goes in `android/app/src/<flavor>/google-services.json`.

### 3. iOS Schemes and Configurations

Create three configurations (`Debug.Dev`, `Debug.Staging`, `Debug.Prod` and matching Release) and three schemes. Use an `.xcconfig` file per env:

```
// ios/Config/Dev.xcconfig
PRODUCT_BUNDLE_IDENTIFIER = com.example.app.dev
DISPLAY_NAME = MyApp (Dev)
```

Reference them in `Info.plist`:

```xml
<key>CFBundleDisplayName</key>
<string>$(DISPLAY_NAME)</string>
```

Build via Fastlane or `xcodebuild -scheme MyApp-Staging`.

### 4. Expo Config Plugins

If on Expo, use `app.config.ts`:

```ts
// app.config.ts
import type { ExpoConfig } from 'expo/config';

const env = process.env.APP_ENV ?? 'dev';

const names: Record<string, { name: string; id: string }> = {
  dev: { name: 'MyApp (Dev)', id: 'com.example.app.dev' },
  staging: { name: 'MyApp (Staging)', id: 'com.example.app.staging' },
  prod: { name: 'MyApp', id: 'com.example.app' },
};

const config: ExpoConfig = {
  name: names[env]!.name,
  slug: 'myapp',
  ios: { bundleIdentifier: names[env]!.id },
  android: { package: names[env]!.id },
  extra: { appEnv: env },
};

export default config;
```

Build profiles in `eas.json`:

```json
{
  "build": {
    "dev":     { "env": { "APP_ENV": "dev" },     "distribution": "internal" },
    "staging": { "env": { "APP_ENV": "staging" }, "distribution": "internal" },
    "prod":    { "env": { "APP_ENV": "prod" },    "distribution": "store" }
  }
}
```

### 5. Scripts

`package.json`:

```json
{
  "scripts": {
    "start:dev": "APP_ENV=dev react-native start",
    "android:dev": "APP_ENV=dev react-native run-android --variant=devDebug --appIdSuffix .dev",
    "android:staging": "APP_ENV=staging react-native run-android --variant=stagingRelease --appIdSuffix .staging",
    "ios:dev": "APP_ENV=dev react-native run-ios --scheme MyApp-Dev",
    "ios:prod": "APP_ENV=prod react-native run-ios --scheme MyApp"
  }
}
```

### 6. Icons and Splash Per Env

- Android: put icons in `android/app/src/<flavor>/res/mipmap-*`.
- iOS: add one asset catalog per configuration and reference it in the target build settings.
- Expo: override `ios.icon` / `android.icon` in `app.config.ts` branches.

### 7. Guardrails

- Never display "Dev" builds in the public store. Enforce with CI checks on applicationId/bundleId before submission.
- Log `env.name` on startup (dev + staging only) so QA can confirm the environment.
- Fail fast if `env.apiBaseUrl` is missing; do not fall back silently to another environment.

## Checklist

- [ ] Three distinct applicationIds/bundleIds exist for dev, staging, and prod.
- [ ] Display name and icon differ per environment so QA can tell them apart.
- [ ] `APP_ENV` is set at build time and consumed via a single `env` module.
- [ ] Environment is not branched on `__DEV__` alone.
- [ ] Flavor-specific Firebase / Sentry / API keys live in env-specific files, not checked-in plaintext.
- [ ] CI rejects prod submissions built from non-prod configurations.
