---
name: rn-architecture
description: Expert guidance on structuring a React Native 0.75+ app with a feature-sliced layout, New Architecture (Fabric + TurboModules) enabled, and strict TypeScript. Use this when asked about project structure, folder layout, or module boundaries.
---

# React Native Architecture & Feature Slicing

## Instructions

When designing or refactoring a React Native application, prefer a **feature-sliced** folder structure with strict dependency direction: `app` → `features` → `shared` → `domain`. The **New Architecture** (Fabric + TurboModules, Hermes, bridgeless mode) is the default target.

### 1. Top-Level Layout

```
src/
├── app/                # Root providers, navigation container, entry point
│   ├── App.tsx
│   ├── providers/      # QueryClient, ThemeProvider, GestureHandlerRootView
│   └── navigation/     # RootStackParamList, linking config
├── features/           # Self-contained feature slices
│   └── articles/
│       ├── api/        # TanStack Query hooks, repository bindings
│       ├── model/      # Zustand stores, derived selectors
│       ├── ui/         # Screens and feature-scoped components
│       └── index.ts    # Public API of the slice (barrel)
├── shared/             # Cross-cutting: ui kit, hooks, utils
│   ├── ui/
│   ├── hooks/
│   └── lib/
├── domain/             # Pure TS: entities, Zod schemas, use-cases
└── infrastructure/     # Platform adapters: storage, http, logger
```

Rules:

- A `feature` **never** imports from another `feature`. If two features share logic, promote it to `shared/` or `domain/`.
- `domain/` has zero React or React Native imports. Enforce with an ESLint `no-restricted-imports` rule.
- Use TypeScript path aliases (`@app/*`, `@features/*`, `@shared/*`, `@domain/*`) in `tsconfig.json` and mirror them in `babel.config.js` via `babel-plugin-module-resolver`.

### 2. Enable the New Architecture

In `android/gradle.properties`:

```properties
newArchEnabled=true
hermesEnabled=true
```

In `ios/Podfile`:

```ruby
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

Verify at runtime:

```tsx
import { Platform } from 'react-native';

export const isFabric = (): boolean =>
  // @ts-expect-error internal flag
  global?.nativeFabricUIManager != null;

export const archInfo = {
  fabric: isFabric(),
  hermes: !!(globalThis as { HermesInternal?: unknown }).HermesInternal,
  platform: Platform.OS,
} as const;
```

### 3. Typed Entry Point

`src/app/App.tsx`:

```tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { NavigationContainer } from '@react-navigation/native';
import { RootNavigator } from './navigation/RootNavigator';
import { ThemeProvider } from '@shared/ui/theme';

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 30_000, retry: 2 } },
});

export function App(): JSX.Element {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider>
          <NavigationContainer>
            <RootNavigator />
          </NavigationContainer>
        </ThemeProvider>
      </QueryClientProvider>
    </GestureHandlerRootView>
  );
}
```

### 4. Feature Barrels

Each feature exposes a tight public API:

```ts
// src/features/articles/index.ts
export { ArticlesScreen } from './ui/ArticlesScreen';
export { useArticles } from './api/useArticles';
export type { Article } from '@domain/articles';
```

Nothing else from `features/articles/**` should be imported outside the slice.

## Checklist

- [ ] `newArchEnabled=true` and Hermes is on for both platforms.
- [ ] `domain/` contains zero React or React Native imports (enforced by ESLint).
- [ ] Features do not import from sibling features.
- [ ] TypeScript path aliases are declared in both `tsconfig.json` and `babel.config.js`.
- [ ] `tsconfig.json` has `strict`, `noUncheckedIndexedAccess`, and `exactOptionalPropertyTypes` enabled.
- [ ] Each feature exposes a single barrel `index.ts` with only the public API.
