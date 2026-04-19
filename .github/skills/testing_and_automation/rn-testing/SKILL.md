---
name: rn-testing
description: Expert guidance on unit and component testing in React Native with Jest and React Native Testing Library, including hook tests, mocks, and snapshot discipline. Use when asked about unit tests, component tests, or mocks.
---

# React Native Testing with Jest + RNTL

## Instructions

Use `jest` with `jest-preset-react-native` (or `jest-expo`) and `@testing-library/react-native`. Test behavior, not implementation.

### 1. Jest Config

`jest.config.js`:

```js
module.exports = {
  preset: 'react-native',
  setupFiles: ['./jest.setup.ts'],
  setupFilesAfterEach: ['@testing-library/react-native/extend-expect'],
  transformIgnorePatterns: [
    'node_modules/(?!(?:react-native|@react-native|@react-navigation|expo|expo-.*|@shopify/flash-list|react-native-reanimated)/)',
  ],
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json'],
  testMatch: ['**/*.(spec|test).(ts|tsx)'],
};
```

`jest.setup.ts`:

```ts
import 'react-native-gesture-handler/jestSetup';

jest.mock('react-native-reanimated', () => require('react-native-reanimated/mock'));

jest.mock('react-native-mmkv', () => {
  const store = new Map<string, string>();
  return {
    MMKV: class {
      getString = (k: string) => store.get(k);
      set = (k: string, v: string) => void store.set(k, v);
      delete = (k: string) => void store.delete(k);
    },
  };
});
```

### 2. Component Tests -- Query by User-Visible Attributes

```tsx
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Button } from '@shared/ui/Button';

describe('Button', () => {
  it('invokes onPress when enabled', () => {
    const onPress = jest.fn();
    render(<Button label="Save" onPress={onPress} />);
    fireEvent.press(screen.getByRole('button', { name: 'Save' }));
    expect(onPress).toHaveBeenCalledTimes(1);
  });

  it('is not pressable when disabled', () => {
    const onPress = jest.fn();
    render(<Button label="Save" onPress={onPress} disabled />);
    const btn = screen.getByRole('button', { name: 'Save' });
    expect(btn).toBeDisabled();
    fireEvent.press(btn);
    expect(onPress).not.toHaveBeenCalled();
  });
});
```

Prefer role/name queries. Avoid `testID` except for E2E selectors.

### 3. Testing Hooks

```ts
// useCounter.test.ts
import { renderHook, act } from '@testing-library/react-native';
import { useCounter } from './useCounter';

it('increments', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.inc());
  expect(result.current.count).toBe(1);
});
```

### 4. Testing with TanStack Query

```tsx
// useArticles.test.tsx
import { renderHook, waitFor } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useArticles } from './useArticles';

function createWrapper() {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false, gcTime: 0 } } });
  return function Wrapper({ children }: { children: React.ReactNode }) {
    return <QueryClientProvider client={qc}>{children}</QueryClientProvider>;
  };
}

it('loads articles', async () => {
  globalThis.fetch = jest.fn(async () =>
    new Response(JSON.stringify([{ id: '1', title: 't', body: 'b', updatedAt: 1 }]), { status: 200 }),
  ) as unknown as typeof fetch;

  const { result } = renderHook(() => useArticles(), { wrapper: createWrapper() });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toHaveLength(1);
});
```

### 5. Mocking Repositories

Prefer real in-memory fakes over `jest.fn()` chains:

```ts
// src/infrastructure/articleRepository.fake.ts
import type { ArticleRepository } from '@domain/articles/ArticleRepository';
import type { Article } from '@domain/articles/Article';

export function createFakeArticleRepo(seed: Article[] = []): ArticleRepository {
  const store = new Map<string, Article>(seed.map((a) => [a.id, a]));
  return {
    list: async () => [...store.values()],
    get: async (id) => store.get(id) ?? null,
    save: async (a) => void store.set(a.id, a),
  };
}
```

Injected fakes read as production code, not test-shaped mocks.

### 6. Snapshot Discipline

- Use snapshots for **stable structural output** only (theme tokens, generated config).
- Never snapshot screens -- they change often and the diffs are unreadable.
- Always review diffs manually before accepting an updated snapshot.

### 7. CI

- Run with `--ci --runInBand --coverage`.
- Fail if coverage drops under a project-wide threshold in `jest.config.js`:

```js
coverageThreshold: {
  global: { branches: 70, functions: 75, lines: 80, statements: 80 },
}
```

## Checklist

- [ ] Components are tested via role / label queries, not `testID` or DOM internals.
- [ ] Hooks use `renderHook` + `act` for updates.
- [ ] TanStack Query tests use a per-test `QueryClient` with `retry: false`.
- [ ] Reanimated and MMKV are mocked once in `jest.setup.ts`.
- [ ] Repository dependencies have in-memory fakes, not ad-hoc `jest.fn()` chains.
- [ ] Snapshots are used sparingly and reviewed on every update.
- [ ] CI enforces a coverage threshold.
