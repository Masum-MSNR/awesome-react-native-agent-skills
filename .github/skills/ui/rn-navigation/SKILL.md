---
name: rn-navigation
description: Expert guidance on React Navigation 7 with native-stack, typed routes, deep links, and tab+stack composition. Use when asked about navigation, routing, deep linking, or screen typing.
---

# React Navigation 7 in React Native

## Instructions

Use `@react-navigation/native` v7 with `@react-navigation/native-stack`. Every route must be typed via a shared `ParamList`.

### 1. Typed Route Definitions

```ts
// src/app/navigation/types.ts
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';

export type RootStackParamList = {
  Tabs: undefined;
  Article: { id: string };
  Login: { returnTo?: string } | undefined;
};

export type RootTabsParamList = {
  Home: undefined;
  Search: { query?: string } | undefined;
  Profile: undefined;
};

export type RootStackScreenProps<T extends keyof RootStackParamList> =
  NativeStackScreenProps<RootStackParamList, T>;

export type RootTabsScreenProps<T extends keyof RootTabsParamList> =
  BottomTabScreenProps<RootTabsParamList, T>;

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

Declaring `RootParamList` globally gives `useNavigation()` the correct typing without casts.

### 2. Navigator Composition

```tsx
// src/app/navigation/RootNavigator.tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import type { RootStackParamList, RootTabsParamList } from './types';
import { HomeScreen } from '@features/home/ui/HomeScreen';
import { SearchScreen } from '@features/search/ui/SearchScreen';
import { ProfileScreen } from '@features/profile/ui/ProfileScreen';
import { ArticleScreen } from '@features/articles/ui/ArticleScreen';
import { LoginScreen } from '@features/auth/ui/LoginScreen';

const Stack = createNativeStackNavigator<RootStackParamList>();
const Tabs = createBottomTabNavigator<RootTabsParamList>();

function TabsNavigator() {
  return (
    <Tabs.Navigator screenOptions={{ headerShown: false }}>
      <Tabs.Screen name="Home" component={HomeScreen} />
      <Tabs.Screen name="Search" component={SearchScreen} />
      <Tabs.Screen name="Profile" component={ProfileScreen} />
    </Tabs.Navigator>
  );
}

export function RootNavigator() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Tabs" component={TabsNavigator} options={{ headerShown: false }} />
      <Stack.Screen name="Article" component={ArticleScreen} />
      <Stack.Screen
        name="Login"
        component={LoginScreen}
        options={{ presentation: 'modal' }}
      />
    </Stack.Navigator>
  );
}
```

### 3. Deep Links

```ts
// src/app/navigation/linking.ts
import type { LinkingOptions } from '@react-navigation/native';
import type { RootStackParamList } from './types';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://app.example.com'],
  config: {
    screens: {
      Tabs: {
        screens: {
          Home: 'home',
          Search: 'search/:query?',
          Profile: 'me',
        },
      },
      Article: 'articles/:id',
      Login: 'login',
    },
  },
};
```

Pass `linking` to `NavigationContainer`:

```tsx
<NavigationContainer linking={linking}>
  <RootNavigator />
</NavigationContainer>
```

Register the URL scheme in `ios/<App>/Info.plist` (`CFBundleURLTypes`) and in `android/app/src/main/AndroidManifest.xml` (`<intent-filter>`).

### 4. Screen Component Pattern

```tsx
// src/features/articles/ui/ArticleScreen.tsx
import { View, Text } from 'react-native';
import type { RootStackScreenProps } from '@app/navigation/types';
import { useArticle } from '../api/useArticle';

export function ArticleScreen({ route }: RootStackScreenProps<'Article'>) {
  const { id } = route.params;
  const article = useArticle(id);
  if (!article.data) return null;
  return (
    <View>
      <Text accessibilityRole="header">{article.data.title}</Text>
      <Text>{article.data.body}</Text>
    </View>
  );
}
```

### 5. Imperative Navigation from Non-React Code

```ts
// src/app/navigation/rootNavigation.ts
import { createNavigationContainerRef } from '@react-navigation/native';
import type { RootStackParamList } from './types';

export const navigationRef = createNavigationContainerRef<RootStackParamList>();

export function navigate<T extends keyof RootStackParamList>(
  name: T,
  params: RootStackParamList[T],
) {
  if (navigationRef.isReady()) navigationRef.navigate(name as never, params as never);
}
```

Use this sparingly -- only for push-notification handlers and auth redirects.

## Checklist

- [ ] Every route is typed via `RootStackParamList` / tab param list.
- [ ] `RootParamList` is declared in the global `ReactNavigation` namespace.
- [ ] Deep links are configured in `linking` and registered natively on both platforms.
- [ ] Modal-style screens use `presentation: 'modal'` on native-stack.
- [ ] Screens use `RootStackScreenProps<'Name'>` for `route` and `navigation` typing.
- [ ] Imperative navigation uses a single `navigationRef`, not scattered refs.
