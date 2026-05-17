# React Coding Challenges — Interview Implementatsiyalar

> 20 ta MAJBURIY challenge: custom hooks, HOCs, patterns, R19 features. Har biri uchun: signature, implementation, usage, edge cases, variations, tests.

---

## Mundarija

**Custom Hooks (1-7):**
1. [useDebounce](#1-usedebounce-middle)
2. [useThrottle](#2-usethrottle-middle)
3. [usePrevious](#3-useprevious-middle)
4. [useLocalStorage](#4-uselocalstorage-middle)
5. [useIntersectionObserver](#5-useintersectionobserver-middle)
6. [useFetch](#6-usefetch-middle)
7. [useEventListener](#7-useeventlistener-middle)

**HOCs va Class Patterns (8-10):**
8. [HOC: withAuth](#8-hoc-withauth-middle)
9. [HOC: withErrorBoundary](#9-hoc-witherrorboundary-middle)
10. [ErrorBoundary class component](#10-errorboundary-class-middle)

**Architectural Patterns (11-12):**
11. [Compound Component pattern](#11-compound-component-senior)
12. [useReducer + Context global state](#12-usereducer--context-middle)

**Performance va Library Patterns (13-17):**
13. [React.memo + useCallback optimize](#13-memo--usecallback-middle)
14. [Virtualized list (windowing)](#14-virtualized-list-senior)
15. [useAsync — Promise hook](#15-useasync-middle)
16. [Suspense + lazy code splitting](#16-suspense--lazy-middle)
17. [Render Props pattern](#17-render-props-middle)

**React 19 Features (18-20):**
18. [R19 use() data fetching](#18-r19-use-data-middle)
19. [useOptimistic optimistic UI](#19-useoptimistic-senior)
20. [Custom forwardRef + useImperativeHandle](#20-forwardref--useimperativehandle-middle)

---

## 1. `useDebounce` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function useDebounce<T>(value: T, delay: number): T;
```

### Description

Returns a debounced version of `value` — only updates after `delay` ms of inactivity.

### Implementation

```tsx
import { useState, useEffect } from "react";

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

### Usage

```tsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      fetchResults(debouncedQuery);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// User types "hello" rapidly:
// query updates: "h", "he", "hel", "hell", "hello"
// debouncedQuery: stays "" until 300ms after last keystroke → "hello"
```

### Edge Cases

- **Initial render**: `debouncedValue = value` (no delay on first render)
- **Rapid changes**: Each change resets timer — last value after delay wins
- **Component unmount**: Cleanup clears pending timer (no memory leak)
- **Delay change**: Timer restarts with new delay

### Variations

**With cancel function:**

```tsx
function useDebounce<T>(value: T, delay: number): {
  debouncedValue: T;
  cancel: () => void;
} {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    timerRef.current = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, [value, delay]);

  const cancel = useCallback(() => {
    if (timerRef.current) clearTimeout(timerRef.current);
  }, []);

  return { debouncedValue, cancel };
}
```

**Debounced callback:**

```tsx
function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number,
): T {
  const callbackRef = useRef(callback);
  callbackRef.current = callback;

  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  return useCallback(((...args: Parameters<T>) => {
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      callbackRef.current(...args);
    }, delay);
  }) as T, [delay]);
}

// Usage:
function Component() {
  const debouncedSearch = useDebouncedCallback((q: string) => {
    fetch(`/api?q=${q}`);
  }, 300);

  return <input onChange={(e) => debouncedSearch(e.target.value)} />;
}
```

### Tests

```tsx
import { renderHook, act } from "@testing-library/react";

test("debounces value updates", () => {
  jest.useFakeTimers();

  const { result, rerender } = renderHook(
    ({ value }) => useDebounce(value, 300),
    { initialProps: { value: "initial" } }
  );

  expect(result.current).toBe("initial");

  rerender({ value: "updated" });
  expect(result.current).toBe("initial"); // Not yet

  act(() => {
    jest.advanceTimersByTime(300);
  });

  expect(result.current).toBe("updated");
});
```

### Common Mistakes

- ❌ **`setTimeout` ID in `useState`**: Causes re-render — use `useRef`.
- ❌ **Missing cleanup**: Pending timers fire after unmount → state update on unmounted component.
- ❌ **Stale callback**: Callback closes over stale state — use ref pattern (`callbackRef`).

</details>

---

## 2. `useThrottle` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function useThrottle<T>(value: T, interval: number): T;
```

### Description

Returns throttled value — updates at most once per `interval` ms.

### Implementation

```tsx
import { useState, useEffect, useRef } from "react";

function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState<T>(value);
  const lastExecutedRef = useRef<number>(Date.now());

  useEffect(() => {
    const now = Date.now();
    const timeSinceLastExecution = now - lastExecutedRef.current;

    if (timeSinceLastExecution >= interval) {
      // Execute immediately
      lastExecutedRef.current = now;
      setThrottledValue(value);
    } else {
      // Schedule for the remainder of the interval
      const timer = setTimeout(() => {
        lastExecutedRef.current = Date.now();
        setThrottledValue(value);
      }, interval - timeSinceLastExecution);

      return () => clearTimeout(timer);
    }
  }, [value, interval]);

  return throttledValue;
}
```

### Usage

```tsx
function ScrollPosition() {
  const [scrollY, setScrollY] = useState(0);
  const throttledScrollY = useThrottle(scrollY, 100);

  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    window.addEventListener("scroll", handler);
    return () => window.removeEventListener("scroll", handler);
  }, []);

  return <p>Scroll Y: {throttledScrollY}</p>;
  // Updates at most once per 100ms
}
```

### Throttle vs Debounce

| Aspect | Debounce | Throttle |
|--------|----------|----------|
| Strategy | Delay after last change | Limit rate |
| Updates | One after activity stops | At fixed intervals |
| Use case | Search input | Scroll, resize, mouse move |

```
Input changes: |||||||||
Debounce(300):                     ↓
Throttle(300): ↓     ↓     ↓     ↓
```

### Edge Cases

- **First update**: Executes immediately (no delay).
- **Rapid changes**: Last value within interval wins (replaces queued).
- **Long idle**: Next change executes immediately.

### Variations

**Throttled callback:**

```tsx
function useThrottledCallback<T extends (...args: any[]) => any>(
  callback: T,
  interval: number,
): T {
  const callbackRef = useRef(callback);
  callbackRef.current = callback;

  const lastExecutedRef = useRef(0);
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  return useCallback(((...args: Parameters<T>) => {
    const now = Date.now();
    const timeSinceLastExecution = now - lastExecutedRef.current;

    if (timeSinceLastExecution >= interval) {
      lastExecutedRef.current = now;
      callbackRef.current(...args);
    } else {
      if (timerRef.current) clearTimeout(timerRef.current);
      timerRef.current = setTimeout(() => {
        lastExecutedRef.current = Date.now();
        callbackRef.current(...args);
      }, interval - timeSinceLastExecution);
    }
  }) as T, [interval]);
}
```

### Common Mistakes

- ❌ **Using `setInterval`**: Doesn't reset on new value — wrong semantics.
- ❌ **Trailing call missing**: Without scheduling — last value lost.
- ❌ **Time tracking via state**: Re-renders on every change — use `useRef`.

</details>

---

## 3. `usePrevious` [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function usePrevious<T>(value: T): T | undefined;
```

### Description

Returns the previous value of a state/prop. `undefined` on first render.

### Implementation

```tsx
import { useEffect, useRef } from "react";

function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  });

  return ref.current;
}
```

### Usage

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount ?? "—"}</p>
      <p>Diff: {prevCount !== undefined ? count - prevCount : 0}</p>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
    </div>
  );
}
```

### How it works

```
Render 1: ref.current = undefined → return undefined
Effect 1: ref.current = 0 (initial value)

Render 2: ref.current = 0 → return 0
Effect 2: ref.current = 1 (new value)

Render 3: ref.current = 1 → return 1
...
```

`useEffect` runs **after** render — so during render, `ref.current` still holds previous value.

### Edge Cases

- **First render**: Returns `undefined` — caller must handle.
- **Same value**: `previous === current` if no actual change.
- **Object/array**: Returns reference — `previous === current` if same reference.

### Variations

**With initial value:**

```tsx
function usePrevious<T>(value: T, initial: T): T {
  const ref = useRef<T>(initial);
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

**Distinguish "never updated" from "undefined":**

```tsx
function usePreviousChange<T>(value: T): {
  previous: T | undefined;
  hasChanged: boolean;
} {
  const ref = useRef<T | undefined>(undefined);
  const hasChangedRef = useRef(false);

  useEffect(() => {
    if (!Object.is(ref.current, value)) {
      hasChangedRef.current = true;
    }
    ref.current = value;
  });

  return {
    previous: ref.current,
    hasChanged: hasChangedRef.current,
  };
}
```

### Tests

```tsx
test("returns previous value", () => {
  const { result, rerender } = renderHook(({ value }) => usePrevious(value), {
    initialProps: { value: 1 },
  });

  expect(result.current).toBeUndefined();

  rerender({ value: 2 });
  expect(result.current).toBe(1);

  rerender({ value: 3 });
  expect(result.current).toBe(2);
});
```

</details>

---

## 4. `useLocalStorage` [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void, () => void];
```

### Description

Persists state to `localStorage`. SSR-safe (no errors on server). Returns `[value, setValue, removeValue]`.

### Implementation

```tsx
import { useState, useCallback, useEffect } from "react";

function useLocalStorage<T>(
  key: string,
  initialValue: T,
): [T, (value: T | ((prev: T) => T)) => void, () => void] {
  // Lazy initial state — read from localStorage once
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === "undefined") {
      return initialValue; // SSR — no window
    }
    try {
      const item = window.localStorage.getItem(key);
      return item !== null ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  // Setter
  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      try {
        const valueToStore =
          value instanceof Function ? value(storedValue) : value;
        setStoredValue(valueToStore);
        if (typeof window !== "undefined") {
          window.localStorage.setItem(key, JSON.stringify(valueToStore));
        }
      } catch (error) {
        console.warn(`Error writing localStorage key "${key}":`, error);
      }
    },
    [key, storedValue],
  );

  // Remove
  const removeValue = useCallback(() => {
    try {
      if (typeof window !== "undefined") {
        window.localStorage.removeItem(key);
      }
      setStoredValue(initialValue);
    } catch (error) {
      console.warn(`Error removing localStorage key "${key}":`, error);
    }
  }, [key, initialValue]);

  // Sync across tabs (storage event)
  useEffect(() => {
    if (typeof window === "undefined") return;

    const handleStorageChange = (e: StorageEvent) => {
      if (e.key === key && e.newValue !== null) {
        try {
          setStoredValue(JSON.parse(e.newValue) as T);
        } catch (error) {
          console.warn(`Error parsing storage event:`, error);
        }
      }
    };

    window.addEventListener("storage", handleStorageChange);
    return () => window.removeEventListener("storage", handleStorageChange);
  }, [key]);

  return [storedValue, setValue, removeValue];
}
```

### Usage

```tsx
function ThemeSwitcher() {
  const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");

  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      Current: {theme}
    </button>
  );
}

function User() {
  const [user, setUser, removeUser] = useLocalStorage<{ name: string } | null>(
    "user",
    null,
  );

  return (
    <div>
      {user ? (
        <>
          <p>Logged in as {user.name}</p>
          <button onClick={removeUser}>Logout</button>
        </>
      ) : (
        <button onClick={() => setUser({ name: "Ali" })}>Login</button>
      )}
    </div>
  );
}
```

### Edge Cases

- **SSR**: `window` undefined — returns `initialValue`.
- **localStorage quota exceeded**: `setItem` throws — caught, value still in state.
- **Invalid JSON**: Parse fails — falls back to `initialValue`.
- **localStorage disabled** (private mode): `setItem`/`getItem` may throw.

### Variations

**With sync via `storage` event (multi-tab):**

Already included above. When user updates in another tab, current tab updates too.

**TypeScript-strict variant:**

```tsx
function useLocalStorageTyped<T>(
  key: string,
  initialValue: T,
  validator?: (value: unknown) => value is T,
): [T, (value: T) => void] {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    if (stored === null) return initialValue;
    try {
      const parsed = JSON.parse(stored);
      if (validator && !validator(parsed)) {
        return initialValue;
      }
      return parsed as T;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = (newValue: T) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, setStoredValue];
}
```

### Common Mistakes

- ❌ **Reading localStorage in render**: SSR error or hydration mismatch — use lazy `useState`.
- ❌ **No JSON parse error handling**: Bad data crashes app.
- ❌ **Missing window check**: Crashes on server.
- ❌ **Not memoizing setter**: Re-creates on every render.

</details>

---

## 5. `useIntersectionObserver` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function useIntersectionObserver(
  options?: IntersectionObserverInit,
): [React.RefObject<HTMLElement>, IntersectionObserverEntry | null];
```

### Description

Observes element visibility using `IntersectionObserver`. Returns a ref and the latest entry.

### Implementation

```tsx
import { useEffect, useRef, useState } from "react";

function useIntersectionObserver<T extends HTMLElement = HTMLDivElement>(
  options: IntersectionObserverInit = {},
): [React.RefObject<T>, IntersectionObserverEntry | null] {
  const ref = useRef<T>(null);
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null);

  useEffect(() => {
    const node = ref.current;
    if (!node) return;

    const observer = new IntersectionObserver(
      (entries) => setEntry(entries[0]),
      options,
    );

    observer.observe(node);
    return () => observer.disconnect();
  }, [options.root, options.rootMargin, options.threshold]);

  return [ref, entry];
}
```

### Usage

**Lazy load images:**

```tsx
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const [ref, entry] = useIntersectionObserver<HTMLImageElement>({
    rootMargin: "100px",
    threshold: 0.1,
  });
  const isVisible = entry?.isIntersecting ?? false;

  return (
    <img
      ref={ref}
      src={isVisible ? src : "/placeholder.svg"}
      alt={alt}
      loading="lazy"
    />
  );
}
```

**Infinite scroll:**

```tsx
function InfiniteList() {
  const [items, setItems] = useState<string[]>([]);
  const [page, setPage] = useState(1);
  const [ref, entry] = useIntersectionObserver({ threshold: 1 });
  const isVisible = entry?.isIntersecting ?? false;

  useEffect(() => {
    fetch(`/api/items?page=${page}`)
      .then((r) => r.json())
      .then((newItems) => setItems((prev) => [...prev, ...newItems]));
  }, [page]);

  useEffect(() => {
    if (isVisible) {
      setPage((p) => p + 1);
    }
  }, [isVisible]);

  return (
    <>
      <ul>
        {items.map((item, idx) => (
          <li key={idx}>{item}</li>
        ))}
      </ul>
      <div ref={ref}>Loading more...</div>
    </>
  );
}
```

### Edge Cases

- **Element ref null at mount**: `useEffect` runs after mount — ref attached.
- **Conditional render**: Element unmounts → observer disconnects via cleanup.
- **Options change**: Effect deps trigger re-creation. Use stable options (memoize if dynamic).

### Variations

**With `freezeOnceVisible`:**

```tsx
function useIntersectionObserver<T extends HTMLElement = HTMLDivElement>(
  options: IntersectionObserverInit & { freezeOnceVisible?: boolean } = {},
): [React.RefObject<T>, IntersectionObserverEntry | null] {
  const ref = useRef<T>(null);
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null);
  const frozenRef = useRef(false);

  useEffect(() => {
    const node = ref.current;
    if (!node || frozenRef.current) return;

    const observer = new IntersectionObserver(([newEntry]) => {
      setEntry(newEntry);

      if (options.freezeOnceVisible && newEntry.isIntersecting) {
        frozenRef.current = true;
        observer.disconnect();
      }
    }, options);

    observer.observe(node);
    return () => observer.disconnect();
  }, [options.root, options.rootMargin, options.threshold, options.freezeOnceVisible]);

  return [ref, entry];
}
```

### Common Mistakes

- ❌ **No cleanup**: `disconnect()` missing → memory leak.
- ❌ **Object literal options**: New object each render → effect re-runs every render.
- ❌ **Multiple targets, single ref**: Use multiple hooks or shared observer pattern.

</details>

---

## 6. `useFetch` [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T = unknown>(
  url: string,
  options?: RequestInit,
): UseFetchResult<T>;
```

### Description

Basic fetch hook with `loading`, `error`, `data` states. Aborts in-flight on URL change or unmount.

### Implementation

```tsx
import { useState, useEffect } from "react";

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T = unknown>(
  url: string,
  options?: RequestInit,
): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { ...options, signal: controller.signal })
      .then((res) => {
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        }
        return res.json();
      })
      .then((data: T) => {
        setData(data);
        setLoading(false);
      })
      .catch((err: Error) => {
        if (err.name !== "AbortError") {
          setError(err);
          setLoading(false);
        }
      });

    return () => controller.abort();
  }, [url]); // options not in deps — usually stable

  return { data, loading, error };
}
```

### Usage

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <p>Error: {error.message}</p>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### Edge Cases

- **URL change**: Cleanup aborts previous, fetches new.
- **Component unmount**: Cleanup aborts in-flight.
- **Network error**: Caught, sets error state.
- **HTTP error (4xx, 5xx)**: Manual `res.ok` check — sets error.

### Variations

**With refetch:**

```tsx
interface UseFetchResultWithRefetch<T> extends UseFetchResult<T> {
  refetch: () => void;
}

function useFetch<T>(url: string): UseFetchResultWithRefetch<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [trigger, setTrigger] = useState(0);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);

    fetch(url, { signal: controller.signal })
      .then((r) => r.json())
      .then(setData)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url, trigger]);

  const refetch = () => setTrigger((t) => t + 1);

  return { data, loading, error, refetch };
}
```

**With cache (production):**

For real apps — use `react-query`, `swr`, or `tanstack/query`. Custom cache implementations get complex.

### Common Mistakes

- ❌ **No abort**: Race conditions on rapid URL changes.
- ❌ **No HTTP error check**: 4xx/5xx not caught (fetch only rejects on network error).
- ❌ **State on unmounted component**: Setting state after unmount → React warning.
- ❌ **`options` in deps**: Object literal changes every render → infinite loop.

</details>

---

## 7. `useEventListener` [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Signature

```typescript
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element?: Window | HTMLElement | null,
): void;
```

### Description

Attaches event listener with automatic cleanup. Stable callback reference (no re-attach on every render).

### Implementation

```tsx
import { useEffect, useRef } from "react";

function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window | HTMLElement | null = window,
): void {
  // Save handler ref to avoid re-attaching on each render
  const handlerRef = useRef(handler);
  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);

  useEffect(() => {
    if (!element) return;

    const eventListener: EventListener = (event) => {
      handlerRef.current(event as WindowEventMap[K]);
    };

    element.addEventListener(eventName, eventListener);
    return () => element.removeEventListener(eventName, eventListener);
  }, [eventName, element]);
}
```

### Usage

```tsx
function KeyboardShortcuts() {
  useEventListener("keydown", (e) => {
    if (e.key === "Escape") {
      console.log("Escape pressed");
    }
    if ((e.ctrlKey || e.metaKey) && e.key === "s") {
      e.preventDefault();
      console.log("Save");
    }
  });

  return <div>Press keys</div>;
}

function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);

  useEventListener("scroll", () => {
    setScrollY(window.scrollY);
  });

  return <p>Scroll: {scrollY}</p>;
}

function ButtonClicks() {
  const buttonRef = useRef<HTMLButtonElement>(null);

  useEventListener(
    "click" as any, // HTMLElementEventMap
    () => console.log("Button clicked"),
    buttonRef.current,
  );

  return <button ref={buttonRef}>Click me</button>;
}
```

### Edge Cases

- **Handler changes per render**: ref pattern — always uses latest handler without re-attaching.
- **Element ref null**: Guarded with `if (!element) return;`.
- **Element unmount**: Cleanup removes listener.

### Variations

**Multiple event types:**

```tsx
function useMultipleEventListener(
  events: { name: string; handler: (e: Event) => void }[],
  element: HTMLElement | Window = window,
) {
  useEffect(() => {
    events.forEach(({ name, handler }) => element.addEventListener(name, handler));
    return () => {
      events.forEach(({ name, handler }) => element.removeEventListener(name, handler));
    };
  }, [events, element]);
}
```

**With options (capture, passive, once):**

```tsx
function useEventListenerWithOptions<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window | HTMLElement = window,
  options?: AddEventListenerOptions,
): void {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const listener: EventListener = (e) =>
      handlerRef.current(e as WindowEventMap[K]);

    element.addEventListener(eventName, listener, options);
    return () => element.removeEventListener(eventName, listener, options);
  }, [eventName, element, options?.capture, options?.passive, options?.once]);
}
```

### Common Mistakes

- ❌ **Inline handler in deps**: New function each render → re-attach. Use ref pattern.
- ❌ **No cleanup**: Memory leak.
- ❌ **Forgetting `passive`**: For scroll/touch — `passive: true` improves perf.

</details>

---

## 8. HOC: `withAuth` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Higher-Order Component that wraps a component, redirecting unauthenticated users to login.

### Implementation

```tsx
import { ComponentType, useEffect } from "react";
import { useAuth } from "./AuthContext";
import { useNavigate } from "react-router-dom";

interface WithAuthOptions {
  redirectTo?: string;
  loadingComponent?: ComponentType;
}

function withAuth<P extends object>(
  WrappedComponent: ComponentType<P>,
  options: WithAuthOptions = {},
): ComponentType<P> {
  const { redirectTo = "/login", loadingComponent: Loading = () => <p>Loading...</p> } =
    options;

  function WithAuth(props: P) {
    const { user, loading } = useAuth();
    const navigate = useNavigate();

    useEffect(() => {
      if (!loading && !user) {
        navigate(redirectTo);
      }
    }, [user, loading, navigate]);

    if (loading) return <Loading />;
    if (!user) return null; // While redirecting

    return <WrappedComponent {...props} />;
  }

  // Preserve display name for debugging
  WithAuth.displayName = `withAuth(${WrappedComponent.displayName || WrappedComponent.name || "Component"})`;

  return WithAuth;
}

export default withAuth;
```

### Usage

```tsx
function Dashboard({ userId }: { userId: string }) {
  return <h1>Dashboard for {userId}</h1>;
}

const ProtectedDashboard = withAuth(Dashboard, {
  redirectTo: "/login",
});

// Routes
function App() {
  return (
    <Routes>
      <Route path="/dashboard/:id" element={<ProtectedDashboard userId="123" />} />
      <Route path="/login" element={<Login />} />
    </Routes>
  );
}
```

### AuthContext (companion):

```tsx
import { createContext, useContext, useState, useEffect, ReactNode } from "react";

interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthContextValue {
  user: User | null;
  loading: boolean;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch("/api/me")
      .then((r) => (r.ok ? r.json() : null))
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);

  const login = async (credentials: Credentials) => {
    const res = await fetch("/api/login", {
      method: "POST",
      body: JSON.stringify(credentials),
    });
    const user = await res.json();
    setUser(user);
  };

  const logout = () => {
    fetch("/api/logout", { method: "POST" });
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}
```

### Modern alternative — Hook + Conditional render

```tsx
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user, loading } = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (!loading && !user) {
      navigate("/login");
    }
  }, [user, loading, navigate]);

  if (loading) return <Loading />;
  if (!user) return null;
  return <>{children}</>;
}

// Usage
<Route path="/dashboard" element={
  <ProtectedRoute>
    <Dashboard />
  </ProtectedRoute>
} />
```

### Common Mistakes

- ❌ **No loading state**: Flash of "redirecting to login" when user is actually authenticated.
- ❌ **`displayName` missing**: DevTools shows "Anonymous" — hard to debug.
- ❌ **Props not forwarded**: `{...props}` spread missing.

</details>

---

## 9. HOC: `withErrorBoundary` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Wraps a component in an error boundary.

### Implementation

```tsx
import { ComponentType, ErrorInfo, ReactNode, Component } from "react";

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.props.onError?.(error, errorInfo);
  }

  reset = () => {
    this.setState({ error: null });
  };

  render() {
    if (this.state.error !== null) {
      return this.props.fallback(this.state.error, this.reset);
    }
    return this.props.children;
  }
}

interface WithErrorBoundaryOptions {
  fallback: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

function withErrorBoundary<P extends object>(
  WrappedComponent: ComponentType<P>,
  options: WithErrorBoundaryOptions,
): ComponentType<P> {
  function WithErrorBoundary(props: P) {
    return (
      <ErrorBoundary fallback={options.fallback} onError={options.onError}>
        <WrappedComponent {...props} />
      </ErrorBoundary>
    );
  }

  WithErrorBoundary.displayName = `withErrorBoundary(${
    WrappedComponent.displayName || WrappedComponent.name || "Component"
  })`;

  return WithErrorBoundary;
}

export default withErrorBoundary;
```

### Usage

```tsx
function RiskyComponent() {
  // May throw
  if (Math.random() > 0.5) {
    throw new Error("Random failure");
  }
  return <p>Success</p>;
}

const SafeComponent = withErrorBoundary(RiskyComponent, {
  fallback: (error, reset) => (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  ),
  onError: (error, info) => {
    logToTelemetry({ error, componentStack: info.componentStack });
  },
});

function App() {
  return <SafeComponent />;
}
```

### Common Mistakes

- ❌ **Catching async errors**: Error boundaries don't catch async errors in event handlers — manual handling.
- ❌ **No error logging**: Production errors silently swallowed.
- ❌ **No reset**: Once errored, stuck — provide reset.

</details>

---

## 10. `ErrorBoundary` class component [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Class component implementing error boundary lifecycle. Catches errors in render phase of children, lifecycle methods, and constructors.

### Implementation

```tsx
import { Component, ErrorInfo, ReactNode } from "react";

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  resetKeys?: Array<unknown>;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    // Update state when an error is thrown
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // Side effects (logging, telemetry)
    this.props.onError?.(error, errorInfo);
  }

  componentDidUpdate(prevProps: ErrorBoundaryProps): void {
    // Reset error state when resetKeys change
    if (
      this.state.hasError &&
      this.props.resetKeys &&
      prevProps.resetKeys &&
      this.props.resetKeys.length === prevProps.resetKeys.length &&
      this.props.resetKeys.some((key, idx) => !Object.is(key, prevProps.resetKeys![idx]))
    ) {
      this.reset();
    }
  }

  reset = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError && this.state.error !== null) {
      const { fallback } = this.props;
      if (typeof fallback === "function") {
        return fallback(this.state.error, this.reset);
      }
      return fallback ?? <div>Something went wrong</div>;
    }
    return this.props.children;
  }
}
```

### Usage

```tsx
function App() {
  const [retryKey, setRetryKey] = useState(0);

  return (
    <ErrorBoundary
      resetKeys={[retryKey]}
      fallback={(error, reset) => (
        <div>
          <h2>Something went wrong</h2>
          <details>
            <summary>Error details</summary>
            <pre>{error.message}</pre>
          </details>
          <button
            onClick={() => {
              setRetryKey((k) => k + 1); // Trigger reset
              reset();
            }}
          >
            Retry
          </button>
        </div>
      )}
      onError={(error, info) => {
        console.error("Caught:", error);
        // logToService(error, info.componentStack);
      }}
    >
      <RiskyChild />
    </ErrorBoundary>
  );
}
```

### What ErrorBoundary catches

✅ **Catches:**
- Render errors in children
- Lifecycle method errors
- Constructor errors

❌ **Does NOT catch:**
- Async errors (event handlers, setTimeout, fetch)
- SSR errors
- Errors in the boundary itself

### Edge Cases

- **Boundary's own render error**: Propagates to outer boundary.
- **Async errors**: Manual try-catch + `setState`.
- **`getDerivedStateFromError` async**: Cannot — must be synchronous.

### R19 root-level callbacks

```tsx
const root = createRoot(container, {
  onCaughtError: (error, info) => {
    // Caught by ErrorBoundary
  },
  onUncaughtError: (error, info) => {
    // No boundary caught — fallback
  },
  onRecoverableError: (error, info) => {
    // Hydration mismatch, recovered
  },
});
```

### Common Mistakes

- ❌ **`getDerivedStateFromError` side effects**: Logging — use `componentDidCatch`.
- ❌ **No reset mechanism**: Errored state persists.
- ❌ **Catching errors that should propagate**: Some errors are bugs that should crash.

</details>

---

## 11. Compound Component pattern [Senior]

<details>
<summary><strong>Implementation</strong></summary>

### Description

A pattern where related components share implicit state via context, exposing a flexible API. Example: `<Tabs>`, `<Accordion>`, `<Select>`.

### Implementation — Tabs

```tsx
import { createContext, useContext, useState, useMemo, ReactNode } from "react";

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error("Tabs components must be used within <Tabs>");
  }
  return ctx;
}

interface TabsProps {
  defaultActive: string;
  children: ReactNode;
}

function Tabs({ defaultActive, children }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultActive);
  // Context value'ni memoize qilamiz — har render yangi object yaratilsa,
  // barcha consumer'lar re-render bo'ladi
  const value = useMemo(() => ({ activeTab, setActiveTab }), [activeTab]);
  return (
    <TabsContext.Provider value={value}>
      <div role="tablist">{children}</div>
    </TabsContext.Provider>
  );
}

interface TabListProps {
  children: ReactNode;
}

function TabList({ children }: TabListProps) {
  return <div className="tab-list">{children}</div>;
}

interface TabProps {
  id: string;
  children: ReactNode;
}

function Tab({ id, children }: TabProps) {
  const { activeTab, setActiveTab } = useTabsContext();
  const isActive = activeTab === id;
  return (
    <button
      role="tab"
      aria-selected={isActive}
      onClick={() => setActiveTab(id)}
      className={isActive ? "active" : ""}
    >
      {children}
    </button>
  );
}

interface TabPanelsProps {
  children: ReactNode;
}

function TabPanels({ children }: TabPanelsProps) {
  return <div className="tab-panels">{children}</div>;
}

interface TabPanelProps {
  id: string;
  children: ReactNode;
}

function TabPanel({ id, children }: TabPanelProps) {
  const { activeTab } = useTabsContext();
  if (activeTab !== id) return null;
  return (
    <div role="tabpanel" id={`panel-${id}`}>
      {children}
    </div>
  );
}

// Attach as static properties
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

export { Tabs };
```

### Usage

```tsx
function ProductPage() {
  return (
    <Tabs defaultActive="overview">
      <Tabs.List>
        <Tabs.Tab id="overview">Overview</Tabs.Tab>
        <Tabs.Tab id="specs">Specifications</Tabs.Tab>
        <Tabs.Tab id="reviews">Reviews</Tabs.Tab>
      </Tabs.List>

      <Tabs.Panels>
        <Tabs.Panel id="overview">
          <h2>Product Overview</h2>
          <p>...</p>
        </Tabs.Panel>
        <Tabs.Panel id="specs">
          <h2>Specifications</h2>
          <ul>...</ul>
        </Tabs.Panel>
        <Tabs.Panel id="reviews">
          <h2>Customer Reviews</h2>
          <Reviews />
        </Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

### Benefits

- **Implicit state sharing** via Context — no prop drilling
- **Flexible composition** — children can be reordered, wrapped, nested
- **Type safety** — discrete components for each role
- **Self-documenting** — clear hierarchy

### Variations

**Compound + Render Props (advanced control):**

```tsx
interface TabsProps {
  defaultActive: string;
  children: (api: TabsContextValue) => ReactNode;
}

function Tabs({ defaultActive, children }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultActive);
  const value = { activeTab, setActiveTab };
  return (
    <TabsContext.Provider value={value}>
      {children(value)}
    </TabsContext.Provider>
  );
}

// Usage
<Tabs defaultActive="a">
  {({ activeTab, setActiveTab }) => (
    <>
      <Tabs.List>...</Tabs.List>
      <p>Active: {activeTab}</p>
    </>
  )}
</Tabs>
```

**Controlled + Uncontrolled:**

```tsx
interface TabsProps {
  active?: string; // Controlled
  defaultActive?: string; // Uncontrolled
  onActiveChange?: (id: string) => void;
  children: ReactNode;
}

function Tabs({ active: activeProp, defaultActive, onActiveChange, children }: TabsProps) {
  const [internalActive, setInternalActive] = useState(defaultActive ?? "");
  const isControlled = activeProp !== undefined;
  const activeTab = isControlled ? activeProp : internalActive;

  const setActiveTab = (id: string) => {
    if (!isControlled) setInternalActive(id);
    onActiveChange?.(id);
  };

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}
```

### Edge Cases

- **`Tabs.Tab` outside `<Tabs>`**: Context throws — explicit error.
- **Duplicate IDs**: First match wins (current implementation).
- **Empty active state**: No panel rendered — handle defaultActive carefully.

### Common Mistakes

- ❌ **No context check**: Components used outside Provider — silent failure.
- ❌ **Inline Context value**: New object each render → all consumers re-render.
- ❌ **No keyboard navigation**: ARIA tabs need arrow keys, Home/End.

</details>

---

## 12. `useReducer` + Context global state [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Global state management using `useReducer` (state logic) + Context (provider). Lightweight Redux alternative.

### Implementation

```tsx
import { createContext, useContext, useReducer, ReactNode, Dispatch } from "react";

// State + Action types
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  total: number;
}

type CartAction =
  | { type: "ADD"; payload: Omit<CartItem, "quantity"> }
  | { type: "REMOVE"; payload: { id: string } }
  | { type: "UPDATE_QUANTITY"; payload: { id: string; quantity: number } }
  | { type: "CLEAR" };

// Reducer
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD": {
      const existing = state.items.find((i) => i.id === action.payload.id);
      const items = existing
        ? state.items.map((i) =>
            i.id === action.payload.id ? { ...i, quantity: i.quantity + 1 } : i
          )
        : [...state.items, { ...action.payload, quantity: 1 }];
      return { items, total: computeTotal(items) };
    }
    case "REMOVE": {
      const items = state.items.filter((i) => i.id !== action.payload.id);
      return { items, total: computeTotal(items) };
    }
    case "UPDATE_QUANTITY": {
      if (action.payload.quantity <= 0) {
        const items = state.items.filter((i) => i.id !== action.payload.id);
        return { items, total: computeTotal(items) };
      }
      const items = state.items.map((i) =>
        i.id === action.payload.id ? { ...i, quantity: action.payload.quantity } : i
      );
      return { items, total: computeTotal(items) };
    }
    case "CLEAR":
      return { items: [], total: 0 };
    default:
      return state;
  }
}

function computeTotal(items: CartItem[]): number {
  return items.reduce((sum, i) => sum + i.price * i.quantity, 0);
}

const initialState: CartState = { items: [], total: 0 };

// Contexts — split state and dispatch for performance
const CartStateContext = createContext<CartState | null>(null);
const CartDispatchContext = createContext<Dispatch<CartAction> | null>(null);

// Provider
export function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  return (
    <CartStateContext.Provider value={state}>
      <CartDispatchContext.Provider value={dispatch}>
        {children}
      </CartDispatchContext.Provider>
    </CartStateContext.Provider>
  );
}

// Hooks
export function useCartState(): CartState {
  const ctx = useContext(CartStateContext);
  if (!ctx) throw new Error("useCartState must be used within CartProvider");
  return ctx;
}

export function useCartDispatch(): Dispatch<CartAction> {
  const ctx = useContext(CartDispatchContext);
  if (!ctx) throw new Error("useCartDispatch must be used within CartProvider");
  return ctx;
}
```

### Usage

```tsx
// Top-level
function App() {
  return (
    <CartProvider>
      <ProductList />
      <CartSummary />
    </CartProvider>
  );
}

// Read state
function CartSummary() {
  const { items, total } = useCartState();
  return (
    <div>
      <p>Items: {items.length}</p>
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  );
}

// Dispatch actions
function ProductCard({ product }: { product: Product }) {
  const dispatch = useCartDispatch();

  return (
    <div>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => dispatch({ type: "ADD", payload: product })}>
        Add to Cart
      </button>
    </div>
  );
}

function CartItem({ item }: { item: CartItem }) {
  const dispatch = useCartDispatch();

  return (
    <li>
      <span>{item.name} x {item.quantity}</span>
      <button
        onClick={() =>
          dispatch({
            type: "UPDATE_QUANTITY",
            payload: { id: item.id, quantity: item.quantity - 1 },
          })
        }
      >
        -
      </button>
      <button
        onClick={() =>
          dispatch({
            type: "UPDATE_QUANTITY",
            payload: { id: item.id, quantity: item.quantity + 1 },
          })
        }
      >
        +
      </button>
      <button onClick={() => dispatch({ type: "REMOVE", payload: { id: item.id } })}>
        Remove
      </button>
    </li>
  );
}
```

### Why split state and dispatch contexts?

```tsx
// ❌ Combined — any state change re-renders all consumers using dispatch
const CartContext = createContext<{ state, dispatch }>(...);

// ✅ Split — components using only dispatch don't re-render on state change
const CartStateContext = createContext<State>(...);
const CartDispatchContext = createContext<Dispatch<Action>>(...);
```

### Variations

**With selector hook:**

```tsx
import { useSyncExternalStoreWithSelector } from "use-sync-external-store/with-selector";

// Custom store — manual subscriptions
function createStore<S, A>(reducer: (s: S, a: A) => S, initial: S) {
  let state = initial;
  const listeners = new Set<() => void>();

  return {
    getState: () => state,
    dispatch: (action: A) => {
      state = reducer(state, action);
      listeners.forEach((l) => l());
    },
    subscribe: (cb: () => void) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
  };
}

const cartStore = createStore(cartReducer, initialState);

function useCartSelector<T>(selector: (s: CartState) => T): T {
  return useSyncExternalStoreWithSelector(
    cartStore.subscribe,
    cartStore.getState,
    cartStore.getState, // SSR
    selector,
  );
}

// Usage:
function CartCount() {
  const count = useCartSelector((s) => s.items.length);
  return <p>{count}</p>;
}
```

### Common Mistakes

- ❌ **Combining state + dispatch in one context**: All consumers re-render on any change.
- ❌ **Mutating state in reducer**: Reducer must be pure.
- ❌ **No initial state**: `useReducer(reducer)` — undefined initial → bugs.
- ❌ **Side effects in reducer**: Pure functions only.

</details>

---

## 13. `React.memo` + `useCallback` optimize [Middle]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Optimize a re-rendering list using `React.memo` for items + `useCallback` for stable handlers.

### Before (unoptimized)

```tsx
function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: "1", text: "Learn React", done: false },
    { id: "2", text: "Build app", done: false },
  ]);
  const [filter, setFilter] = useState("");

  const toggleTodo = (id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t))
    );
  };

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {todos
        .filter((t) => t.text.includes(filter))
        .map((t) => (
          <TodoItem key={t.id} todo={t} onToggle={toggleTodo} />
        ))}
    </>
  );
}

function TodoItem({ todo, onToggle }: TodoItemProps) {
  console.log("TodoItem render:", todo.id);
  return (
    <div>
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => onToggle(todo.id)}
      />
      {todo.text}
    </div>
  );
}

// Problem: typing in filter input re-renders all TodoItems
// (because toggleTodo new ref each render)
```

### After (optimized)

```tsx
import { memo, useCallback, useMemo, useState } from "react";

interface Todo {
  id: string;
  text: string;
  done: boolean;
}

interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
}

const TodoItem = memo(function TodoItem({ todo, onToggle }: TodoItemProps) {
  console.log("TodoItem render:", todo.id);
  return (
    <div>
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => onToggle(todo.id)}
      />
      {todo.text}
    </div>
  );
});

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: "1", text: "Learn React", done: false },
    { id: "2", text: "Build app", done: false },
  ]);
  const [filter, setFilter] = useState("");

  // Stable reference — no re-create on filter change
  const toggleTodo = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t))
    );
  }, []);

  // Memoize filtered list (optional, for large lists)
  const filtered = useMemo(
    () => todos.filter((t) => t.text.includes(filter)),
    [todos, filter]
  );

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {filtered.map((t) => (
        <TodoItem key={t.id} todo={t} onToggle={toggleTodo} />
      ))}
    </>
  );
}

// Now: filter typing only re-renders TodoList
// TodoItems re-render only when todo prop changes (toggle)
```

### Key changes

1. **`memo(TodoItem)`** — bailout when `todo` and `onToggle` props don't change
2. **`useCallback(toggleTodo)`** — stable reference across renders
3. **Functional update `setTodos((prev) => ...)`** — no `todos` in deps
4. **`useMemo(filtered)`** — avoid re-computing on every render

### Profiling — verify

```tsx
import { Profiler } from "react";

<Profiler
  id="TodoList"
  onRender={(id, phase, actualDuration, baseDuration) => {
    console.log(`${id} ${phase}: actual=${actualDuration}ms, base=${baseDuration}ms`);
  }}
>
  <TodoList />
</Profiler>
```

`actualDuration < baseDuration` → memoization effective.

### When NOT to memoize

```tsx
// ❌ Trivial component — memo overhead > render
const Greeting = memo(({ name }: { name: string }) => <p>Hello, {name}</p>);

// ✅ Inline OK
function Greeting({ name }: { name: string }) {
  return <p>Hello, {name}</p>;
}
```

### Common Mistakes

- ❌ **`useCallback` deps include state**: `[count]` → callback re-creates on every count change. Use functional update.
- ❌ **Memo without `useCallback`**: Inline handler — each render new ref → memo bypass.
- ❌ **Object/array literal in props**: `<TodoItem todo={{ ...todo }} />` → new ref every render.

</details>

---

## 14. Virtualized list (windowing) [Senior]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Render only visible items in a long list. Track scroll position, compute visible range, position items absolutely.

### Implementation

```tsx
import {
  useState,
  useRef,
  useMemo,
  useCallback,
  useLayoutEffect,
  CSSProperties,
} from "react";

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  containerHeight: number;
  overscan?: number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

export function VirtualList<T>({
  items,
  itemHeight,
  containerHeight,
  overscan = 5,
  renderItem,
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);
  const rafRef = useRef<number | null>(null);

  // RAF-throttled scroll handler
  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    const newScrollTop = e.currentTarget.scrollTop;
    if (rafRef.current !== null) return;
    rafRef.current = requestAnimationFrame(() => {
      setScrollTop(newScrollTop);
      rafRef.current = null;
    });
  }, []);

  // Compute visible range
  const { startIndex, endIndex, totalHeight, visibleItems } = useMemo(() => {
    const totalHeight = items.length * itemHeight;
    const start = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
    const end = Math.min(
      items.length,
      Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan,
    );
    const visibleItems = items.slice(start, end);
    return { startIndex: start, endIndex: end, totalHeight, visibleItems };
  }, [items, itemHeight, scrollTop, containerHeight, overscan]);

  // Cleanup RAF on unmount
  useLayoutEffect(() => {
    return () => {
      if (rafRef.current !== null) {
        cancelAnimationFrame(rafRef.current);
      }
    };
  }, []);

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{
        height: containerHeight,
        overflow: "auto",
        position: "relative",
      }}
    >
      <div style={{ height: totalHeight, position: "relative" }}>
        {visibleItems.map((item, idx) => {
          const actualIndex = startIndex + idx;
          const style: CSSProperties = {
            position: "absolute",
            top: 0,
            left: 0,
            right: 0,
            height: itemHeight,
            transform: `translateY(${actualIndex * itemHeight}px)`,
          };
          return (
            <div key={actualIndex} style={style} role="listitem">
              {renderItem(item, actualIndex)}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### Usage

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

function UserList() {
  const users: User[] = useMemo(
    () =>
      Array.from({ length: 10000 }, (_, i) => ({
        id: `user-${i}`,
        name: `User ${i}`,
        email: `user${i}@example.com`,
      })),
    []
  );

  return (
    <VirtualList
      items={users}
      itemHeight={50}
      containerHeight={600}
      overscan={5}
      renderItem={(user) => (
        <div style={{ padding: 8, borderBottom: "1px solid #ccc" }}>
          <strong>{user.name}</strong>
          <span style={{ marginLeft: 8, color: "gray" }}>{user.email}</span>
        </div>
      )}
    />
  );
}

// 10K users render — but only ~20 in DOM at a time
// Smooth scrolling, low memory
```

### Variations

**Variable-size items:**

```tsx
interface VariableVirtualListProps<T> {
  items: T[];
  estimatedItemHeight: number;
  containerHeight: number;
  getItemHeight: (index: number) => number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

export function VariableVirtualList<T>({
  items,
  estimatedItemHeight,
  containerHeight,
  getItemHeight,
  renderItem,
}: VariableVirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);

  // Cumulative offsets (precomputed)
  const offsets = useMemo(() => {
    const arr: number[] = [0];
    for (let i = 0; i < items.length; i++) {
      arr[i + 1] = arr[i] + getItemHeight(i);
    }
    return arr;
  }, [items.length, getItemHeight]);

  const totalHeight = offsets[items.length];

  // Binary search for visible range
  const startIndex = useMemo(() => {
    let lo = 0, hi = items.length - 1;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (offsets[mid + 1] <= scrollTop) lo = mid + 1;
      else hi = mid;
    }
    return Math.max(0, lo - 1);
  }, [scrollTop, offsets, items.length]);

  const endIndex = useMemo(() => {
    let i = startIndex;
    while (i < items.length && offsets[i] < scrollTop + containerHeight) {
      i++;
    }
    return Math.min(items.length, i);
  }, [startIndex, scrollTop, containerHeight, offsets, items.length]);

  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height: containerHeight, overflow: "auto" }}
    >
      <div style={{ height: totalHeight, position: "relative" }}>
        {visibleItems.map((item, idx) => {
          const actualIndex = startIndex + idx;
          return (
            <div
              key={actualIndex}
              style={{
                position: "absolute",
                top: offsets[actualIndex],
                left: 0,
                right: 0,
                height: getItemHeight(actualIndex),
              }}
            >
              {renderItem(item, actualIndex)}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### Edge Cases

- **Empty items**: Total height 0, no items rendered.
- **Container height 0**: Visible range empty.
- **Scroll position larger than total**: Browser clamps scrollTop.

### Common Mistakes

- ❌ **`top: index * height` instead of `transform`**: Less performant (paint vs composite).
- ❌ **No RAF throttle**: Scroll handler fires synchronously — jank.
- ❌ **Inline new function as `renderItem`**: Re-creates each render, memoize.

### Production note

For real apps, use `@tanstack/react-virtual` or `react-window`. Custom impl OK for specific requirements.

</details>

---

## 15. `useAsync` — Promise hook [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Hook for executing async functions with loading/error/data states. Cancellable, refetchable.

### Implementation

```tsx
import { useState, useEffect, useCallback, useRef } from "react";

type AsyncStatus = "idle" | "pending" | "success" | "error";

interface UseAsyncResult<T, A extends any[]> {
  status: AsyncStatus;
  data: T | null;
  error: Error | null;
  execute: (...args: A) => Promise<void>;
  reset: () => void;
}

function useAsync<T, A extends any[] = []>(
  asyncFn: (...args: A) => Promise<T>,
  immediate: boolean = false,
  immediateArgs?: A,
): UseAsyncResult<T, A> {
  const [status, setStatus] = useState<AsyncStatus>("idle");
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const mountedRef = useRef(true);

  useEffect(() => {
    mountedRef.current = true;
    return () => {
      mountedRef.current = false;
    };
  }, []);

  const execute = useCallback(
    async (...args: A) => {
      setStatus("pending");
      setData(null);
      setError(null);

      try {
        const result = await asyncFn(...args);
        if (mountedRef.current) {
          setData(result);
          setStatus("success");
        }
      } catch (err) {
        if (mountedRef.current) {
          setError(err as Error);
          setStatus("error");
        }
      }
    },
    [asyncFn],
  );

  const reset = useCallback(() => {
    setStatus("idle");
    setData(null);
    setError(null);
  }, []);

  useEffect(() => {
    if (immediate && immediateArgs) {
      execute(...immediateArgs);
    } else if (immediate) {
      execute(...([] as unknown as A));
    }
  }, [immediate]);

  return { status, data, error, execute, reset };
}

export default useAsync;
```

### Usage

```tsx
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("Failed to fetch");
  return res.json();
}

function UserProfile() {
  const { status, data: user, error, execute } = useAsync(fetchUser);

  return (
    <div>
      <button onClick={() => execute("123")} disabled={status === "pending"}>
        {status === "pending" ? "Loading..." : "Load User"}
      </button>
      {status === "error" && <p>Error: {error?.message}</p>}
      {user && <p>{user.name}</p>}
    </div>
  );
}

// Auto-execute
function UserAuto({ id }: { id: string }) {
  const { status, data, error } = useAsync(fetchUser, true, [id]);

  if (status === "pending") return <Spinner />;
  if (status === "error") return <p>{error?.message}</p>;
  return <p>{data?.name}</p>;
}
```

### Variations

**With AbortController:**

```tsx
function useAsyncWithAbort<T>(
  asyncFn: (signal: AbortSignal) => Promise<T>,
): UseAsyncResult<T, []> {
  const controllerRef = useRef<AbortController | null>(null);

  const execute = useCallback(async () => {
    // Abort previous
    controllerRef.current?.abort();
    controllerRef.current = new AbortController();

    setStatus("pending");
    try {
      const result = await asyncFn(controllerRef.current.signal);
      setData(result);
      setStatus("success");
    } catch (err) {
      if ((err as Error).name !== "AbortError") {
        setError(err as Error);
        setStatus("error");
      }
    }
  }, [asyncFn]);

  // ... rest
}
```

### Common Mistakes

- ❌ **No mounted check**: Set state after unmount → React warning.
- ❌ **No abort**: Race conditions with rapid execute.
- ❌ **Resetting before async**: User sees flicker — set state only on completion.

</details>

---

## 16. Suspense + lazy code splitting [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Code-split route or component using `React.lazy` + Suspense.

### Implementation

```tsx
import { lazy, Suspense, ReactNode } from "react";
import { ErrorBoundary } from "react-error-boundary";

// Route components — code split
const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const NotFound = lazy(() => import("./pages/NotFound"));

interface RouteFallbackProps {
  children: ReactNode;
}

function RouteFallback({ children }: RouteFallbackProps) {
  return (
    <ErrorBoundary
      fallbackRender={({ error, resetErrorBoundary }) => (
        <div role="alert">
          <h2>Failed to load page</h2>
          <pre>{error.message}</pre>
          <button onClick={resetErrorBoundary}>Retry</button>
        </div>
      )}
    >
      <Suspense fallback={<PageSkeleton />}>{children}</Suspense>
    </ErrorBoundary>
  );
}

import { BrowserRouter, Routes, Route } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route
          path="/"
          element={
            <RouteFallback>
              <Home />
            </RouteFallback>
          }
        />
        <Route
          path="/dashboard"
          element={
            <RouteFallback>
              <Dashboard />
            </RouteFallback>
          }
        />
        <Route
          path="/settings"
          element={
            <RouteFallback>
              <Settings />
            </RouteFallback>
          }
        />
        <Route
          path="*"
          element={
            <RouteFallback>
              <NotFound />
            </RouteFallback>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Hover prefetch

```tsx
import { useCallback } from "react";

function NavLink({ href, importFn, children }: NavLinkProps) {
  const handleHover = useCallback(() => {
    importFn(); // Trigger import (cached after first)
  }, [importFn]);

  return (
    <a href={href} onMouseEnter={handleHover} onFocus={handleHover}>
      {children}
    </a>
  );
}

// Usage
<NavLink
  href="/dashboard"
  importFn={() => import("./pages/Dashboard")}
>
  Dashboard
</NavLink>
```

### Named exports workaround

```tsx
// Default export — direct
const Home = lazy(() => import("./Home")); // Home is default export

// Named export — wrap
const Settings = lazy(() =>
  import("./pages").then((mod) => ({ default: mod.Settings }))
);
```

### Common Mistakes

- ❌ **No Suspense**: Lazy throws Promise → uncaught.
- ❌ **No ErrorBoundary**: Network failure → uncaught error.
- ❌ **Lazy in render**: `const X = lazy(...)` inside component → new lazy each render.

</details>

---

## 17. Render Props pattern [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Pattern where a component takes a function as a prop (or children) and calls it with state/data. Largely superseded by Hooks, but still used in libraries.

### Implementation

```tsx
import { useState, useEffect, ReactNode } from "react";

interface MouseTrackerProps {
  children: (state: { x: number; y: number }) => ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);

  return <>{children(position)}</>;
}
```

### Usage

```tsx
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <div>
          Mouse at ({x}, {y})
        </div>
      )}
    </MouseTracker>
  );
}
```

### Generic data fetcher

```tsx
interface FetchProps<T> {
  url: string;
  children: (state: { data: T | null; loading: boolean; error: Error | null }) => ReactNode;
}

function Fetch<T>({ url, children }: FetchProps<T>) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    setState({ data: null, loading: true, error: null });
    fetch(url)
      .then((r) => r.json())
      .then((data) => setState({ data, loading: false, error: null }))
      .catch((error) => setState({ data: null, loading: false, error }));
  }, [url]);

  return <>{children(state)}</>;
}

// Usage
<Fetch<User[]> url="/api/users">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <p>{error.message}</p>;
    return <UserList users={data ?? []} />;
  }}
</Fetch>
```

### Compared to Hooks

```tsx
// Render props
<MouseTracker>{({ x, y }) => <p>{x}, {y}</p>}</MouseTracker>

// Hooks — cleaner
function App() {
  const { x, y } = useMouse();
  return <p>{x}, {y}</p>;
}
```

### When still useful

- Reusing legacy code
- Class component-only contexts
- Need to share state to components that may not be Hooks
- Library APIs (e.g., `react-virtual`, `formik`'s `<Form>`)

### Common Mistakes

- ❌ **Inline JSX in render prop**: Re-creates each render → child re-render. Hoist or memoize.
- ❌ **Multiple render props**: Nesting hell — Hooks are flatter.

</details>

---

## 18. R19 `use()` data fetching [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Use the R19 `use()` Hook to suspend on a Promise. Wrap in Suspense for loading state, ErrorBoundary for errors.

### Implementation

```tsx
import { use, useMemo, Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

interface UserProfileProps {
  userPromise: Promise<User>;
}

function UserProfile({ userPromise }: UserProfileProps) {
  const user = use(userPromise);
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

interface UserPageProps {
  userId: string;
}

export function UserPage({ userId }: UserPageProps) {
  // Stable promise per userId
  const userPromise = useMemo(() => fetchUser(userId), [userId]);

  return (
    <ErrorBoundary
      fallbackRender={({ error, resetErrorBoundary }) => (
        <div>
          <p>Failed to load user: {error.message}</p>
          <button onClick={resetErrorBoundary}>Retry</button>
        </div>
      )}
    >
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### With cache

```tsx
const userCache = new Map<string, Promise<User>>();

function getUserPromise(id: string): Promise<User> {
  if (!userCache.has(id)) {
    userCache.set(id, fetchUser(id));
  }
  return userCache.get(id)!;
}

// Usage:
function UserPage({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userPromise={getUserPromise(userId)} />
    </Suspense>
  );
}
```

### Conditional `use()`

`use()` — yagona Hook bo'lib, **conditional ravishda chaqirish mumkin** (boshqa Hook'lar bunga ruxsat bermaydi). Ammo `useMemo` shu cheklovga bo'ysunadi — uni har doim top-level'da chaqirish kerak.

```tsx
function ConditionalProfile({ userId, showProfile }: { userId: string; showProfile: boolean }) {
  // useMemo top-level — har render'da chaqiriladi
  const userPromise = useMemo(() => fetchUser(userId), [userId]);

  if (showProfile) {
    const user = use(userPromise); // ✅ use() conditional ishlatish mumkin
    return <p>{user.name}</p>;
  }
  return <p>Hidden</p>;
}
```

### `use(Context)` — alternative to `useContext`

```tsx
import { use, createContext } from "react";

const ThemeContext = createContext<{ mode: "light" | "dark" }>({ mode: "light" });

function ThemedComponent() {
  const theme = use(ThemeContext); // ✅ Same as useContext
  return <div className={theme.mode}>...</div>;
}

// Conditional context read
function ConditionalContext({ shouldRead }: { shouldRead: boolean }) {
  if (shouldRead) {
    const theme = use(ThemeContext); // ✅ Conditional with use()
    return <div>{theme.mode}</div>;
  }
  return null;
}
```

### Common Mistakes

- ❌ **New promise each render**: Infinite suspend. Memoize.
- ❌ **No Suspense**: Throw uncaught.
- ❌ **No ErrorBoundary**: Rejection uncaught.
- ❌ **Calling fetch directly in render**: Side effect — wrap in `useMemo`.

</details>

---

## 19. `useOptimistic` — optimistic UI [Senior]

<details>
<summary><strong>Implementation</strong></summary>

### Description

R19 hook for optimistic UI updates. Show update immediately, revert on failure. Pairs with Server Actions.

### Implementation

```tsx
import { useOptimistic, useState, useTransition } from "react";

interface Comment {
  id: string;
  text: string;
  pending?: boolean;
}

async function postComment(text: string): Promise<Comment> {
  const res = await fetch("/api/comments", {
    method: "POST",
    body: JSON.stringify({ text }),
  });
  if (!res.ok) throw new Error("Failed");
  return res.json();
}

interface CommentListProps {
  initialComments: Comment[];
}

export function CommentList({ initialComments }: CommentListProps) {
  const [comments, setComments] = useState<Comment[]>(initialComments);
  const [isPending, startTransition] = useTransition();

  const [optimisticComments, addOptimisticComment] = useOptimistic<
    Comment[],
    string
  >(comments, (state, newCommentText) => [
    ...state,
    {
      id: `optimistic-${Date.now()}`,
      text: newCommentText,
      pending: true,
    },
  ]);

  const handleSubmit = async (formData: FormData) => {
    const text = formData.get("text") as string;
    if (!text) return;

    startTransition(async () => {
      addOptimisticComment(text); // Optimistic UI

      try {
        const newComment = await postComment(text);
        setComments((prev) => [...prev, newComment]);
      } catch (error) {
        console.error("Failed:", error);
        // Optimistic state auto-reverts on error
      }
    });
  };

  return (
    <div>
      <ul>
        {optimisticComments.map((c) => (
          <li key={c.id} style={{ opacity: c.pending ? 0.5 : 1 }}>
            {c.text}
            {c.pending && <span> (sending...)</span>}
          </li>
        ))}
      </ul>

      <form action={handleSubmit}>
        <input name="text" placeholder="New comment" />
        <button type="submit" disabled={isPending}>
          {isPending ? "Sending..." : "Send"}
        </button>
      </form>
    </div>
  );
}
```

### How it works

1. `useOptimistic(state, updateFn)` returns `[optimisticState, addOptimisticUpdate]`
2. Inside transition, call `addOptimisticUpdate(value)` → optimistic state updated
3. UI renders immediately with new state
4. When transition completes, optimistic state reverts (returns to base `state`)
5. Real `setState` reflects committed value

### Like/Unlike example

```tsx
import { useOptimistic, useTransition } from "react";

interface Post {
  id: string;
  liked: boolean;
  likeCount: number;
}

interface LikeButtonProps {
  post: Post;
  onLike: () => Promise<void>;
}

export function LikeButton({ post, onLike }: LikeButtonProps) {
  const [isPending, startTransition] = useTransition();

  const [optimisticPost, updateOptimistic] = useOptimistic<Post, void>(
    post,
    (state) => ({
      ...state,
      liked: !state.liked,
      likeCount: state.liked ? state.likeCount - 1 : state.likeCount + 1,
    }),
  );

  const handleClick = () => {
    startTransition(async () => {
      updateOptimistic();
      try {
        await onLike();
      } catch (err) {
        console.error(err); // Reverts auto
      }
    });
  };

  return (
    <button onClick={handleClick} disabled={isPending}>
      {optimisticPost.liked ? "❤️" : "🤍"} {optimisticPost.likeCount}
    </button>
  );
}
```

### Common Mistakes

- ❌ **Use outside transition**: `useOptimistic` requires `startTransition` or Server Actions.
- ❌ **Mutating state in update fn**: Update fn must return new state (immutable).
- ❌ **No error handling**: User can't tell if action succeeded — reflect status.

</details>

---

## 20. Custom `forwardRef` + `useImperativeHandle` [Middle+]

<details>
<summary><strong>Implementation</strong></summary>

### Description

Forward refs through wrapping components and expose imperative API.

**Note R19**: R19'da `ref` oddiy prop sifatida qabul qilinadi — `forwardRef` wrapper kerak emas. `forwardRef` **hozircha deprecated emas**, faqat kelajakdagi major versiyalarda deprecation rejalashtirilgan. Yangi kod uchun ref-as-prop pattern tavsiya etiladi.

### R19 native — ref as prop

```tsx
import { useImperativeHandle, useRef } from "react";

interface InputHandle {
  focus: () => void;
  blur: () => void;
  clear: () => void;
}

interface FancyInputProps {
  placeholder: string;
  ref?: React.Ref<InputHandle>; // R19 — ref as regular prop
}

function FancyInput({ placeholder, ref }: FancyInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(
    ref,
    () => ({
      focus: () => inputRef.current?.focus(),
      blur: () => inputRef.current?.blur(),
      clear: () => {
        if (inputRef.current) {
          inputRef.current.value = "";
        }
      },
    }),
    [],
  );

  return <input ref={inputRef} placeholder={placeholder} />;
}
```

### R18 — `forwardRef`

```tsx
import { forwardRef, useImperativeHandle, useRef } from "react";

interface InputHandle {
  focus: () => void;
  blur: () => void;
  clear: () => void;
}

interface FancyInputProps {
  placeholder: string;
}

const FancyInput = forwardRef<InputHandle, FancyInputProps>(
  function FancyInput({ placeholder }, ref) {
    const inputRef = useRef<HTMLInputElement>(null);

    useImperativeHandle(
      ref,
      () => ({
        focus: () => inputRef.current?.focus(),
        blur: () => inputRef.current?.blur(),
        clear: () => {
          if (inputRef.current) {
            inputRef.current.value = "";
          }
        },
      }),
      [],
    );

    return <input ref={inputRef} placeholder={placeholder} />;
  },
);
```

### Usage

```tsx
function App() {
  const inputRef = useRef<InputHandle>(null);

  return (
    <div>
      <FancyInput ref={inputRef} placeholder="Type here" />
      <button onClick={() => inputRef.current?.focus()}>Focus</button>
      <button onClick={() => inputRef.current?.clear()}>Clear</button>
    </div>
  );
}
```

### When to use

✅ **Use cases:**
- Wrapping native elements (focus, scroll)
- Modal/dialog programmatic control (`.open()`, `.close()`)
- Animation triggers
- Imperative library integration (Three.js, charts)

❌ **Avoid for:**
- State that should be props (declarative)
- Cross-cutting state (use Context)

### Variations

**Conditional API:**

```tsx
const Modal = forwardRef<ModalHandle, ModalProps>(
  function Modal({ children }, ref) {
    const [open, setOpen] = useState(false);

    useImperativeHandle(
      ref,
      () => ({
        open: () => setOpen(true),
        close: () => setOpen(false),
        toggle: () => setOpen((o) => !o),
        isOpen: () => open,
      }),
      [open], // Re-create when state changes
    );

    if (!open) return null;
    return <div role="dialog">{children}</div>;
  },
);
```

**Composing forwardRef + memo:**

```tsx
const FancyInput = memo(
  forwardRef<InputHandle, FancyInputProps>(
    function FancyInput({ placeholder }, ref) {
      // ...
    }
  )
);
```

### Common Mistakes

- ❌ **Exposing entire DOM node**: Defeats encapsulation. Expose specific methods.
- ❌ **Empty deps in `useImperativeHandle`**: When state inside, missing deps → stale.
- ❌ **Using ref for state**: Refs for imperative actions, state for data.

</details>

---

## 21. `useToggle` [Junior+]

<details>
<summary><strong>Yechim</strong></summary>

### Signature

```typescript
function useToggle(initialValue?: boolean): [boolean, () => void, (value: boolean) => void];
```

### Description

Boolean state'ni boshqaradi: 1) toggle (invert), 2) explicit set. Common: modal open/close, theme switch, expand/collapse.

### Implementation

```tsx
import { useState, useCallback } from "react";

function useToggle(initialValue: boolean = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue((v) => !v);
  }, []);

  return [value, toggle, setValue] as const;
}
```

### Usage

```tsx
function Modal() {
  const [isOpen, toggleOpen, setOpen] = useToggle(false);

  return (
    <>
      <button onClick={toggleOpen}>Toggle</button>
      <button onClick={() => setOpen(true)}>Force Open</button>
      <button onClick={() => setOpen(false)}>Force Close</button>
      {isOpen && <Dialog onClose={() => setOpen(false)} />}
    </>
  );
}
```

### Edge Cases

- **Functional toggle**: `setValue(v => !v)` — closure-immune.
- **Explicit set with `false`**: Use `setValue(false)`, not `setValue(undefined)`.
- **TypeScript tuple**: `as const` ensures `[boolean, () => void, (v: boolean) => void]` — not `Array<...>`.

### Common Mistakes

- ❌ `() => setValue(!value)` — stale closure.
- ❌ Returning array without `as const` — type widens.

</details>

---

## 22. `useClickOutside` [Middle]

<details>
<summary><strong>Yechim</strong></summary>

### Signature

```typescript
function useClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (e: MouseEvent | TouchEvent) => void
): void;
```

### Description

Element tashqarisida click bo'lganda callback chaqiradi. Common: dropdown close, popover dismiss, modal click-outside.

### Implementation

```tsx
import { useEffect, RefObject } from "react";

function useClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (e: MouseEvent | TouchEvent) => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      const el = ref.current;
      if (!el || el.contains(event.target as Node)) {
        return;
      }
      handler(event);
    };

    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);

    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}
```

### Usage

```tsx
function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useClickOutside(ref, () => setOpen(false));

  return (
    <div ref={ref}>
      <button onClick={() => setOpen(o => !o)}>Toggle</button>
      {open && <ul className="menu">...</ul>}
    </div>
  );
}
```

### Edge Cases

- **Stale handler closure**: Use `useRef` for latest handler if needed.
- **Touch events**: `touchstart` for mobile (some users tap-to-dismiss).
- **Portal'd content**: Portal child not in `ref.current.contains` — handle separately.

### Common Mistakes

- ❌ Using `click` event (fires after mousedown — cleanup won't prevent).
- ❌ Forgetting `touchstart` for mobile.

</details>

---

## 23. `useMediaQuery` [Middle+]

<details>
<summary><strong>Yechim</strong></summary>

### Signature

```typescript
function useMediaQuery(query: string): boolean;
```

### Description

CSS media query'ga reactive boolean. SSR-safe (default false). Common: responsive layouts, dark mode detection.

### Implementation

```tsx
import { useSyncExternalStore } from "react";

function useMediaQuery(query: string): boolean {
  return useSyncExternalStore(
    (callback) => {
      const mql = window.matchMedia(query);
      mql.addEventListener("change", callback);
      return () => mql.removeEventListener("change", callback);
    },
    () => window.matchMedia(query).matches,
    () => false  // SSR
  );
}
```

### Usage

```tsx
function ResponsiveLayout() {
  const isMobile = useMediaQuery("(max-width: 768px)");
  const isDark = useMediaQuery("(prefers-color-scheme: dark)");
  const reducedMotion = useMediaQuery("(prefers-reduced-motion: reduce)");

  return (
    <div className={isDark ? "theme-dark" : "theme-light"}>
      {isMobile ? <MobileNav /> : <DesktopNav />}
      {reducedMotion ? <StaticIcon /> : <AnimatedIcon />}
    </div>
  );
}
```

### Edge Cases

- **SSR initial render**: Returns false (server). Hydration may flicker if differs from client.
- **Old browsers**: `addEventListener` on MediaQueryList — older browsers use `addListener`.
- **Multiple queries**: Each `useMediaQuery` separate subscription.

### Common Mistakes

- ❌ Reading `window.matchMedia` without subscription — won't react to changes.
- ❌ Using `useEffect` (re-render cycle delay) — `useSyncExternalStore` better.

</details>

---

## 24. `useCopyToClipboard` [Middle]

<details>
<summary><strong>Yechim</strong></summary>

### Signature

```typescript
function useCopyToClipboard(): [
  copiedText: string | null,
  copy: (text: string) => Promise<boolean>
];
```

### Description

Text'ni clipboard'ga copy qiladi (modern Clipboard API). Returns: copied text + copy function. Common: "Copy link", "Copy code".

### Implementation

```tsx
import { useState, useCallback } from "react";

function useCopyToClipboard(): [string | null, (text: string) => Promise<boolean>] {
  const [copiedText, setCopiedText] = useState<string | null>(null);

  const copy = useCallback(async (text: string): Promise<boolean> => {
    if (!navigator?.clipboard) {
      console.warn("Clipboard not supported");
      return false;
    }

    try {
      await navigator.clipboard.writeText(text);
      setCopiedText(text);
      return true;
    } catch (error) {
      console.error("Copy failed:", error);
      setCopiedText(null);
      return false;
    }
  }, []);

  return [copiedText, copy];
}
```

### Usage

```tsx
function ShareButton({ url }: { url: string }) {
  const [copied, copy] = useCopyToClipboard();
  const [showFeedback, setShowFeedback] = useState(false);

  const handleClick = async () => {
    const success = await copy(url);
    if (success) {
      setShowFeedback(true);
      setTimeout(() => setShowFeedback(false), 2000);
    }
  };

  return (
    <button onClick={handleClick}>
      {showFeedback ? "Copied!" : "Copy link"}
    </button>
  );
}
```

### Edge Cases

- **HTTPS required**: Clipboard API requires secure context.
- **Permission**: Some browsers prompt user permission.
- **Fallback**: `document.execCommand("copy")` deprecated — modern Clipboard API only.

### Common Mistakes

- ❌ Not handling permission denial.
- ❌ Forgetting `await` — async API.

</details>

---

## 25. `useGeolocation` [Middle+]

<details>
<summary><strong>Yechim</strong></summary>

### Signature

```typescript
interface GeolocationState {
  loading: boolean;
  accuracy: number | null;
  latitude: number | null;
  longitude: number | null;
  error: GeolocationPositionError | null;
}

function useGeolocation(options?: PositionOptions): GeolocationState;
```

### Description

Browser Geolocation API'ga reactive wrapper. Watches position changes, cleanup on unmount.

### Implementation

```tsx
import { useEffect, useState } from "react";

function useGeolocation(options?: PositionOptions) {
  const [state, setState] = useState<GeolocationState>({
    loading: true,
    accuracy: null,
    latitude: null,
    longitude: null,
    error: null,
  });

  useEffect(() => {
    if (!navigator?.geolocation) {
      setState({
        loading: false,
        accuracy: null,
        latitude: null,
        longitude: null,
        error: { code: 2, message: "Geolocation not supported" } as any,
      });
      return;
    }

    const onSuccess = (position: GeolocationPosition) => {
      setState({
        loading: false,
        accuracy: position.coords.accuracy,
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
        error: null,
      });
    };

    const onError = (error: GeolocationPositionError) => {
      setState((s) => ({ ...s, loading: false, error }));
    };

    const watchId = navigator.geolocation.watchPosition(onSuccess, onError, options);

    return () => navigator.geolocation.clearWatch(watchId);
  }, [options]);

  return state;
}
```

### Usage

```tsx
function LocationDisplay() {
  const { loading, latitude, longitude, error } = useGeolocation();

  if (loading) return <Spinner />;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <p>
      Lat: {latitude?.toFixed(4)}, Lon: {longitude?.toFixed(4)}
    </p>
  );
}
```

### Edge Cases

- **HTTPS required**: Geolocation requires secure context.
- **Permission denial**: Error code 1 — user blocked.
- **Options object stability**: Pass via `useMemo` to prevent re-subscribe.

### Common Mistakes

- ❌ `getCurrentPosition` instead of `watchPosition` — only initial value.
- ❌ Not handling permission denial.

</details>

---

## Xulosa

20 ta majburiy implementation tugatildi:

- **Custom Hooks (1-7)**: useDebounce, useThrottle, usePrevious, useLocalStorage, useIntersectionObserver, useFetch, useEventListener
- **HOC va Class Patterns (8-10)**: withAuth, withErrorBoundary, ErrorBoundary class
- **Architectural (11-12)**: Compound Component, useReducer + Context
- **Performance (13-14)**: memo + useCallback, Virtualization
- **Library Patterns (15-17)**: useAsync, Suspense + lazy, Render Props
- **R19 Features (18-20)**: use() data, useOptimistic, forwardRef + useImperativeHandle

**Asosiy printsiplar:**

1. **Cleanup har joyda** — useEffect, AbortController, observer.disconnect
2. **SSR safety** — `typeof window !== "undefined"`, getServerSnapshot
3. **TypeScript types** — generics, interface props, return types
4. **Error handling** — try-catch, AbortError check, fallback UI
5. **Performance** — memoization, RAF throttle, useCallback stable refs
6. **Edge cases** — initial render, unmount, network failure, race conditions

**Keyingi fayl:** `08-react-19.md` — R19 specific APIs (Document hoisting, Web Components, RSC, Server Actions).

