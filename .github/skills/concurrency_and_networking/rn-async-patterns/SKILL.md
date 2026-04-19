---
name: rn-async-patterns
description: Expert guidance on Promises, async/await, cancellation with AbortController, timeouts, and retries in React Native / TypeScript. Use when asked about async code, cancellation, timeouts, or retries.
---

# React Native Async Patterns

## Instructions

Always use `async`/`await` with typed errors and `AbortController` for cancellation. Never leak promises whose resolution outlives the component.

### 1. Cancellable Fetch

```ts
// src/shared/lib/http.ts
export class HttpError extends Error {
  constructor(public readonly status: number, public readonly url: string, msg: string) {
    super(msg);
    this.name = 'HttpError';
  }
}

export async function getJson<T>(url: string, signal?: AbortSignal): Promise<T> {
  const res = await fetch(url, { signal });
  if (!res.ok) throw new HttpError(res.status, url, `HTTP ${res.status}`);
  return (await res.json()) as T;
}
```

### 2. Effect + AbortController

```tsx
import { useEffect, useState } from 'react';
import { getJson } from '@shared/lib/http';
import type { Article } from '@domain/articles';

export function useLoadArticle(id: string) {
  const [data, setData] = useState<Article | null>(null);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const ac = new AbortController();
    getJson<Article>(`/articles/${id}`, ac.signal)
      .then(setData)
      .catch((e: unknown) => {
        if (ac.signal.aborted) return;
        setError(e instanceof Error ? e : new Error(String(e)));
      });
    return () => ac.abort();
  }, [id]);

  return { data, error };
}
```

### 3. Timeout

```ts
export function withTimeout<T>(p: Promise<T>, ms: number, ac?: AbortController): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    const t = setTimeout(() => {
      ac?.abort();
      reject(new Error(`timeout after ${ms}ms`));
    }, ms);
    p.then(
      (v) => {
        clearTimeout(t);
        resolve(v);
      },
      (e) => {
        clearTimeout(t);
        reject(e);
      },
    );
  });
}
```

Usage:

```ts
const ac = new AbortController();
const data = await withTimeout(getJson<Article>('/articles/1', ac.signal), 5_000, ac);
```

### 4. Retry with Backoff

```ts
export async function retry<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  opts: { attempts?: number; baseMs?: number; signal?: AbortSignal } = {},
): Promise<T> {
  const { attempts = 3, baseMs = 300, signal } = opts;
  let lastErr: unknown;
  for (let i = 0; i < attempts; i++) {
    if (signal?.aborted) throw new Error('aborted');
    try {
      return await fn(signal ?? new AbortController().signal);
    } catch (e) {
      lastErr = e;
      const delay = baseMs * 2 ** i + Math.random() * baseMs;
      await new Promise((r) => setTimeout(r, delay));
    }
  }
  throw lastErr;
}
```

Do not retry on 4xx responses (client errors). Add a predicate argument if needed:

```ts
const shouldRetry = (e: unknown) =>
  e instanceof HttpError ? e.status >= 500 : true;
```

### 5. Parallelism Control

For a bounded batch, use `Promise.all` with chunking:

```ts
export async function mapLimit<I, O>(
  items: readonly I[],
  limit: number,
  fn: (item: I, index: number) => Promise<O>,
): Promise<O[]> {
  const out: O[] = new Array(items.length);
  let i = 0;
  async function worker() {
    while (i < items.length) {
      const idx = i++;
      out[idx] = await fn(items[idx]!, idx);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return out;
}
```

### 6. Never Do This

- Fire-and-forget `.then()` chains in effects without cancellation.
- `setState` inside an async function after unmount. Always guard on `signal.aborted`.
- `async` functions passed directly to `useEffect` -- the return must be a cleanup function, not a Promise. Declare the async function inside.
- Swallow abort errors -- check `signal.aborted` or `err.name === 'AbortError'` and return.

## Checklist

- [ ] Every network call accepts and forwards an `AbortSignal`.
- [ ] Effects that start async work return a cleanup that aborts it.
- [ ] Timeouts use a combined pattern that also aborts the underlying request.
- [ ] Retries use exponential backoff with jitter and skip 4xx responses.
- [ ] Bounded concurrency uses `mapLimit` or `p-limit`; no unbounded `Promise.all` over user input.
- [ ] Abort errors are recognised and not reported as real failures.
