---
name: rn-accessibility
description: Expert guidance on accessibility in React Native, covering accessibilityRole, labels, state, focus, touch targets, and RTL. Use when asked about a11y, screen readers, TalkBack, VoiceOver, or RTL.
---

# React Native Accessibility

## Instructions

Accessibility is non-negotiable. Every interactive element must expose role, label, and state. Every text element must respect dynamic type.

### 1. Roles, Labels, and State

```tsx
import { Pressable, Text } from 'react-native';

export function FollowButton({
  following,
  onToggle,
}: {
  following: boolean;
  onToggle: () => void;
}) {
  return (
    <Pressable
      accessibilityRole="button"
      accessibilityLabel={following ? 'Unfollow author' : 'Follow author'}
      accessibilityState={{ selected: following }}
      accessibilityHint="Double tap to toggle following this author"
      onPress={onToggle}
    >
      <Text>{following ? 'Following' : 'Follow'}</Text>
    </Pressable>
  );
}
```

Rules:

- `accessibilityRole` is required on anything pressable or semantically distinct (`button`, `link`, `header`, `image`, `switch`, `tab`).
- `accessibilityLabel` overrides child text for screen readers. Keep it concise and action-oriented.
- Use `accessibilityState` (`selected`, `checked`, `disabled`, `busy`, `expanded`) rather than baking state into the label.

### 2. Touch Targets

Minimum target size is 44x44 pt (iOS HIG) / 48dp (Material). Use `hitSlop` to expand without changing visual size:

```tsx
<Pressable hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }} />
```

### 3. Grouping and Hiding

- To treat a composite widget as a single announcement: wrap children in a `View` with `accessible` and `accessibilityLabel`.
- To hide decorative elements: `accessibilityElementsHidden` (iOS) and `importantForAccessibility="no-hide-descendants"` (Android).

```tsx
<View
  accessible
  accessibilityLabel={`${title}, ${subtitle}, tap to open`}
  accessibilityRole="button"
>
  <Image source={icon} accessibilityElementsHidden importantForAccessibility="no-hide-descendants" />
  <Text>{title}</Text>
  <Text>{subtitle}</Text>
</View>
```

### 4. Dynamic Type and Contrast

- Do not lock `fontSize` without `allowFontScaling`. Leave the default `allowFontScaling={true}` unless a measured reason forbids it.
- Ensure contrast ratios of at least 4.5:1 for body text and 3:1 for large text. Verify with the Accessibility Inspector (iOS) and Accessibility Scanner (Android).

```tsx
<Text allowFontScaling maxFontSizeMultiplier={1.8}>
  {body}
</Text>
```

### 5. Focus and Reduced Motion

```tsx
import { AccessibilityInfo, findNodeHandle } from 'react-native';
import { useRef, useEffect } from 'react';

export function useAutoFocus<T>(shouldFocus: boolean) {
  const ref = useRef<T>(null);
  useEffect(() => {
    if (!shouldFocus || !ref.current) return;
    const node = findNodeHandle(ref.current as unknown as number);
    if (node) AccessibilityInfo.setAccessibilityFocus(node);
  }, [shouldFocus]);
  return ref;
}
```

Respect reduced motion preferences before animating:

```ts
const reduced = await AccessibilityInfo.isReduceMotionEnabled();
```

### 6. RTL

Enable RTL globally when the user language is RTL:

```ts
import { I18nManager } from 'react-native';
import { getLocales } from 'expo-localization';

const rtl = ['ar', 'he', 'fa', 'ur'].includes(getLocales()[0]?.languageCode ?? '');
if (I18nManager.isRTL !== rtl) {
  I18nManager.allowRTL(rtl);
  I18nManager.forceRTL(rtl);
  // Requires app restart to take effect.
}
```

Use logical styles (`marginStart`, `paddingEnd`) instead of `left`/`right` where possible. Mirror chevrons and directional icons with `transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }]`.

## Checklist

- [ ] Every pressable has `accessibilityRole` and `accessibilityLabel`.
- [ ] Toggles and selections use `accessibilityState`, not label mutations.
- [ ] Touch targets are at least 44x44 / 48dp (use `hitSlop` when needed).
- [ ] Decorative images are hidden from assistive tech.
- [ ] Text scales with the system unless a hard cap is justified.
- [ ] Reduced motion preference is checked before playing non-essential animations.
- [ ] Layout uses `marginStart` / `paddingEnd` and respects `I18nManager.isRTL`.
