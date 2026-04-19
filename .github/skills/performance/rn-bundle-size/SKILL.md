---
name: rn-bundle-size
description: Expert guidance on shrinking React Native bundle size with Hermes bytecode, Metro config, tree shaking, and inline requires. Use when asked about bundle size, startup, or Metro configuration.
---

# React Native Bundle Size

## Instructions

A smaller bundle parses faster and starts faster. Target the JS bundle first -- native code changes are a distant second.

### 1. Hermes Bytecode

Hermes compiles JS to bytecode at build time, cutting parse cost and reducing app size.

- `android/gradle.properties`: `hermesEnabled=true`
- `ios/Podfile`: `:hermes_enabled => true` (default in modern templates)

Verify at runtime:

```ts
export const isHermes = !!(globalThis as { HermesInternal?: unknown }).HermesInternal;
```

### 2. Metro Config

`metro.config.js`:

```js
const { getDefaultConfig } = require('expo/metro-config'); // or '@react-native/metro-config'

const config = getDefaultConfig(__dirname);

config.transformer = {
  ...config.transformer,
  getTransformOptions: async () => ({
    transform: {
      experimentalImportSupport: true,
      inlineRequires: true,
    },
  }),
  minifierConfig: {
    keep_classnames: false,
    keep_fnames: false,
    mangle: { keep_classnames: false, keep_fnames: false },
    compress: { drop_console: false },
  },
};

config.resolver = {
  ...config.resolver,
  unstable_enablePackageExports: true,
};

module.exports = config;
```

`inlineRequires` defers `require` calls until first use -- large cold-start wins. `experimentalImportSupport` enables ESM semantics so Metro can drop unused named exports.

### 3. Tree Shaking Friendly Imports

Never pull a whole icon or util library:

```ts
// Bad -- pulls the whole package
import _ from 'lodash';
_.debounce(fn, 200);

// Good -- only pulls the one function
import debounce from 'lodash/debounce';
debounce(fn, 200);
```

For icon sets, prefer `@expo/vector-icons/<FontName>` sub-paths or `react-native-svg` with individually imported SVGs.

### 4. Source Maps and Analysis

Generate a bundle and analyze it:

```sh
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output dist/android.bundle \
  --sourcemap-output dist/android.map

npx react-native-bundle-visualizer --platform android
```

Look for:

- Duplicated copies of `react`, `scheduler`, or any polyfill (indicates misconfigured resolver).
- Full `moment` or `lodash` builds (replace with `date-fns` / `dayjs` / per-function imports).
- Large asset bundles -- move images to remote CDN or OTA delivery where possible.

### 5. Native Size

- Android: enable R8 (`android.enableR8=true`) and split APKs by ABI:

  ```gradle
  android {
    splits {
      abi {
        enable true
        reset()
        include 'arm64-v8a', 'armeabi-v7a', 'x86_64'
        universalApk false
      }
    }
  }
  ```

- iOS: enable bitcode-free thinning (`ENABLE_BITCODE = NO` is now the default), strip debug symbols from release, and compress `Assets.car` by using an asset catalog.

### 6. Shipping Deltas

- Use `expo-updates` (EAS Update) or CodePush to ship JS-only fixes without a store review.
- Keep native modules out of hot-fix cycles; updates that change native code require a full build.

### 7. Dev-Only Code

Guard dev-only tooling so it never lands in release:

```ts
if (__DEV__) {
  require('./devtools/reactotron');
}
```

## Checklist

- [ ] Hermes is enabled on both platforms.
- [ ] `inlineRequires` and `experimentalImportSupport` are on in `metro.config.js`.
- [ ] No full-library imports of `lodash`, `moment`, or icon packs.
- [ ] A bundle visualizer report has been captured and inspected.
- [ ] Android release builds use R8 and ABI splits; iOS release strips debug symbols.
- [ ] Dev-only tools (Reactotron, DevMenu helpers) are guarded by `__DEV__`.
