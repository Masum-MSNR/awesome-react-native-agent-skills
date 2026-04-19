---
name: rn-images
description: Expert guidance on efficient image loading in React Native with expo-image or react-native-fast-image, including caching, placeholders, priority, and memory hygiene. Use when asked about images, avatars, or media lists.
---

# React Native Images

## Instructions

Prefer `expo-image` (works in bare and managed workflows). Fall back to `react-native-fast-image` only on projects that cannot bump to a recent Expo SDK.

### 1. Basic Remote Image

```tsx
import { Image } from 'expo-image';

const BLURHASH =
  'L6PZfSi_.AyE_3t7t7R**0o#DgR4';

export function Avatar({ uri, size = 48 }: { uri: string; size?: number }) {
  return (
    <Image
      source={{ uri }}
      style={{ width: size, height: size, borderRadius: size / 2 }}
      placeholder={{ blurhash: BLURHASH }}
      contentFit="cover"
      transition={200}
      cachePolicy="memory-disk"
      priority="high"
      recyclingKey={uri}
      accessibilityIgnoresInvertColors
    />
  );
}
```

Key props:

- `cachePolicy` -- `memory-disk` for most remote images; `none` for sensitive content.
- `priority` -- `high` for above-the-fold, `low` for off-screen list items.
- `recyclingKey` -- required for lists; tells the view when to re-decode.
- `contentFit` -- the modern replacement for `resizeMode`.

### 2. Lists with Many Images

Pair with FlashList and give it a sensible `drawDistance`:

```tsx
import { FlashList } from '@shopify/flash-list';
import { Image } from 'expo-image';

type Row = { id: string; thumb: string; title: string };

export function Gallery({ data }: { data: readonly Row[] }) {
  return (
    <FlashList
      data={data}
      estimatedItemSize={120}
      keyExtractor={(r) => r.id}
      drawDistance={400}
      renderItem={({ item }) => (
        <Image
          source={{ uri: item.thumb }}
          style={{ width: 120, height: 120 }}
          cachePolicy="memory-disk"
          priority="normal"
          recyclingKey={item.id}
        />
      )}
    />
  );
}
```

### 3. Prefetching

```ts
import { Image } from 'expo-image';

export async function prefetchArticles(urls: readonly string[]): Promise<void> {
  await Image.prefetch([...urls], 'memory-disk');
}
```

Call `prefetchArticles` when you know the user is about to see a screen (e.g., on hover of a list item or when a query resolves).

### 4. Fixed Asset Sizing

Always give local assets an explicit `width` and `height`. Undefined dimensions force a synchronous measure pass.

```tsx
import logo from '@assets/logo.png';

<Image source={logo} style={{ width: 96, height: 32 }} contentFit="contain" />;
```

For vector art, use `react-native-svg` instead of PNGs and set `width`/`height` on the `<Svg>` root.

### 5. Memory Hygiene

- Never load full-resolution images in list rows. Request resized URLs from the backend (`?w=256`) or use a CDN transform.
- Clear the memory cache on background:

```ts
import { Image } from 'expo-image';
import { AppState } from 'react-native';

AppState.addEventListener('change', (s) => {
  if (s === 'background') Image.clearMemoryCache();
});
```

- On OOM-prone Android devices, set `cachePolicy="disk"` for large photo grids.

## Checklist

- [ ] All remote images use `expo-image` (or a deliberate fallback) with `cachePolicy`, `priority`, and `recyclingKey`.
- [ ] List images receive a server- or CDN-resized URL, not full resolution.
- [ ] Local assets have explicit `width` and `height`.
- [ ] Placeholders use `blurhash`, `thumbhash`, or a low-res source.
- [ ] Memory cache is cleared on `background` for image-heavy screens.
- [ ] Above-the-fold images use `priority="high"`; off-screen ones do not.
