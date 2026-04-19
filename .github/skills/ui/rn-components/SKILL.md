---
name: rn-components
description: Expert guidance on building composable React Native components with StyleSheet, Unistyles, or NativeWind, and using FlashList for performant lists. Use when asked about components, styling, or list rendering.
---

# React Native Components & Styling

## Instructions

Write **function components only**. Keep them small, pure, and presentational. Push state and effects into hooks.

### 1. Component Shape

```tsx
// src/shared/ui/Button.tsx
import { memo } from 'react';
import { Pressable, Text, StyleSheet } from 'react-native';
import type { PressableProps } from 'react-native';

type Variant = 'primary' | 'secondary';

type ButtonProps = Omit<PressableProps, 'children'> & {
  label: string;
  variant?: Variant;
};

export const Button = memo(function Button({
  label,
  variant = 'primary',
  disabled,
  ...rest
}: ButtonProps) {
  return (
    <Pressable
      accessibilityRole="button"
      accessibilityState={{ disabled: !!disabled }}
      disabled={disabled}
      style={({ pressed }) => [
        styles.base,
        styles[variant],
        pressed && styles.pressed,
        disabled && styles.disabled,
      ]}
      {...rest}
    >
      <Text style={[styles.label, variant === 'secondary' && styles.labelSecondary]}>
        {label}
      </Text>
    </Pressable>
  );
});

const styles = StyleSheet.create({
  base: { paddingHorizontal: 16, paddingVertical: 12, borderRadius: 12, alignItems: 'center' },
  primary: { backgroundColor: '#1f6feb' },
  secondary: { backgroundColor: 'transparent', borderWidth: 1, borderColor: '#1f6feb' },
  pressed: { opacity: 0.85 },
  disabled: { opacity: 0.4 },
  label: { color: '#ffffff', fontWeight: '600', fontSize: 16 },
  labelSecondary: { color: '#1f6feb' },
});
```

Rules:

- Use `memo` only when props are stable and the component is rendered many times.
- Accept a `style` prop via `StyleProp<ViewStyle>` to keep components composable.
- Keep every styling decision in `StyleSheet.create` or a theme-aware wrapper. No inline object literals in JSX.

### 2. Theming with Unistyles

```ts
// src/shared/ui/theme/unistyles.ts
import { UnistylesRegistry } from 'react-native-unistyles';

const light = { colors: { bg: '#ffffff', text: '#0b1220', accent: '#1f6feb' } } as const;
const dark = { colors: { bg: '#0b1220', text: '#e6edf3', accent: '#58a6ff' } } as const;

type AppThemes = { light: typeof light; dark: typeof dark };
type AppBreakpoints = { sm: 0; md: 600; lg: 900 };

declare module 'react-native-unistyles' {
  export interface UnistylesThemes extends AppThemes {}
  export interface UnistylesBreakpoints extends AppBreakpoints {}
}

UnistylesRegistry.addThemes({ light, dark })
  .addBreakpoints({ sm: 0, md: 600, lg: 900 })
  .addConfig({ adaptiveThemes: true });
```

### 3. NativeWind (Tailwind) Alternative

Use only when the team already writes Tailwind. Configure `tailwind.config.js`, add the babel plugin, and style via `className`:

```tsx
import { View, Text } from 'react-native';

export function Card({ title }: { title: string }) {
  return (
    <View className="rounded-2xl bg-white p-4 shadow">
      <Text className="text-base font-semibold text-slate-900">{title}</Text>
    </View>
  );
}
```

### 4. Lists with FlashList

Always prefer `@shopify/flash-list` over `FlatList` for lists longer than one screen.

```tsx
import { FlashList } from '@shopify/flash-list';
import { ArticleRow } from './ArticleRow';
import type { Article } from '@domain/articles';

export function ArticleList({ data }: { data: readonly Article[] }) {
  return (
    <FlashList
      data={data}
      keyExtractor={(a) => a.id}
      estimatedItemSize={96}
      renderItem={({ item }) => <ArticleRow article={item} />}
      getItemType={(a) => (a.body.length > 400 ? 'long' : 'short')}
      drawDistance={250}
    />
  );
}
```

Guidance:

- Provide `estimatedItemSize` based on the measured row height.
- Use `getItemType` to let FlashList recycle cells of different layouts.
- Memoise row components (`memo` + stable callbacks with `useCallback`).

## Checklist

- [ ] No class components anywhere in the codebase.
- [ ] No inline style objects in JSX; all styles come from `StyleSheet.create` or theme hooks.
- [ ] Lists longer than one screen use `FlashList` with `estimatedItemSize` and stable `keyExtractor`.
- [ ] Interactive components expose `accessibilityRole` and `accessibilityState`.
- [ ] Row components are memoised and callbacks are wrapped in `useCallback`.
