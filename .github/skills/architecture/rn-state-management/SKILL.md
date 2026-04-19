---
name: rn-state-management
description: Expert guidance on choosing and using Zustand, Redux Toolkit, or Jotai for client state and TanStack Query for server state in React Native 0.75+. Use when asked about state management, stores, or server caching.
---

# React Native State Management

## Instructions

Split state into two kinds: **client state** (UI, session, local forms) and **server state** (remote data). Never conflate them.

- Client state -> `zustand` (default), `@reduxjs/toolkit` for large teams already on Redux, or `jotai` for atom-granular state.
- Server state -> `@tanstack/react-query` v5.

### 1. Zustand (Preferred Default)

```ts
// src/features/cart/model/cartStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { mmkvStorage } from '@infrastructure/storage/mmkv';

type CartItem = { id: string; qty: number };

type CartState = {
  items: CartItem[];
  add: (id: string) => void;
  remove: (id: string) => void;
  total: () => number;
};

export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],
      add: (id) =>
        set((s) => {
          const existing = s.items.find((i) => i.id === id);
          return existing
            ? { items: s.items.map((i) => (i.id === id ? { ...i, qty: i.qty + 1 } : i)) }
            : { items: [...s.items, { id, qty: 1 }] };
        }),
      remove: (id) => set((s) => ({ items: s.items.filter((i) => i.id !== id) })),
      total: () => get().items.reduce((n, i) => n + i.qty, 0),
    }),
    { name: 'cart', storage: createJSONStorage(() => mmkvStorage) },
  ),
);
```

Use **selectors** at call sites to avoid over-rendering:

```tsx
const count = useCartStore((s) => s.items.length);
```

### 2. Redux Toolkit + RTK Query

```ts
// src/app/store.ts
import { configureStore } from '@reduxjs/toolkit';
import { setupListeners } from '@reduxjs/toolkit/query';
import { articlesApi } from '@features/articles/api/articlesApi';

export const store = configureStore({
  reducer: { [articlesApi.reducerPath]: articlesApi.reducer },
  middleware: (gDM) => gDM().concat(articlesApi.middleware),
});
setupListeners(store.dispatch);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```ts
// src/features/articles/api/articlesApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import type { Article } from '@domain/articles';

export const articlesApi = createApi({
  reducerPath: 'articlesApi',
  baseQuery: fetchBaseQuery({ baseUrl: 'https://api.example.com' }),
  tagTypes: ['Article'],
  endpoints: (b) => ({
    list: b.query<Article[], void>({ query: () => '/articles', providesTags: ['Article'] }),
  }),
});

export const { useListQuery } = articlesApi;
```

### 3. TanStack Query for Server State

```tsx
// src/features/articles/api/useArticles.ts
import { useQuery } from '@tanstack/react-query';
import { articleRepository } from '@infrastructure/articleRepository';

export function useArticles() {
  return useQuery({
    queryKey: ['articles'],
    queryFn: ({ signal }) => articleRepository.list({ signal }),
    staleTime: 60_000,
  });
}
```

Use `useMutation` with `onMutate` + `queryClient.setQueryData` for optimistic updates (see `rn-data-fetching`).

### 4. When to Pick Which

| Scenario | Pick |
|----------|------|
| Small to medium app, few global stores | Zustand |
| Large org already on Redux, wants RTK Query | Redux Toolkit |
| Many independent atoms, derived values | Jotai |
| Anything coming from an HTTP/GraphQL API | TanStack Query |

## Checklist

- [ ] Server data lives in TanStack Query (or RTK Query) cache, not in Zustand/Redux.
- [ ] Every Zustand consumer uses a **selector**, not `useStore()` without one.
- [ ] Persisted stores use MMKV-backed storage, not `AsyncStorage` for hot paths.
- [ ] Mutations use `onMutate` + rollback for optimistic updates.
- [ ] No prop drilling deeper than two levels without considering a store or context.
