---
name: rn-performance-audit
description: Expert guidance on auditing React Native performance, including Hermes sampling profiler, re-render detection, and list performance. Use when asked about slow screens, jank, dropped frames, or excessive re-renders.
---

# React Native Performance Audit

## Instructions

Measure before optimizing. The cheapest wins almost always come from fixing re-renders and list configuration, not from native code changes.

### 1. Hermes Sampling Profiler

Hermes ships a built-in sampler. Record a trace from a dev build:

```ts
import { startSamplingProfiler, stopSamplingProfiler } from 'react-native/Libraries/Utilities/HMRClient';
// In dev, connect via Chrome DevTools -> "Performance" tab on the Hermes target.
```

Or via the command line:

```sh
adb shell "run-as com.example.app killall -SIGUSR1 com.example.app"
# Pull /data/data/com.example.app/cache/sampling-profiler-trace-*.cpuprofile
```

Load the `.cpuprofile` in Chrome DevTools. Focus on:

- Functions with > 5% total time.
- Deep stacks inside `commitRoot` / `reconcileChildren` (indicates too much work per commit).
- Long runs of `JSON.parse` or `require` during startup.

### 2. Detecting Re-renders

Enable the React DevTools "Highlight updates when components render" option during manual testing.

Add a tiny hook to log why a component re-rendered:

```ts
import { useRef, useEffect } from 'react';

export function useWhyDidYouRender<T extends Record<string, unknown>>(name: string, props: T) {
  const prev = useRef<T>(props);
  useEffect(() => {
    const changed: Record<string, { from: unknown; to: unknown }> = {};
    for (const k of Object.keys({ ...prev.current, ...props })) {
      if (prev.current[k] !== props[k]) {
        changed[k] = { from: prev.current[k], to: props[k] };
      }
    }
    if (Object.keys(changed).length) {
      console.log(`[render] ${name}`, changed);
    }
    prev.current = props;
  });
}
```

Common fixes:

- Wrap row components with `React.memo` and use stable keys.
- Replace inline callbacks with `useCallback`.
- Replace inline object/array props with `useMemo` or module-level constants.
- Use Zustand selectors or `useSyncExternalStore` so components subscribe only to what they render.

### 3. List Performance

For any `FlashList` or `FlatList`:

```tsx
<FlashList
  data={data}
  keyExtractor={(x) => x.id}
  estimatedItemSize={96}
  drawDistance={400}
  renderItem={renderItem}
  getItemType={(x) => x.variant}
  removeClippedSubviews
/>
```

Checklist:

- `estimatedItemSize` matches the measured row height within ~15%.
- `renderItem` is defined outside the list or wrapped with `useCallback`.
- Row images use `expo-image` with `recyclingKey` (see `rn-images`).
- Heavy row children are deferred with `InteractionManager.runAfterInteractions` or Suspense boundaries.

### 4. Startup Time

- Enable `inlineRequires` in `metro.config.js` so modules load lazily.
- Defer non-critical providers until after the first screen renders.
- Avoid synchronous work (JSON parsing, crypto) in `App.tsx` module scope.
- Measure with `performance.now()` around the root render:

```ts
const t0 = performance.now();
// after first useEffect in App:
console.log('ttr', performance.now() - t0);
```

### 5. Navigation Transitions

- Prefer `native-stack` over `@react-navigation/stack` for 60fps transitions.
- Use `useIsFocused()` to gate expensive work off-screen.
- Avoid mounting large tab screens eagerly; enable `lazy: true` on the tab navigator.

### 6. Animations

See `rn-animations`. A jumpy animation almost always means:

- It is running on the JS thread (missing `useAnimatedStyle`).
- It is animating a layout property.
- It is fighting a re-render storm from a parent.

## Checklist

- [ ] A Hermes CPU profile was captured for the slow flow and analyzed.
- [ ] Row components are `memo`-wrapped and callbacks are stable.
- [ ] `FlashList` has a measured `estimatedItemSize` within ~15% of reality.
- [ ] Inline object/array/function props are hoisted or memoised.
- [ ] Startup has no synchronous heavy work in module scope.
- [ ] Off-screen tabs are lazy; off-screen work is gated by `useIsFocused`.
