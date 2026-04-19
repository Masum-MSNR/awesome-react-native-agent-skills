---
name: rn-offline-sync
description: Expert guidance on offline-first sync in React Native -- queueing mutations, conflict handling, and background sync. Use when asked about offline write, sync, queues, or conflicts.
---

# React Native Offline Sync

## Instructions

Offline-first means reads resolve from local storage and writes are queued until the network returns. The queue is durable, ordered, and observable.

### 1. Detecting Connectivity

```ts
import NetInfo from '@react-native-community/netinfo';
import { onlineManager } from '@tanstack/react-query';

NetInfo.addEventListener((state) => {
  onlineManager.setOnline(state.isConnected === true && state.isInternetReachable !== false);
});
```

`onlineManager` drives TanStack Query's pause/resume behavior.

### 2. Durable Mutation Queue

```ts
// src/infrastructure/syncQueue.ts
import { MMKV } from 'react-native-mmkv';
import { nanoid } from 'nanoid/non-secure';

const kv = new MMKV({ id: 'sync-queue' });

export type QueuedMutation = {
  id: string;
  kind: 'like' | 'comment' | 'update-profile';
  payload: unknown;
  createdAt: number;
  attempts: number;
};

const KEY = 'queue';

function read(): QueuedMutation[] {
  const raw = kv.getString(KEY);
  return raw ? (JSON.parse(raw) as QueuedMutation[]) : [];
}
function write(q: QueuedMutation[]): void {
  kv.set(KEY, JSON.stringify(q));
}

export const syncQueue = {
  enqueue(m: Omit<QueuedMutation, 'id' | 'createdAt' | 'attempts'>): QueuedMutation {
    const item: QueuedMutation = { id: nanoid(), createdAt: Date.now(), attempts: 0, ...m };
    write([...read(), item]);
    return item;
  },
  peek: () => read(),
  remove(id: string) {
    write(read().filter((m) => m.id !== id));
  },
  bump(id: string) {
    write(read().map((m) => (m.id === id ? { ...m, attempts: m.attempts + 1 } : m)));
  },
};
```

### 3. Drain Worker

```ts
// src/infrastructure/syncWorker.ts
import NetInfo from '@react-native-community/netinfo';
import { syncQueue, type QueuedMutation } from './syncQueue';
import { apply } from './applyMutation';

let running = false;

export async function drain(): Promise<void> {
  if (running) return;
  running = true;
  try {
    for (const m of syncQueue.peek()) {
      const net = await NetInfo.fetch();
      if (!net.isConnected) return;
      try {
        await apply(m);
        syncQueue.remove(m.id);
      } catch (e) {
        syncQueue.bump(m.id);
        if (shouldDrop(m, e)) syncQueue.remove(m.id);
        else break; // stop on first failure to preserve ordering
      }
    }
  } finally {
    running = false;
  }
}

function shouldDrop(m: QueuedMutation, e: unknown): boolean {
  // Drop on permanent 4xx; keep retrying on 5xx / network.
  return e instanceof Error && e.message.startsWith('HTTP 4');
}

NetInfo.addEventListener((s) => {
  if (s.isConnected) void drain();
});
```

### 4. Optimistic Local Writes

```ts
// src/features/articles/api/useOptimisticLike.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { syncQueue } from '@infrastructure/syncQueue';
import { drain } from '@infrastructure/syncWorker';
import { articleKeys } from './useArticles';
import type { Article } from '@domain/articles';

export function useOptimisticLike() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async ({ id, liked }: { id: string; liked: boolean }) => {
      syncQueue.enqueue({ kind: 'like', payload: { id, liked } });
      void drain();
    },
    onMutate: ({ id, liked }) => {
      const prev = qc.getQueryData<Article>(articleKeys.detail(id));
      if (prev) qc.setQueryData<Article>(articleKeys.detail(id), { ...prev, liked });
      return { prev };
    },
    onError: (_e, { id }, ctx) => {
      if (ctx?.prev) qc.setQueryData(articleKeys.detail(id), ctx.prev);
    },
  });
}
```

### 5. Conflict Resolution

Three strategies, pick one per entity:

- **Last-write-wins**: trivial, safe for leaf data (likes, read flags).
- **Server-wins**: on 409, replace local with server value and keep a "discarded changes" toast.
- **Merge**: field-level diff; only practical when the entity is a CRDT-ish structure (counter, set).

```ts
async function applyLike(m: QueuedMutation): Promise<void> {
  const { id, liked } = m.payload as { id: string; liked: boolean };
  const res = await fetch(`/articles/${id}/like`, { method: liked ? 'POST' : 'DELETE' });
  if (res.status === 409) {
    // Server wins: refetch and discard local change.
    await fetch(`/articles/${id}`); // triggers TanStack Query invalidation on the way back
    return;
  }
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
}
```

### 6. Background Sync (Advanced)

For periodic drains without the app open:

- **iOS**: `expo-background-fetch` or `react-native-background-fetch`.
- **Android**: `expo-background-fetch` (wraps WorkManager) or a bare `react-native-background-task` via WorkManager.

Both have OS-imposed minimum intervals (15 minutes on Android, opportunistic on iOS).

## Checklist

- [ ] Connectivity drives TanStack Query's `onlineManager`.
- [ ] Mutation queue is durable (MMKV) and preserves order.
- [ ] Drain worker runs single-flight, resumes on reconnect, and preserves ordering on failure.
- [ ] 4xx drops the item; 5xx / network retries with backoff.
- [ ] Conflict strategy is documented per entity (LWW / server-wins / merge).
- [ ] Optimistic updates always have a rollback path on failure.
- [ ] Background sync (if used) respects OS minimum intervals and exits cleanly on budget.
