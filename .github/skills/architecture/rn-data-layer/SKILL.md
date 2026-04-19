---
name: rn-data-layer
description: Expert guidance on building an offline-first data layer in React Native with MMKV, WatermelonDB, or SQLite behind a repository abstraction. Use when asked about persistence, caching, or offline support.
---

# React Native Data Layer & Offline-First

## Instructions

Data flows through a **repository** interface defined in `domain/`. Screens and hooks never touch storage clients directly.

### 1. Pick the Right Store

| Need | Use |
|------|-----|
| Small key-value (settings, tokens) | `react-native-mmkv` |
| Structured rows, moderate size | `op-sqlite` (New Architecture) |
| Rich reactive models, sync | `@nozbe/watermelondb` |
| Files / blobs | `react-native-fs` / `expo-file-system` |

Avoid `@react-native-async-storage/async-storage` for hot paths -- it is serialized and slow.

### 2. MMKV Adapter

```ts
// src/infrastructure/storage/mmkv.ts
import { MMKV } from 'react-native-mmkv';
import type { StateStorage } from 'zustand/middleware';

export const kv = new MMKV({ id: 'app' });

export const mmkvStorage: StateStorage = {
  getItem: (k) => kv.getString(k) ?? null,
  setItem: (k, v) => kv.set(k, v),
  removeItem: (k) => kv.delete(k),
};
```

### 3. Repository Interface

```ts
// src/domain/articles/ArticleRepository.ts
import type { Article } from './Article';

export interface ArticleRepository {
  list(opts?: { signal?: AbortSignal }): Promise<Article[]>;
  get(id: string): Promise<Article | null>;
  save(article: Article): Promise<void>;
}
```

### 4. SQLite Implementation with `op-sqlite`

```ts
// src/infrastructure/articleRepository.sqlite.ts
import { open } from '@op-engineering/op-sqlite';
import type { ArticleRepository } from '@domain/articles/ArticleRepository';
import type { Article } from '@domain/articles/Article';

const db = open({ name: 'app.db' });

db.execute(`
  CREATE TABLE IF NOT EXISTS articles (
    id TEXT PRIMARY KEY NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    updated_at INTEGER NOT NULL
  )
`);

export const sqliteArticleRepository: ArticleRepository = {
  async list() {
    const { rows } = await db.executeAsync('SELECT * FROM articles ORDER BY updated_at DESC');
    return (rows?._array ?? []) as Article[];
  },
  async get(id) {
    const { rows } = await db.executeAsync('SELECT * FROM articles WHERE id = ?', [id]);
    return (rows?._array?.[0] as Article | undefined) ?? null;
  },
  async save(a) {
    await db.executeAsync(
      'INSERT OR REPLACE INTO articles (id, title, body, updated_at) VALUES (?, ?, ?, ?)',
      [a.id, a.title, a.body, a.updatedAt],
    );
  },
};
```

### 5. Offline-First Read-Through Cache

```ts
// src/features/articles/api/useArticles.ts
import { useQuery } from '@tanstack/react-query';
import { sqliteArticleRepository as local } from '@infrastructure/articleRepository.sqlite';
import { httpArticleRepository as remote } from '@infrastructure/articleRepository.http';

export function useArticles() {
  return useQuery({
    queryKey: ['articles'],
    queryFn: async ({ signal }) => {
      const cached = await local.list();
      try {
        const fresh = await remote.list({ signal });
        await Promise.all(fresh.map(local.save));
        return fresh;
      } catch {
        if (cached.length) return cached;
        throw new Error('offline and no cache');
      }
    },
    staleTime: 30_000,
  });
}
```

### 6. Validate at Boundaries

Validate every remote payload with Zod before it enters the domain:

```ts
import { z } from 'zod';

export const ArticleSchema = z.object({
  id: z.string(),
  title: z.string().min(1),
  body: z.string(),
  updatedAt: z.number().int(),
});
export type Article = z.infer<typeof ArticleSchema>;
```

## Checklist

- [ ] All persistence is accessed through a repository interface in `domain/`.
- [ ] Remote payloads are validated with Zod (or equivalent) before entering the cache.
- [ ] Hot-path key-value reads use MMKV, not AsyncStorage.
- [ ] SQLite / WatermelonDB migrations are versioned and idempotent.
- [ ] Repositories accept an `AbortSignal` and forward it to `fetch`.
- [ ] Offline reads return cached data when the network fails; errors bubble only when nothing is cached.
