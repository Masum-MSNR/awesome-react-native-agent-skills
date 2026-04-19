# AGENTS Guidelines for This Repository

This repository contains a React Native application project. When working on the project interactively with an AI coding agent, please follow the guidelines below to ensure architectural consistency, maximum performance, and a smooth development experience.

## 1. Project Specifications
- **React Native:** 0.75+ with the New Architecture enabled (Fabric renderer + TurboModules, Hermes + bridgeless mode).
- **TypeScript:** 5.4+ with `strict`, `noUncheckedIndexedAccess`, and `exactOptionalPropertyTypes` enabled.
- **Minimum Platforms:** iOS 15.1+, Android SDK 24+ (minSdkVersion 24, targetSdkVersion 34+).
- **Package Manager:** `pnpm` (preferred) or `yarn` classic. Do not mix lockfiles. Use `npx pod-install` or `bundle exec pod install` for iOS.
- **Expo:** An Expo SDK 51+ managed or bare workflow is an optional but supported path. Prefer Expo config plugins over manual native edits when available.

## 2. Architecture & Design Patterns
We follow a **feature-sliced** folder structure with hooks-first composition:
- **Presentation Layer:** Function components only, hooks for behaviour. No class components. Keep components pure and push side effects into hooks.
- **Domain Layer:** Plain TypeScript modules under `src/domain/` for entities, validators (Zod), and use-cases. No React or React Native imports.
- **Data Layer:** Repository pattern wrapping local sources (`react-native-mmkv`, `@nozbe/watermelondb`, `op-sqlite`) and remote clients (`fetch`, `ky`, or `axios`) behind typed interfaces.
- **State Management:** `Zustand` for lightweight client state; `@reduxjs/toolkit` + RTK Query for teams that already use Redux; `jotai` for atom-granular state.
- **Server State:** `@tanstack/react-query` v5 for all remote data. Never store server data in Zustand/Redux if it is already cacheable via Query.
- **Dependency Injection:** Prefer factory functions and React context providers over service locators.

## 3. Asynchronous Programming
- **Concurrency:** Prefer `async`/`await` with `AbortController` for cancellation. Attach `signal` to `fetch` and TanStack Query `queryFn`.
- **Heavy Work:** Offload CPU-bound work to `react-native-worklets-core` or native TurboModules. Avoid blocking the JS thread with large JSON parsing on startup.
- **Cleanup:** Always cancel subscriptions and abort in-flight requests inside `useEffect` cleanup.

## 4. UI Framework
- **Navigation:** `@react-navigation/native` v7 with `native-stack`. All routes typed via a central `RootStackParamList`. Deep links configured via `linking` config.
- **Styling:** `StyleSheet.create` as the floor; `react-native-unistyles` for themeable multi-variant styling; `nativewind` only when the team already uses Tailwind.
- **Lists:** `@shopify/flash-list` for any list longer than one screen. Always set `estimatedItemSize` and stable `keyExtractor`.
- **Images:** `expo-image` (preferred) or `react-native-fast-image` for caching, placeholders, and priority hints.
- **Animations:** `react-native-reanimated` v3 worklets and `react-native-gesture-handler` v2. `react-native-skia` for canvas work.

## 5. Testing Philosophy
- **Unit Tests:** `jest` + `@testing-library/react-native` + `@testing-library/react-hooks` patterns (via `renderHook`). Use `jest-mock-extended` or hand-rolled mocks for repositories.
- **Component Tests:** Prefer user-visible queries (`getByRole`, `getByLabelText`) over `testID`. Reserve `testID` for E2E selectors.
- **E2E Tests:** `maestro` (preferred for simplicity and CI flake resistance) or `detox` (when deeper native hooks are required).
- **Snapshots:** Use sparingly; prefer explicit assertions on rendered text, roles, and accessibility state.

## 6. Native & Tooling
- **Native Modules:** Prefer Expo config plugins. When direct native code is required, co-locate Kotlin under `android/app/src/main/java/...` and Swift under `ios/<AppName>/...` with clear package boundaries.
- **Metro:** Enable `inlineRequires` and `experimentalImportSupport` for faster startup. Keep `metro.config.js` minimal.
- **Hermes:** Always on. Ship Hermes bytecode in release builds.
- **External Documentation:** When unsure of a React Native API, consult the official [React Native docs](https://reactnative.dev) and [React Navigation docs](https://reactnavigation.org). For packages, check npm for current stable versions and New Architecture compatibility.

## 7. Useful Agent Skills Recap

| Skill Folder                  | Purpose                                                          |
| ----------------------------- | ---------------------------------------------------------------- |
| `architecture/`               | Project structure, state management, and data layer.             |
| `ui/`                         | Components, navigation, images, accessibility, animations, i18n. |
| `performance/`                | Re-render audits, list perf, bundle size.                        |
| `migration/`                  | Class → hooks, old architecture → Fabric + TurboModules.         |
| `testing_and_automation/`     | Jest/RTL unit tests, Maestro/Detox E2E.                          |
| `concurrency_and_networking/` | Async patterns, TanStack Query, offline sync.                    |
| `build_and_tooling/`          | Flavors, schemes, EAS Build, CI/CD.                              |

---

Following these practices ensures that the agent-assisted development workflow stays reliable and consistent. When in doubt, always refer to the specific agent skills provided in `.github/skills/` for deeper task-specific context!

*Note to developers: Update this file whenever the project makes architectural shifts to ensure AI agents stay aligned with your conventions.*
