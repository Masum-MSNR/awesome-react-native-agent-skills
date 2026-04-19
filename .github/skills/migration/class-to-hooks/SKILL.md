---
name: class-to-hooks
description: Guidance on migrating legacy React Native class components to function components with hooks. Use when asked to modernize, refactor, or remove class components.
---

# Migrating Class Components to Hooks

## Instructions

All new components must be function components. When modernizing a class component, apply a predictable mapping from lifecycle methods to hooks.

### 1. Lifecycle Mapping

| Class method | Hook replacement |
|--------------|------------------|
| `constructor` + `state` | `useState` / `useReducer` |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate(prevProps, prevState)` | `useEffect(() => { ... }, [deps])` |
| `componentWillUnmount` | Cleanup function returned from `useEffect` |
| `shouldComponentUpdate` | `React.memo` + stable props, or `useMemo` |
| `getDerivedStateFromProps` | Derive during render; avoid state duplication |
| `componentDidCatch` | `react-error-boundary` or a small class wrapper (one of the rare remaining class uses) |

### 2. Before: Class Component

```tsx
import { Component } from 'react';
import { View, Text, ActivityIndicator } from 'react-native';
import { fetchArticle } from '@infrastructure/api';
import type { Article } from '@domain/articles';

type Props = { id: string };
type State = { article: Article | null; error: string | null; loading: boolean };

export class ArticleScreen extends Component<Props, State> {
  state: State = { article: null, error: null, loading: true };
  private aborter = new AbortController();

  componentDidMount() {
    this.load();
  }

  componentDidUpdate(prev: Props) {
    if (prev.id !== this.props.id) {
      this.aborter.abort();
      this.aborter = new AbortController();
      this.setState({ loading: true, article: null, error: null });
      this.load();
    }
  }

  componentWillUnmount() {
    this.aborter.abort();
  }

  private async load() {
    try {
      const article = await fetchArticle(this.props.id, this.aborter.signal);
      this.setState({ article, loading: false });
    } catch (e) {
      this.setState({ error: String(e), loading: false });
    }
  }

  render() {
    if (this.state.loading) return <ActivityIndicator />;
    if (this.state.error) return <Text>{this.state.error}</Text>;
    return (
      <View>
        <Text>{this.state.article?.title}</Text>
      </View>
    );
  }
}
```

### 3. After: Function Component + Hooks

```tsx
import { useEffect, useState } from 'react';
import { View, Text, ActivityIndicator } from 'react-native';
import { fetchArticle } from '@infrastructure/api';
import type { Article } from '@domain/articles';

export function ArticleScreen({ id }: { id: string }) {
  const [article, setArticle] = useState<Article | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const ac = new AbortController();
    setLoading(true);
    setError(null);
    setArticle(null);
    fetchArticle(id, ac.signal)
      .then(setArticle)
      .catch((e: unknown) => !ac.signal.aborted && setError(String(e)))
      .finally(() => !ac.signal.aborted && setLoading(false));
    return () => ac.abort();
  }, [id]);

  if (loading) return <ActivityIndicator />;
  if (error) return <Text>{error}</Text>;
  return (
    <View>
      <Text>{article?.title}</Text>
    </View>
  );
}
```

### 4. Even Better: Delegate to TanStack Query

Once you have removed the class, replace hand-rolled fetching with a reusable hook (see `rn-data-fetching`):

```tsx
import { useQuery } from '@tanstack/react-query';
import { fetchArticle } from '@infrastructure/api';

export function useArticle(id: string) {
  return useQuery({
    queryKey: ['article', id],
    queryFn: ({ signal }) => fetchArticle(id, signal),
  });
}
```

### 5. Common Pitfalls

- **Stale closures**: any value referenced inside `useEffect`, `useCallback`, or `useMemo` must be in the dependency array or derived from a ref. Enable `react-hooks/exhaustive-deps`.
- **Setting state after unmount**: always gate `setState` on `!signal.aborted` or use TanStack Query which handles it.
- **`getDerivedStateFromProps`**: 95% of cases should become derived values computed during render, not new state.
- **`ref` forwarding**: use `forwardRef` + `useImperativeHandle` only when you need to expose imperative methods.

### 6. Error Boundaries

Error boundaries still require a class. Wrap once and reuse:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallbackRender={({ error }) => <Text>{error.message}</Text>}>
  <ArticleScreen id={id} />
</ErrorBoundary>;
```

## Checklist

- [ ] No new class components in the codebase.
- [ ] Every migrated `useEffect` has a dependency array and a cleanup function when needed.
- [ ] In-flight requests are cancelled with `AbortController` on unmount and prop change.
- [ ] Derived values are computed during render, not mirrored into state.
- [ ] `react-hooks/exhaustive-deps` lint rule is enabled and clean.
- [ ] Error boundaries are provided via `react-error-boundary`, not ad-hoc `componentDidCatch` classes.
