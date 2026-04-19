---
name: rn-localization
description: Expert guidance on i18n in React Native with i18next or Lingui, ICU plurals, dynamic locale switching, and RTL layout mirroring. Use when asked about translations, plurals, locale, or RTL.
---

# React Native Localization & i18n

## Instructions

Use `i18next` + `react-i18next` as the default stack. `@lingui/react` is an acceptable alternative when the team prefers macro-based message extraction.

### 1. Setup

```ts
// src/app/i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import { getLocales } from 'expo-localization';
import en from '@assets/i18n/en.json';
import fr from '@assets/i18n/fr.json';
import ar from '@assets/i18n/ar.json';

const fallback = 'en';
const device = getLocales()[0]?.languageCode ?? fallback;

void i18n.use(initReactI18next).init({
  resources: { en: { translation: en }, fr: { translation: fr }, ar: { translation: ar } },
  lng: device,
  fallbackLng: fallback,
  interpolation: { escapeValue: false },
  returnNull: false,
  compatibilityJSON: 'v4',
});

export default i18n;
```

`compatibilityJSON: 'v4'` enables native ICU plural handling.

### 2. Using Translations

```tsx
// src/features/articles/ui/ArticleCount.tsx
import { Text } from 'react-native';
import { useTranslation } from 'react-i18next';

export function ArticleCount({ n }: { n: number }) {
  const { t } = useTranslation();
  return <Text>{t('articles.count', { count: n })}</Text>;
}
```

`en.json`:

```json
{
  "articles": {
    "count_one": "{{count}} article",
    "count_other": "{{count}} articles"
  }
}
```

`ar.json` (uses six plural categories automatically):

```json
{
  "articles": {
    "count_zero": "لا توجد مقالات",
    "count_one": "مقالة واحدة",
    "count_two": "مقالتان",
    "count_few": "{{count}} مقالات",
    "count_many": "{{count}} مقالة",
    "count_other": "{{count}} مقالة"
  }
}
```

### 3. Dynamic Locale Switching

```ts
// src/shared/i18n/useLocale.ts
import { useCallback } from 'react';
import i18n from '@app/i18n';
import { I18nManager } from 'react-native';
import { kv } from '@infrastructure/storage/mmkv';

const RTL = new Set(['ar', 'he', 'fa', 'ur']);

export function useLocale() {
  const change = useCallback(async (lng: string) => {
    await i18n.changeLanguage(lng);
    kv.set('locale', lng);
    const rtl = RTL.has(lng);
    if (I18nManager.isRTL !== rtl) {
      I18nManager.allowRTL(rtl);
      I18nManager.forceRTL(rtl);
      // Requires app restart (RNRestart or Expo Updates.reloadAsync) to take effect.
    }
  }, []);
  return { current: i18n.language, change };
}
```

### 4. Formatting Numbers, Dates, and Currencies

Use the platform `Intl` APIs -- Hermes 0.75 ships ICU:

```ts
export const formatCurrency = (n: number, locale: string, currency: string) =>
  new Intl.NumberFormat(locale, { style: 'currency', currency }).format(n);

export const formatRelative = (d: Date, locale: string) => {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
  const diffMin = Math.round((d.getTime() - Date.now()) / 60_000);
  return rtf.format(diffMin, 'minute');
};
```

### 5. RTL Layout

- Prefer logical styles: `marginStart`, `paddingEnd`, `textAlign: 'left'` (which becomes logical start).
- Mirror directional glyphs manually:

```tsx
import { I18nManager, Image } from 'react-native';
import chevron from '@assets/chevron.png';

<Image
  source={chevron}
  style={{ transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }] }}
/>;
```

- Test both directions in CI with a visual regression tool (see `rn-e2e-testing`).

### 6. Extraction and CI

- Keep translation keys in flat namespaced files under `src/assets/i18n/<lng>.json`.
- Fail CI if keys diverge between languages (a small Node script comparing key sets).
- Never concatenate translated strings -- use interpolation.

## Checklist

- [ ] i18n resources are split per-language under `assets/i18n/<lng>.json`.
- [ ] ICU plural keys (`_one`, `_other`, plus `_zero`/`_two`/`_few`/`_many` where relevant) are used.
- [ ] Locale changes persist and reload the app to apply RTL flips.
- [ ] Layouts use logical styles (`marginStart`, `paddingEnd`) rather than `left`/`right`.
- [ ] Number, date, and currency formatting use `Intl` APIs, not ad-hoc strings.
- [ ] CI guards against missing keys across languages.
