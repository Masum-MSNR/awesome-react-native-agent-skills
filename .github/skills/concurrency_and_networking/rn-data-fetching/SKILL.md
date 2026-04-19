---
name: rn-data-fetching
description: Expert guidance on TanStack Query v5 in React Native -- queries, mutations, optimistic updates, polling, retries, and offline behavior. Use when asked about data fetching, caching, or server state.
---

# React Native Data Fetching with TanStack Query

## Instructions

All server state flows through `@tanstack/react-query` v5. Components never call `fetch` directly -- they call a typed hook that wraps `useQuery` / `useMutation`.

### 1. Client Setup

```ts
// src/app/queryClient.ts
import { QueryClient, focusManager } from '@tanstack/react-query';
import { AppState } from 'react-native';
import type { AppStateStatus } from 'react-native';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      gcTime: 5 * 60_000,
      retry: (failureCount, err) => {
        if (err instanceof Error && err.name === 'AbortError') return false;
        return failureCount < 2;
      },
      refetchOnReconnect: 'always',
    },
    mutations: { retry: 0 },
  },
});

AppState.addEventListener('change', (s: AppStateStatus) => {
  focusManager.setFocused(s === 'active');
});
```

### 2. Typed Query Hook

```ts
// src/features/articles/api/useArticles.ts
import { useQuery } from '@tanstack/react-query';
import { getJson } from '@shared/lib/http';
import { ArticleSchema } from '@domain/articles';
import { z } from 'zod';

const ListSchema = z.array(ArticleSchema);

export const articleKeys = {
  all: ['articles'] as const,
  list: (q?: string) => ['articles', 'list', q ?? ''] as const,
  detail: (id: string) => ['articles', 'detail', id] as const,
};

export function useArticles(query?: string) {
  return useQuery({
    queryKey: articleKeys.list(query),
    queryFn: async ({ signal }) => {
      const raw = await getJson<unknown>(`/articles?q=${encodeURIComponent(query ?? '')}`, signal);
      return ListSchema.parse(raw);
    },
    staleTime: 60_000,
  });
}
```

### 3. Mutation with Optimistic Update

```ts
// src/features/articles/api/useLikeArticle.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { articleKeys } from './useArticles';
import type { Article } from '@domain/articles';

export function useLikeArticle() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async ({ id, liked }: { id: string; liked: boolean }) => {
      const res = await fetch(`/articles/${id}/like`, {
        method: liked ? 'POST' : 'DELETE',
      });
      if (!res.ok) throw new Error('like failed');
    },
    onMutate: async ({ id, liked }) => {
      await qc.cancelQueries({ queryKey: articleKeys.detail(id) });
      const prev = qc.getQueryData<Article>(articleKeys.detail(id));
      if (prev) qc.setQueryData<Article>(articleKeys.detail(id), { ...prev, liked });
      return { prev };
    },
    onError: (_err, { id }, ctx) => {
      if (ctx?.prev) qc.setQueryData(articleKeys.detail(id), ctx.prev);
    },
    onSettled: (_data, _err, { id }) => {
      void qc.invalidateQueries({ queryKey: articleKeys.detail(id) });
    },
  });
}
```

### 4. Infinite Queries (Paged Lists)

```ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { getJson } from '@shared/lib/http';

type Page = { items: readonly Article[]; nextCursor: string | null };

export function useInfiniteArticles() {
  return useInfiniteQuery({
    queryKey: ['articles', 'infinite'],
    initialPageParam: null as string | null,
    queryFn: ({ pageParam, signal }) =>
      getJson<Page>(`/articles?cursor=${pageParam ?? ''}`, signal),
    getNextPageParam: (last) => last.nextCursor,
  });
}
```

Then in the screen:

```tsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteArticles();
const flat = (data?.pages ?? []).flatMap((p) => p.items);

<FlashList
  data={flat}
  onEndReached={() => hasNextPage && !isFetchingNextPage && fetchNextPage()}
  onEndReachedThreshold={0.6}
  estimatedItemSize={96}
  renderItem={({ item }) => <ArticleRow article={item} />}
  keyExtractor={(a) => a.id}
/>;
```

### 5. Polling and Window Focus

```ts
useQuery({
  queryKey: ['stats'],
  queryFn: getStats,
  refetchInterval: 30_000,
  refetchIntervalInBackground: false,
});
```

### 6. Persistence

Persist the cache for offline resume:

```ts
// src/app/persistQuery.ts
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import { kv } from '@infrastructure/storage/mmkv';
import { queryClient } from './queryClient';

const persister = createSyncStoragePersister({
  storage: {
    getItem: (k) => kv.getString(k) ?? null,
    setItem: (k, v) => kv.set(k, v),
    removeItem: (k) => kv.delete(k),
  },
});

void persistQueryClient({ queryClient, persister, maxAge: 24 * 60 * 60_000 });
```

## Checklist

- [ ] All server data is accessed through typed `useQuery` / `useMutation` hooks.
- [ ] Query keys are centralized in a `*Keys` object per feature.
- [ ] `queryFn` receives and forwards the `signal` for cancellation.
- [ ] Mutations use `onMutate` + rollback for optimistic updates.
- [ ] Responses are validated with Zod (or equivalent) at the boundary.
- [ ] The cache is persisted to MMKV so offline resume is instant.
- [ ] `focusManager` is wired to `AppState` so focus refetches work on mobile.
