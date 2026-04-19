---
name: rn-animations
description: Expert guidance on animations in React Native using Reanimated 3 worklets, Gesture Handler, Skia, and Moti. Use when asked about animations, gestures, transitions, or canvas drawing.
---

# React Native Animations

## Instructions

Use `react-native-reanimated` v3 as the default. Prefer **worklets** running on the UI thread over JS-driven `Animated`. Combine with `react-native-gesture-handler` v2 for gestures.

### 1. Shared Values and Derived Styles

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  Easing,
} from 'react-native-reanimated';
import { Pressable } from 'react-native';

export function ScaleCard({ children }: { children: React.ReactNode }) {
  const scale = useSharedValue(1);
  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));

  return (
    <Pressable
      onPressIn={() => (scale.value = withSpring(0.96, { damping: 14 }))}
      onPressOut={() => (scale.value = withTiming(1, { duration: 180, easing: Easing.out(Easing.cubic) }))}
    >
      <Animated.View style={style}>{children}</Animated.View>
    </Pressable>
  );
}
```

### 2. Gestures with the New API

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withDecay } from 'react-native-reanimated';

export function DraggableBox() {
  const tx = useSharedValue(0);
  const ty = useSharedValue(0);
  const startX = useSharedValue(0);
  const startY = useSharedValue(0);

  const pan = Gesture.Pan()
    .onStart(() => {
      startX.value = tx.value;
      startY.value = ty.value;
    })
    .onUpdate((e) => {
      tx.value = startX.value + e.translationX;
      ty.value = startY.value + e.translationY;
    })
    .onEnd((e) => {
      tx.value = withDecay({ velocity: e.velocityX, clamp: [-200, 200] });
      ty.value = withDecay({ velocity: e.velocityY, clamp: [-400, 400] });
    });

  const style = useAnimatedStyle(() => ({
    transform: [{ translateX: tx.value }, { translateY: ty.value }],
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[{ width: 100, height: 100, backgroundColor: '#1f6feb' }, style]} />
    </GestureDetector>
  );
}
```

### 3. Entering / Exiting Layout Animations

```tsx
import Animated, { FadeIn, FadeOut, LinearTransition } from 'react-native-reanimated';

<Animated.View
  entering={FadeIn.duration(200)}
  exiting={FadeOut.duration(150)}
  layout={LinearTransition.springify()}
/>;
```

### 4. Moti for Declarative Animations

For simple declarative cases, `moti` can be more readable than shared values:

```tsx
import { MotiView } from 'moti';

<MotiView
  from={{ opacity: 0, translateY: 12 }}
  animate={{ opacity: 1, translateY: 0 }}
  transition={{ type: 'timing', duration: 220 }}
/>;
```

### 5. Skia for Canvas

Use `@shopify/react-native-skia` when you need custom drawing, shaders, or 2D graphics:

```tsx
import { Canvas, Circle, Group, useClockValue, useComputedValue } from '@shopify/react-native-skia';

export function Pulse() {
  const clock = useClockValue();
  const r = useComputedValue(() => 40 + Math.sin(clock.current / 200) * 8, [clock]);
  return (
    <Canvas style={{ width: 120, height: 120 }}>
      <Group>
        <Circle cx={60} cy={60} r={r} color="#1f6feb" />
      </Group>
    </Canvas>
  );
}
```

### 6. Rules of Thumb

- Never call `setState` inside a worklet. Use `runOnJS` for the rare crossing.
- Never animate layout props (`width`, `height`, `flex`) on the JS thread; prefer `transform` and `opacity`.
- Wrap list item animations with `Animated.FlatList` / FlashList-compatible wrappers so layout stays stable.
- Respect reduced-motion (see `rn-accessibility`).

## Checklist

- [ ] Animations run on the UI thread via Reanimated worklets.
- [ ] Gestures use `Gesture.*` API with `GestureDetector`, not the deprecated `PanGestureHandler` props API.
- [ ] `transform` and `opacity` are preferred over layout-affecting properties.
- [ ] `runOnJS` is only used when crossing back to the JS thread is genuinely required.
- [ ] Reduced-motion preference disables or shortens non-essential animations.
- [ ] Skia is used for canvas-style custom drawing; not for simple view animation.
