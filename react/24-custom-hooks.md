# Bo'lim 24: Custom Hooks

> Custom hook — boshqa hook'lardan iborat bo'lgan, `use*` prefix bilan nomlangan oddiy JavaScript function. Logic'ni komponentdan ajratib, qayta ishlatish, test qilish va abstraction qilishga imkon beradi. Bu bo'lim custom hook'larning fundamental qoidalari, composition pattern'lari, common toolkit (10+ production-grade hook), `useDebugValue` DevTools integration va TypeScript generic pattern'larni qamrab oladi.

---

## Mundarija

- [Custom Hooks Nima va Nima Uchun](#custom-hooks-nima-va-nima-uchun)
- [`use*` Naming Convention va ESLint Identification](#use-naming-convention-va-eslint-identification)
- [Logic Extraction Pattern](#logic-extraction-pattern)
- [Hook Composition](#hook-composition)
- [Parameters va Return Types — Tuple vs Object](#parameters-va-return-types--tuple-vs-object)
- [`useDebugValue` — DevTools Display](#usedebugvalue--devtools-display)
- [Common Toolkit Overview](#common-toolkit-overview)
- [`usePrevious` — Previous Value Pattern](#useprevious--previous-value-pattern)
- [`useDebounce` va `useDebouncedCallback`](#usedebounce-va-usedebouncedcallback)
- [`useLocalStorage` — Persistent State (SSR-safe)](#uselocalstorage--persistent-state-ssr-safe)
- [`useMediaQuery` — Responsive Design](#usemediaquery--responsive-design)
- [`useWindowSize` — Window Dimensions](#usewindowsize--window-dimensions)
- [`useEventListener` — Event Subscription](#useeventlistener--event-subscription)
- [`useOnClickOutside` — Click Outside Detection](#useonclickoutside--click-outside-detection)
- [`useIntersectionObserver` — Visibility Detection](#useintersectionobserver--visibility-detection)
- [`useFetch` — Async Data (AbortController, race-safe)](#usefetch--async-data-abortcontroller-race-safe)
- [TypeScript Generic Patterns](#typescript-generic-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Custom Hooks Nima va Nima Uchun

### Nazariya

Custom hook — bu **`use*` prefix bilan nomlangan oddiy JavaScript function**, ichida boshqa React hook'lar (built-in yoki custom) chaqirilishi mumkin. Custom hook React'ning yangi feature'i emas — bu **konvensiya**. Hech qanday "registratsiya" yoki maxsus API yo'q. Faqat function yozasiz va `use` prefix bilan nomlaysiz.

```tsx
function useCounter(initial: number = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initial);
  return { count, increment, decrement, reset };
}
```

NIMA UCHUN custom hook kerak — uchta asosiy muammo hal qilinadi:

1. **Logic duplication** — bir xil logic bir nechta komponentda takrorlanadi (form validation, debounced search, localStorage sync). Custom hook bilan bir joyga olinadi.

2. **Mixed concerns** — komponent ichida UI logic + data fetching + side effects + state management aralash. Custom hook'lar concern'larni ajratadi (Single Responsibility Principle).

3. **Testability** — custom hook alohida test qilinishi mumkin (`@testing-library/react` `renderHook`). Komponent test'lari UI'ga, hook test'lari logic'ga fokuslanadi.

QANDAY ISHLAYDI: custom hook chaqirilganda, ichidagi built-in hook'lar **chaqiruvchi komponentning Hook linked list'iga** qo'shiladi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) Hook Indexing). Custom hook o'zining alohida state'i yo'q — u faqat **logic'ni qadoqlaydi**. Har komponent custom hook chaqirsa, **mustaqil hook instances** oladi (per-component state).

```tsx
function ComponentA() {
  const counter1 = useCounter(0);  // Komponent A'ning instance
  return <div>{counter1.count}</div>;
}

function ComponentB() {
  const counter2 = useCounter(10); // Komponent B'ning instance — alohida state
  return <div>{counter2.count}</div>;
}
```

Custom hook **state share qilmaydi** — har chaqiruv yangi state. Agar global state kerak bo'lsa, Context (cross-ref [`19-usecontext.md`](19-usecontext.md)) yoki state library (Zustand, Redux) ishlatiladi.

Custom hook va komponent farqi:

| Xususiyat | Komponent | Custom Hook |
|-----------|-----------|-------------|
| Naming | PascalCase (`Counter`) | camelCase + `use` prefix (`useCounter`) |
| Return | JSX (ReactElement) | Har qanday value (object, tuple, primitive) |
| Hook'lar ichida | ✅ Mumkin | ✅ Mumkin |
| `use*` linter rules | Yo'q | Ha (`react-hooks/rules-of-hooks`) |
| React.memo | ✅ Wrap qilinadi | ❌ Memoize qilinmaydi |
| Re-render | Re-render bo'ladi | "Re-render" tushunchasi yo'q (komponent re-render orqali bajariladi) |

NIMA UCHUN konvensiya muhim: ESLint plugin (`eslint-plugin-react-hooks`) **`use` prefix orqali** function'ni hook deb identify qiladi. `use` bilan boshlanmagan function'da `useState` chaqirilsa — qoida buzilgan (Rules of Hooks). React Compiler R19'da ham shu konvensiyaga tayanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Custom hook chaqirilganda Hook linked list lifecycle:

```
Component'da:
  ┌──────────────────────────┐
  │ function Profile() {     │
  │   useState(0)            │ ← Hook 1
  │   useCounter(5)          │ ← Custom hook
  │   useEffect(...)         │ ← Hook N (custom hook ichidagi hook'lardan keyin)
  │ }                        │
  └──────────────────────────┘

  useCounter ichida:
  ┌──────────────────────────┐
  │ function useCounter(i) { │
  │   useState(i)            │ ← Hook 2 (Profile'ning linked list'ida)
  │   useCallback(...)       │ ← Hook 3 (Profile'ning linked list'ida)
  │ }                        │
  └──────────────────────────┘

Profile'ning Fiber.memoizedState:
  Hook 1 (useState) → Hook 2 (useState from useCounter) → 
  Hook 3 (useCallback from useCounter) → Hook 4 (useEffect)
```

Custom hook **alohida memoizedState yaratmaydi** — chaqiruvchi komponentning Hook linked list'iga "inline" qo'shiladi. React internal'da custom hook va komponent farqlanmaydi (har ikkalasi function call).

`mountWorkInProgressHook` har built-in hook chaqiruvi uchun yangi Hook obyekt yaratadi va linked list'ga ulaydi. Custom hook ichida 5 ta hook bo'lsa — chaqiruvchi komponent'da 5 ta hook qo'shiladi.

Bu sabab — Rules of Hooks custom hook'ga ham qo'llanadi. Conditional `if (cond) useCounter()` chaqirilsa, Hook count o'zgaradi → state corruption yoki "Rendered fewer/more hooks" error.

ESLint identification logic (simplified):

```javascript
// eslint-plugin-react-hooks/src/RulesOfHooks.js
function isHookName(s) {
  if (__EXPERIMENTAL__) {
    return s === 'use' || /^use[A-Z0-9]/.test(s);
  }
  return /^use[A-Z0-9]/.test(s);
}
```

Regex `/^use[A-Z0-9]/` — `use` keyin **uppercase yoki raqam** shart. `useState` ✓, `usercount` ✗ (lowercase u dan keyin), `useCounter` ✓, `use2FA` ✓ (raqam). Plain `use` identifier — R19 `use()` hook (eski plugin'da `__EXPERIMENTAL__` flag bilan, R19 stable bilan default tanib olinadi).

React Compiler (1.0 stable 2025-oktyabr; React'ga bundle qilinmagan, alohida opt-in `babel-plugin-react-compiler`; React 17/18/19 mos) ham shu logic'ni ishlatadi (auto-memoization Rules of React asosida).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Custom hook'siz duplication — `useState` + `useEffect` har komponentda:

```tsx
// ❌ Duplication — har komponent bir xil pattern
function ProductSearch() {
  const [query, setQuery] = useState('');
  const [debouncedQuery, setDebouncedQuery] = useState('');
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedQuery(query), 300);
    return () => clearTimeout(timer);
  }, [query]);
  
  // ... use debouncedQuery
}

function UserSearch() {
  const [name, setName] = useState('');
  const [debouncedName, setDebouncedName] = useState('');
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedName(name), 300);
    return () => clearTimeout(timer);
  }, [name]);
  
  // ... use debouncedName
}
```

Custom hook bilan extract qilingan:

```tsx
// ✅ Reusable
function useDebounced<T>(value: T, delay: number = 300): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debounced;
}

function ProductSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounced(query, 300);
  // ... use debouncedQuery
}

function UserSearch() {
  const [name, setName] = useState('');
  const debouncedName = useDebounced(name, 300);
  // ... use debouncedName
}
```

Mustaqil instance per component:

```tsx
function useCounter(initial: number = 0) {
  const [count, setCount] = useState(initial);
  return { count, setCount };
}

function App() {
  const counter1 = useCounter(0);   // ← Mustaqil
  const counter2 = useCounter(100); // ← Mustaqil
  
  return (
    <>
      <button onClick={() => counter1.setCount(c => c + 1)}>
        A: {counter1.count}
      </button>
      <button onClick={() => counter2.setCount(c => c + 1)}>
        B: {counter2.count}
      </button>
    </>
  );
}
```

Counter A va B'ning state'lari mustaqil — bir-biriga ta'sir qilmaydi.

</details>

---

## `use*` Naming Convention va ESLint Identification

### Nazariya

`use*` prefix — custom hook'larning **majburiy konvensiyasi**. Bu nafaqat stilistik tanlov, balki **functional ahamiyatga ega**:

1. **ESLint plugin** (`eslint-plugin-react-hooks`) `use*` regex orqali function'ni hook deb identify qiladi va Rules of Hooks tekshiradi (top-level only, conditional taqiq).

2. **React Compiler** (1.0 stable 2025-oktyabr; React'ga bundle qilinmagan, alohida opt-in `babel-plugin-react-compiler`; React 17/18/19 mos) `use*` function'larni hook deb tanib, auto-memoization va Rules of React tekshiradi.

3. **Developer ergonomics** — kod o'qiganda `useCart()` chaqiruv darrov hook ekanligi ma'lum. Komponent'larda `<Cart />` (PascalCase JSX), hook'larda `useCart()` (camelCase function call).

Konvensiya qoidalari:

```tsx
// ✅ TO'G'RI
function useCounter() {}
function useFetchUser() {}
function useLocalStorage() {}
function useDebounce() {}

// ❌ NOTO'G'RI
function counter() {}            // use prefix yo'q
function userhook() {}            // use prefix lekin lowercase keyin
function fetchUser() {}           // hook bo'lsa, use prefix kerak
function UseCounter() {}          // PascalCase — komponent emas
function getUser() {}             // sodda function nomi (lekin hook bo'lsa, use kerak)
```

`use*` keyin **uppercase yoki raqam** shart (`/^use[A-Z0-9]/`). Sabab — `user`, `using`, `useless` singari oddiy so'zlar hook deb tanib olinmasin.

```tsx
// ESLint regex: /^use[A-Z0-9]/
function user() {}        // hook EMAS — `use` keyin lowercase
function useCounter() {}  // hook — `use` keyin uppercase C
function use2FA() {}      // hook — `use` keyin raqam (`2`)
```

**Anti-pattern: hook bo'lmagan function'ga `use` prefix berish:**

```tsx
// ❌ NOTO'G'RI — function ichida hook chaqirilmaydi
function useFormatPrice(price: number) {
  return `${price} so'm`;  // sodda formatter, hook emas
}

// ✅ TO'G'RI — sodda function
function formatPrice(price: number) {
  return `${price} so'm`;
}
```

`use` prefix faqat haqiqiy hook'lar uchun (ichida built-in hook chaqirilgan). Sabab — ESLint Rules of Hooks `use*` function'da static analysis qiladi va keraksiz warning beradi.

NIMA UCHUN bu strict bo'lishi muhim: hook'lar va sodda function'lar React lifecycle bilan boshqacha munosabatda bo'ladi. Hook'lar Fiber'ning memoizedState'ga bog'lanadi. Sodda function'lar har chaqiruvda toza ishlaydi. Konvensiya bilan kod o'qiganda darrov ajraladi.

> **Versiya evolyutsiyasi (Naming convention):**
> - **Pre-R16.8 (Hooks oldidan):** Konvensiya yo'q — function nomlari ixtiyoriy.
> - **R16.8+:** `use*` konvensiya kiritildi (Hooks RFC), `eslint-plugin-react-hooks` qoida tekshiradi.
> - **Compiler 1.0 stable (2025-oktyabr):** React Compiler `use*` ga tayanib auto-memoization qiladi (React 17/18/19 mos, alohida opt-in `babel-plugin-react-compiler`, React'ga bundle qilinmagan). Konvensiya endi build behavior'ga ta'sir qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ESLint Rules of Hooks plugin source code (simplified):

```javascript
// eslint-plugin-react-hooks/src/RulesOfHooks.js
const isHook = (node) => {
  if (node.type === 'Identifier') {
    return node.name === 'use' || /^use[A-Z0-9]/.test(node.name);
  }
  return false;
};

const isComponent = (node) => {
  // PascalCase — component
  return /^[A-Z]/.test(node.name);
};

// Linter har function'da hook chaqiruvlarni tekshiradi
function checkRulesOfHooks(node) {
  if (!isHook(node.callee) && !isComponent(node.callee)) {
    return; // Sodda function — Rules of Hooks qo'llanmaydi
  }
  
  // Conditional check
  if (isInsideConditional(node)) {
    report({ node, message: 'Hook conditional ichida chaqirilmaydi' });
  }
  
  // Loop check
  if (isInsideLoop(node)) {
    report({ node, message: 'Hook loop ichida chaqirilmaydi' });
  }
}
```

React Compiler (R17/18/19 bilan mos, opt-in Babel plugin) — `use*` nomli function'larni avtomatik memoize qiladi (Rules of React amal qilsa). Compiler analiz paytida custom hook'lar komponent bilan bir xil tartibda ko'rib chiqiladi.

```javascript
// React Compiler internal logic (simplified)
function shouldOptimize(fn) {
  if (isComponentName(fn.name)) return true;       // PascalCase
  if (isHookName(fn.name)) return true;            // use*
  return false;
}
```

Konvensiya buzilsa (hook'da `use*` yo'q) — Compiler optimize qilmaydi va manual memoization saqlanib qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Hook detection — ESLint qaysi function'larni tekshiradi:

```tsx
// ✅ Hook deb tanib olinadi — Rules of Hooks tekshiriladi
function useTimer(delay: number) {
  const [time, setTime] = useState(0);  // ✅ top-level
  
  useEffect(() => {                     // ✅ top-level
    const id = setInterval(() => setTime(t => t + 1), delay);
    return () => clearInterval(id);
  }, [delay]);
  
  return time;
}

// ❌ Hook deb tanib olinmaydi — `use` prefix yo'q
function timer(delay: number) {
  const [time, setTime] = useState(0);  // ESLint warning: hook regular function'da
  // ...
}

// ✅ Hook deb tanib olinadi va Rules tekshiriladi
function useConditionalState(enabled: boolean) {
  if (enabled) {
    const [v, setV] = useState(0);  // ❌ ESLint error: conditional hook
    return v;
  }
  return null;
}
```

PascalCase vs camelCase ajratish:

```tsx
// Komponent — PascalCase
function Counter() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}

// Hook — camelCase + use prefix
function useCounter() {
  const [count, setCount] = useState(0);
  return { count, setCount };
}

// Sodda function — camelCase, use prefix yo'q
function calculatePrice(items: Item[]) {
  return items.reduce((sum, i) => sum + i.price, 0);
}
```

Anti-pattern — `use` prefix lekin hook emas:

```tsx
// ❌ Anti-pattern — function ichida hook chaqiruv yo'q
function useFormatDate(date: Date): string {
  return date.toLocaleDateString();  // sodda formatter
}

// Bu function ichida useState yoki useEffect bo'lmasa, `use` prefix kerak emas

// ✅ To'g'ri nom
function formatDate(date: Date): string {
  return date.toLocaleDateString();
}
```

Edge case — `use` keyin raqam yoki `_`:

```tsx
// Regex /^use[A-Z0-9]/ — `use` keyin uppercase harf yoki raqam.
function use2FA() {}     // ✅ hook — `use` keyin raqam (`2`)
function use_state() {}  // ❌ underscore — hook deb tanilmaydi (uppercase/raqam yo'q)
function useID() {}      // ✅ uppercase ID
```

</details>

---

## Logic Extraction Pattern

### Nazariya

Logic extraction — komponent ichidagi logic'ni custom hook'ga ko'chirish jarayoni. Pattern uch qadamdan iborat:

1. **Identify** — komponent ichida takrorlanadigan yoki concern bo'yicha alohida logic'ni topish.
2. **Extract** — logic'ni `use*` function ichiga ko'chirish, parameters/return tuzish.
3. **Replace** — komponent ichida custom hook chaqiruv qoldirish, qolgan kodni o'chirish.

Refactoring misol — fetch logic'ni extract qilish:

```tsx
// ❌ Avval — komponent ichida fetch logic
function ProductPage({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    fetch(`/api/products/${id}`, { signal: controller.signal })
      .then(r => r.json())
      .then(data => {
        setProduct(data);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err);
          setLoading(false);
        }
      });
    return () => controller.abort();
  }, [id]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>{error.message}</div>;
  if (!product) return null;
  return <div>{product.name}</div>;
}
```

Extract qilingan:

```tsx
// ✅ Custom hook
function useFetch<T>(url: string): {
  data: T | null;
  loading: boolean;
  error: Error | null;
} {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    fetch(url, { signal: controller.signal })
      .then(r => r.json() as Promise<T>)
      .then(data => {
        setData(data);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err);
          setLoading(false);
        }
      });
    return () => controller.abort();
  }, [url]);
  
  return { data, loading, error };
}

// Komponent — toza
function ProductPage({ id }: { id: string }) {
  const { data: product, loading, error } = useFetch<Product>(`/api/products/${id}`);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>{error.message}</div>;
  if (!product) return null;
  return <div>{product.name}</div>;
}
```

NIMA UCHUN bu pattern muhim: **Single Responsibility Principle**. Komponent **UI**'ga fokuslanadi, custom hook **logic**'ga. Test qilish, refactor qilish, qayta ishlatish osonlashadi.

QANDAY ISHLAYDI: extract paytida diqqat qilinadigan jihatlar:

- **Parameters** — komponent ichidagi `id` propsi → `useFetch(url)` parameter
- **Return value** — komponent ichidagi state'lar → object qaytariladi
- **Dependencies** — `[id]` → `[url]` (parameter o'zgarsa, effect qayta chaqiriladi)
- **Cleanup** — komponentdagi `return () => controller.abort()` saqlanadi

Eng katta extraction xatosi — **ortiqcha extraction**. Agar logic faqat bir joyda ishlatilsa va kichik bo'lsa, custom hook abstraction overhead beradi (ko'proq fayllar, tushunish qiyinroq). Extraction qachon foyda beradi:

| Vaziyat | Extract qilish | Sabab |
|---------|----------------|-------|
| 2+ komponentda takrorlanadi | ✅ Ha | Reuse |
| Komponent 200+ qator | ✅ Ha | Maintainability |
| Concern alohida (fetch, debounce, storage) | ✅ Ha | Single Responsibility |
| Test qilish kerak | ✅ Ha | Hook'ni alohida test qilish oson |
| Bir joyda 10 qator | ❌ Yo'q | Premature abstraction |
| Component-specific UI logic | ❌ Yo'q | UI komponent'da qoladi |

<details>
<summary><strong>Under the Hood</strong></summary>

Extraction internal'da hech narsa o'zgartirmaydi — Hook linked list bir xil. Lekin debug paytida farq:

- DevTools'da custom hook ichidagi state custom hook nomi bilan ko'rinadi (`useDebugValue` qo'shilsa).
- Stack trace'da custom hook function name ko'rinadi.

Custom hook'da `useEffect` chaqirilsa, parent komponent'ning effect list'iga qo'shiladi. Cleanup parent unmount paytida chaqiriladi (komponent'dagi `useEffect` bilan bir xil lifecycle).

Memory cost — custom hook chaqiruv overhead ko'p emas (function call). Hook linked list strukturasi bir xil. React Compiler custom hook'ni komponent kabi alohida unit sifatida memoize qiladi (`use*` nomi orqali tanib oladi) — inline expand qilmaydi, hook'ning o'z body'si o'z scope'ida optimallashtiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Extraction misol — toggle pattern:

```tsx
// Avval — har joyda
function Sidebar() {
  const [isOpen, setIsOpen] = useState(false);
  const toggle = () => setIsOpen(prev => !prev);
  // ...
}

function Modal() {
  const [isOpen, setIsOpen] = useState(false);
  const toggle = () => setIsOpen(prev => !prev);
  // ...
}

// Extract qilingan
function useToggle(initial: boolean = false): [boolean, () => void, (v: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle, setValue];
}

function Sidebar() {
  const [isOpen, toggle] = useToggle();
  // ...
}

function Modal() {
  const [isOpen, toggle, setIsOpen] = useToggle();
  // ...
}
```

Extraction misol — form input handling:

```tsx
// Avval
function ContactForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [message, setMessage] = useState('');
  // ...
}

// Extract qilingan
function useFormFields<T extends Record<string, string>>(initial: T) {
  const [fields, setFields] = useState<T>(initial);
  
  const handleChange = useCallback(
    <K extends keyof T>(field: K) => (
      e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>
    ) => {
      setFields(prev => ({ ...prev, [field]: e.target.value }));
    },
    []
  );
  
  const reset = useCallback(() => setFields(initial), [initial]);
  
  return { fields, handleChange, reset, setFields };
}

function ContactForm() {
  const { fields, handleChange, reset } = useFormFields({
    name: '',
    email: '',
    message: '',
  });
  
  return (
    <form>
      <input value={fields.name} onChange={handleChange('name')} />
      <input value={fields.email} onChange={handleChange('email')} />
      <textarea value={fields.message} onChange={handleChange('message')} />
    </form>
  );
}
```

Extraction misol — navigation event:

```tsx
// Avval
function PageA() {
  useEffect(() => {
    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      e.returnValue = '';
    };
    window.addEventListener('beforeunload', handler);
    return () => window.removeEventListener('beforeunload', handler);
  }, []);
  // ...
}

// Extract qilingan
function useBeforeUnload(when: boolean = true, message?: string) {
  useEffect(() => {
    if (!when) return;
    
    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      e.returnValue = message ?? '';
    };
    
    window.addEventListener('beforeunload', handler);
    return () => window.removeEventListener('beforeunload', handler);
  }, [when, message]);
}

function PageA({ hasUnsavedChanges }: { hasUnsavedChanges: boolean }) {
  useBeforeUnload(hasUnsavedChanges);
  // ...
}
```

</details>

---

## Hook Composition

### Nazariya

Hook composition — custom hook'lar ichida boshqa custom hook'lar chaqirish. Bu **layered architecture** yaratadi — past darajadagi hook'lar primitive logic, yuqori darajadagilar business logic.

```tsx
// Layer 1: Primitive hook
function useFetch<T>(url: string) { /* basic fetch */ }

// Layer 2: Domain-specific hook
function useUser(userId: string) {
  return useFetch<User>(`/api/users/${userId}`);
}

// Layer 3: Feature-specific hook
function useCurrentUser() {
  const session = useSession();
  return useUser(session.userId);
}

// Layer 4: Page-specific hook
function useDashboard() {
  const user = useCurrentUser();
  const posts = useUserPosts(user.data?.id);
  const notifications = useNotifications();
  return { user, posts, notifications };
}
```

NIMA UCHUN bu pattern foydali: **abstraction layers**. Har daraja o'z mas'uliyatiga ega:

- `useFetch` — universal HTTP wrapper
- `useUser` — domain object retrieval
- `useCurrentUser` — auth context bilan integration
- `useDashboard` — page composition

Komponent faqat top-level hook'ni chaqiradi, ichki layerlar yashirin.

QANDAY ISHLAYDI: hook composition Hook linked list'ni **flat** qiladi. Bir komponent 4 layer custom hook chaqirsa, internal'da hook'lar barchasi shu komponentning linked list'iga qo'shiladi.

```
Komponent: useDashboard()
  └─ useCurrentUser()
      ├─ useSession()           ← Hook 1 (komponent linked list)
      └─ useUser()
          └─ useFetch()
              ├─ useState        ← Hook 2
              ├─ useState        ← Hook 3
              ├─ useState        ← Hook 4
              └─ useEffect       ← Hook 5
  └─ useUserPosts()
      └─ useFetch()
          ├─ useState            ← Hook 6
          ├─ ...
```

Hook count o'sadi (custom hook ichidagi har built-in hook qo'shiladi), lekin **flat structure** Rules of Hooks invariantlarini saqlaydi (top-level only, conditional taqiq).

**Composition cheklov**: hook'lar **conditional chaqirilmaydi**. Bu custom hook'larga ham qo'llaniladi:

```tsx
// ❌ Anti-pattern
function useConditionalUser(enabled: boolean) {
  if (enabled) {
    return useUser('123');  // ❌ conditional custom hook
  }
  return null;
}

// ✅ To'g'ri — custom hook har doim chaqiriladi, ichida conditional logic
function useUser(userId: string | null) {
  return useFetch<User>(userId ? `/api/users/${userId}` : null);
}

function useFetch<T>(url: string | null) {
  const [data, setData] = useState<T | null>(null);
  
  useEffect(() => {
    if (!url) return;  // ✅ conditional ichida useEffect, hook emas
    fetch(url).then(/* ... */);
  }, [url]);
  
  return data;
}
```

Pattern: **`null` parameter qabul qilish** custom hook'larda — disable holatini tabiiy qilish.

<details>
<summary><strong>Under the Hood</strong></summary>

Hook composition Fiber memoizedState linked list:

```javascript
// useDashboard ichida 3 ta custom hook chaqirilgan
function useDashboard() {
  const user = useCurrentUser();      // ← useSession + useUser + useFetch
  const posts = useUserPosts(...);    // ← useFetch
  const notifs = useNotifications();  // ← useFetch + useEffect
  return { user, posts, notifs };
}

// Fiber.memoizedState (linked list, simplified):
//
//   Hook 1 (useSession state)
//   Hook 2 (useUser useFetch state)
//   Hook 3 (useUser useFetch loading)
//   Hook 4 (useUser useFetch error)
//   Hook 5 (useUser useFetch useEffect)
//   Hook 6 (useUserPosts useFetch state)
//   Hook 7 (useUserPosts useFetch loading)
//   Hook 8 (useUserPosts useFetch error)
//   Hook 9 (useUserPosts useFetch useEffect)
//   Hook 10 (useNotifications useFetch state)
//   ... va h.k.
```

Linked list flat — composition layer'lari React internal'da farqlanmaydi. Har built-in hook chaqiruv yangi Hook obyekt qo'shadi.

Performance jihatdan composition overhead minimal — function call'lar engine tomonidan inline'lanadi (V8 optimization). Memory cost custom hook'lar soniga emas, **built-in hook'lar soniga** bog'liq.

React Compiler har custom hook'ni alohida unit sifatida memoize qiladi — `useDashboard`, `useCurrentUser`, `useFetch` har biri o'z body'sida optimallashtiriladi, komponent function ichiga ko'chirilmaydi. Composition tuzilishi build'dan keyin ham saqlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Layered composition misol:

```tsx
// Layer 1: Primitive
function useLocalStorage<T>(key: string, initial: T): [T, (v: T) => void] {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initial;
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) as T : initial;
  });
  
  const update = useCallback((newValue: T) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  }, [key]);
  
  return [value, update];
}

// Layer 2: Domain — user preferences
interface UserPreferences {
  theme: 'light' | 'dark';
  language: string;
  fontSize: number;
}

function useUserPreferences(): [UserPreferences, (prefs: Partial<UserPreferences>) => void] {
  const [prefs, setPrefs] = useLocalStorage<UserPreferences>('user-preferences', {
    theme: 'light',
    language: 'uz',
    fontSize: 14,
  });
  
  const update = useCallback((updates: Partial<UserPreferences>) => {
    setPrefs({ ...prefs, ...updates });
  }, [prefs, setPrefs]);
  
  return [prefs, update];
}

// Layer 3: Feature — theme management
function useTheme() {
  const [prefs, updatePrefs] = useUserPreferences();
  
  const toggleTheme = useCallback(() => {
    updatePrefs({ theme: prefs.theme === 'light' ? 'dark' : 'light' });
  }, [prefs.theme, updatePrefs]);
  
  useEffect(() => {
    document.documentElement.dataset.theme = prefs.theme;
  }, [prefs.theme]);
  
  return { theme: prefs.theme, toggleTheme };
}

// Komponent — top-level layer ishlatadi
function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button onClick={toggleTheme}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  );
}
```

Composition with multiple hooks — dashboard:

```tsx
function useDashboard(userId: string) {
  const profile = useFetchUser(userId);
  const stats = useFetchStats(userId);
  const activity = useFetchActivity(userId);
  
  const isLoading = profile.loading || stats.loading || activity.loading;
  const error = profile.error || stats.error || activity.error;
  
  return {
    profile: profile.data,
    stats: stats.data,
    activity: activity.data,
    isLoading,
    error,
  };
}

function Dashboard({ userId }: { userId: string }) {
  const { profile, stats, activity, isLoading, error } = useDashboard(userId);
  
  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!profile) return null;
  
  return (
    <div>
      <ProfileCard user={profile} />
      <StatsCard stats={stats} />
      <ActivityCard activity={activity} />
    </div>
  );
}
```

</details>

---

## Parameters va Return Types — Tuple vs Object

### Nazariya

Custom hook return type — qaytarilgan qiymat strukturasi. Ikkita asosiy convention:

1. **Tuple** — `[value, action]` array sifatida
2. **Object** — `{ value, action }` named property'lar bilan

Konvensiya tanlash community guideline'iga asoslangan:

| Return value soni | Convention | Misol |
|-------------------|------------|-------|
| 1 | Single value | `useMediaQuery() → boolean` |
| 2 | Tuple | `useState() → [value, setter]` |
| 3+ | Object | `useFetch() → { data, loading, error }` |

NIMA UCHUN tuple 2 ta uchun: **destructuring rename oson**:

```tsx
// Tuple — har joyda alohida nom
const [count, setCount] = useCounter(0);
const [isOpen, setIsOpen] = useToggle();
const [user, setUser] = useState<User | null>(null);

// Tuple bilan komponentda 2 ta useCounter — collision yo'q
function App() {
  const [aCount, setACount] = useCounter(0);
  const [bCount, setBCount] = useCounter(10);
}
```

NIMA UCHUN object 3+ uchun: **rename verbose**, **partial destructuring** mumkin:

```tsx
// Object — partial destructuring
const { data, loading } = useFetch<User>('/api/user');  // error ignore

// Object — har property'ning maqsadi nomdan ravshan
const { count, increment, decrement, reset } = useCounter(0);
// vs tuple [count, increment, decrement, reset] — har element pozitsion
```

QANDAY ISHLAYDI — TypeScript jihatdan:

```tsx
// Tuple type — 2-element
function useToggle(): [boolean, () => void] {
  const [value, setValue] = useState(false);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];  // tuple literal
}

// Object type — interface
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  // ...
  return { data, loading, error };
}
```

`as const` — tuple type aniq qilish:

```tsx
function useToggle() {
  const [value, setValue] = useState(false);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle] as const;  // ← `as const` muhim
}

// Ushbu tuple bilan TS readonly tuple infer qiladi:
// readonly [boolean, () => void]
const [v, t] = useToggle();
```

`as const` siz TypeScript `(boolean | (() => void))[]` infer qiladi (loose union array). `as const` bilan strict tuple type.

**Parameters** uchun pattern'lar:

```tsx
// Single parameter
function useMediaQuery(query: string): boolean {}

// Multiple parameters
function useDebounce<T>(value: T, delay: number): T {}

// Object parameter (3+ option)
function useFetch<T>(options: {
  url: string;
  method?: 'GET' | 'POST';
  body?: unknown;
  enabled?: boolean;
}): UseFetchResult<T> {}

// Optional with defaults
function useCounter(initial: number = 0, step: number = 1) {}
```

3+ parameter — object majmuasi (named arguments). Optional parameter'lar default qiymat bilan.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript tuple inference va object inference farqi:

```tsx
// Tuple
function useReturnTuple(): [number, string] {
  return [42, 'hello'];
}

// Compiled type:
// (): [number, string]

const result = useReturnTuple();
// result: [number, string]
const [num, str] = result;
// num: number
// str: string

// Object
function useReturnObject(): { count: number; label: string } {
  return { count: 42, label: 'hello' };
}

// Compiled type:
// (): { count: number; label: string }

const result = useReturnObject();
// result: { count: number; label: string }
const { count, label } = result;
// count: number
// label: string
```

Object literal `{ count, label }` har render'da yangi reference. Custom hook return value React.memo bilan ishlatilsa, har gal "yangi value" ko'rinadi → consumer re-render. Yechim: `useMemo` bilan return value memoize:

```tsx
function useCounter(initial: number) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  
  return useMemo(
    () => ({ count, increment, decrement }),
    [count, increment, decrement]
  );
}
```

Tuple ham bir xil muammo — har gal yangi array literal. Lekin tuple destructuring bilan komponent faqat element'larni saqlaydi (object'ning butun reference'i emas).

React Compiler bu muammoni hal qiladi (auto-memoization, manual `useMemo` kerak emas).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Single value:

```tsx
function useIsMobile(): boolean {
  const [isMobile, setIsMobile] = useState(false);
  
  useEffect(() => {
    const mql = window.matchMedia('(max-width: 768px)');
    setIsMobile(mql.matches);
    
    const handler = (e: MediaQueryListEvent) => setIsMobile(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, []);
  
  return isMobile;
}

// Ishlatish
function Header() {
  const isMobile = useIsMobile();
  return isMobile ? <MobileHeader /> : <DesktopHeader />;
}
```

Tuple — 2 element:

```tsx
function useToggle(initial: boolean = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Ishlatish — har joyda alohida nom
function Sidebar() {
  const [isOpen, toggleSidebar] = useToggle();
  return (
    <aside style={{ display: isOpen ? 'block' : 'none' }}>
      <button onClick={toggleSidebar}>Close</button>
    </aside>
  );
}

function Modal() {
  const [isVisible, toggleModal] = useToggle();
  return (
    <div style={{ display: isVisible ? 'block' : 'none' }}>
      <button onClick={toggleModal}>Close</button>
    </div>
  );
}
```

Object — 3+ properties:

```tsx
interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  set: (value: number) => void;
}

function useCounter(initial: number = 0): UseCounterReturn {
  const [count, setCount] = useState(initial);
  
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initial), [initial]);
  const set = useCallback((value: number) => setCount(value), []);
  
  return { count, increment, decrement, reset, set };
}

// Ishlatish — partial destructuring OK
function CounterButton() {
  const { count, increment } = useCounter(0);
  return <button onClick={increment}>{count}</button>;
}

function CounterFull() {
  const { count, increment, decrement, reset } = useCounter(0);
  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

Hybrid — 2 element ko'p ishlatilsa, lekin qo'shimcha helper'lar:

```tsx
// Tuple bo'lsa ham, helper'lar object'ga
type UseFormReturn<T> = [
  values: T,
  helpers: {
    setField: <K extends keyof T>(field: K, value: T[K]) => void;
    reset: () => void;
    handleSubmit: (e: React.FormEvent) => void;
  }
];

function useForm<T extends Record<string, any>>(initial: T): UseFormReturn<T> {
  const [values, setValues] = useState<T>(initial);
  
  const setField = useCallback(<K extends keyof T>(field: K, value: T[K]) => {
    setValues(v => ({ ...v, [field]: value }));
  }, []);
  
  const reset = useCallback(() => setValues(initial), [initial]);
  
  const handleSubmit = useCallback((e: React.FormEvent) => {
    e.preventDefault();
    console.log('Submit:', values);
  }, [values]);
  
  return [values, { setField, reset, handleSubmit }];
}

// Ishlatish
function ContactForm() {
  const [values, { setField, reset, handleSubmit }] = useForm({
    name: '',
    email: '',
  });
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={values.name}
        onChange={(e) => setField('name', e.target.value)}
      />
      {/* ... */}
    </form>
  );
}
```

</details>

---

## `useDebugValue` — DevTools Display

### Nazariya

`useDebugValue` — custom hook ichida ishlatiladigan **DevTools-only hook**. React DevTools'da custom hook nomi yonida qo'shimcha label/value ko'rsatadi. Production'da no-op (ishlamaydi).

API signature:

```tsx
function useDebugValue<T>(value: T, format?: (value: T) => unknown): void;
```

Argumentlar:
- `value` — DevTools'da ko'rsatiladigan qiymat (string, number, object, va h.k.)
- `format` — optional formatter function (lazy — faqat DevTools open paytida chaqiriladi)

Ishlatish:

```tsx
function useUserStatus(userId: string) {
  const [status, setStatus] = useState<'online' | 'offline' | 'idle'>('offline');
  
  useEffect(() => {
    // ... subscription logic
  }, [userId]);
  
  useDebugValue(status);  // DevTools: "useUserStatus: online"
  
  return status;
}
```

DevTools'da:

```
ProfileCard
  ┌─ useUserStatus: "online"
  ├─ useState: ...
  └─ useEffect: ...
```

NIMA UCHUN: complex custom hook'lar debug qilinishi qiyin. DevTools'da default'da `useState`, `useEffect` raw values ko'rinadi. `useDebugValue` bilan **semantic label** qo'shiladi.

QANDAY ISHLAYDI: `useDebugValue` Hook linked list'ga slot qo'shmaydi va `fiber.memoizedState`'ni o'zgartirmaydi — boshqa hook'lardan farqli. Render natijasiga ta'sir qilmaydi. DEV build'da React DevTools custom hook'ni inspeksiya qilganda `useDebugValue`'ga uzatilgan qiymatni ushlab, "Components" panelida custom hook nomi yonida ko'rsatadi (mexanizm Under the Hood'da). Production build'da bo'sh function.

Lazy formatter — production performance:

```tsx
function useUserData(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  
  // Format function faqat DevTools ochiq bo'lsa chaqiriladi
  useDebugValue(user, (u) => u ? `${u.name} (${u.email})` : 'Loading');
  
  return user;
}
```

Production build'da `useDebugValue` butunlay olib tashlanadi (dead code elimination). Development bundle'da format function lazy — DevTools ochilmasa, `format(user)` ishlamaydi (expensive calculation skip).

**Eng muhim qoida:** `useDebugValue` faqat **custom hook ichida** chaqirilishi kerak. Komponent ichida chaqirilsa — silent no-op (xato bermaydi, lekin DevTools'da ko'rinmaydi).

```tsx
// ❌ Anti-pattern — komponent ichida
function CounterPanel() {
  const [count, setCount] = useState(0);
  useDebugValue(count);  // DevTools'da ko'rinmaydi
  return <div>{count}</div>;
}

// ✅ Custom hook ichida
function useCounter(initial: number) {
  const [count, setCount] = useState(initial);
  useDebugValue(count);  // DevTools'da "useCounter: 42"
  return [count, setCount] as const;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`react-reconciler/src/ReactFiberHooks.js`'da `mountDebugValue` (va unga teng `updateDebugValue`) bo'sh function — Hook linked list'ga slot **qo'shmaydi**, `fiber.memoizedState`'ni o'zgartirmaydi:

```javascript
// react-reconciler/src/ReactFiberHooks.js (soddalashtirilgan)
function mountDebugValue(value, formatterFn) {
  // Bu hook odatda no-op — fiber state'ga hech narsa yozmaydi.
}
const updateDebugValue = mountDebugValue;
```

`useState`/`useEffect` `mountWorkInProgressHook()` chaqirib yangi Hook obyekt yaratadi va linked list'ga ulaydi; `useDebugValue` esa bunday qilmaydi. Shu sababli `useDebugValue` chaqiruvi DEV va PROD'da Hook count'ni o'zgartirmaydi — boshqa hook'larning index'iga ta'sir qilmaydi.

DEV build'da dispatcher (`HooksDispatcherOnMountInDEV` / `...OnUpdateInDEV`) `useDebugValue` ni yuqoridagi `mountDebugValue` / `updateDebugValue` ga map qiladi (DEV warning wrapper bilan). Bu base implementation argumentlarni saqlamaydi — DevTools qiymatni boshqa yo'l bilan oladi.

DevTools integration:

React DevTools custom hook'lar tarkibini ko'rsatish uchun hook function'ni **o'zi qayta render qiladi** (`react-debug-tools` paketidagi `ReactDebugHooks`). Bu render paytida DevTools maxsus dispatcher o'rnatadi; `useDebugValue(value, format)` chaqirilganda dispatcher `value`'ni (DevTools ochiq bo'lsa `format(value)`) ushlab oladi va custom hook nomi yonida ko'rsatadi. Qiymat `fiber.memoizedState`'da emas, DevTools'ning inspeksiya o'tishida yig'iladi.

Production build'da `useDebugValue` bo'sh function — `babel-plugin-react-compiler` yoki bundler dead code elimination orqali chaqiruv olib tashlanishi mumkin. Format function DEV'da ham faqat DevTools panel ochiq bo'lsa chaqiriladi (lazy) — expensive serialization bekorga ishlamaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda label:

```tsx
function useFriendStatus(friendId: string) {
  const [isOnline, setIsOnline] = useState(false);
  
  useEffect(() => {
    // subscription
  }, [friendId]);
  
  useDebugValue(isOnline ? 'Online' : 'Offline');
  
  return isOnline;
}

// DevTools:
// useFriendStatus: "Online"
```

Lazy formatter (expensive calculation):

```tsx
function useDataParser<T>(rawData: string) {
  const parsed = useMemo(() => JSON.parse(rawData) as T, [rawData]);
  
  // Format faqat DevTools ochiq bo'lsa chaqiriladi
  useDebugValue(parsed, (data) => 
    JSON.stringify(data, null, 2)  // expensive operation
  );
  
  return parsed;
}
```

Multiple debug values:

```tsx
function useChatRoom(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  useDebugValue({
    room: roomId,
    messages: messages.length,
    connected: isConnected,
    error: error?.message ?? null,
  });
  
  return { messages, isConnected, error };
}

// DevTools:
// useChatRoom: { room: "general", messages: 42, connected: true, error: null }
```

Production-grade hook with debug:

```tsx
interface UseFetchOptions {
  url: string;
  enabled?: boolean;
}

function useFetch<T>(options: UseFetchOptions) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const status = error ? 'error' : loading ? 'loading' : data ? 'success' : 'idle';
  
  useDebugValue(status, (s) => `${s.toUpperCase()}: ${options.url}`);
  
  // ... fetch logic
  
  return { data, loading, error, status };
}

// DevTools:
// useFetch: "SUCCESS: /api/users/123"
```

</details>

---

## Common Toolkit Overview

### Nazariya

Common custom hooks toolkit — production'da keng tarqalgan hook'lar to'plami. Har bir hook bitta concern hal qiladi va boshqa hook'lar bilan composition qilinishi mumkin.

Asosiy toolkit jadval:

| Hook | Maqsad | Built-in hook'lar |
|------|--------|-------------------|
| `usePrevious` | Avvalgi qiymatni saqlash | `useRef`, `useEffect` |
| `useDebounce` | Value debounce | `useState`, `useEffect` |
| `useDebouncedCallback` | Function debounce | `useRef`, `useCallback` |
| `useThrottle` | Value throttle | `useState`, `useEffect`, `useRef` |
| `useLocalStorage` | Persistent state | `useState`, `useEffect`, `useCallback` |
| `useSessionStorage` | Session-scope state | `useState`, `useEffect`, `useCallback` |
| `useMediaQuery` | Responsive breakpoint | `useSyncExternalStore` (R18+) |
| `useWindowSize` | Window dimensions | `useState`, `useEffect` |
| `useEventListener` | Event subscription | `useEffect`, `useRef` |
| `useOnClickOutside` | Click outside detection | `useEffect`, `useRef` |
| `useIntersectionObserver` | Visibility detection | `useState`, `useEffect`, `useRef` |
| `useFetch` | Async data | `useState`, `useEffect`, `useRef` |
| `useToggle` | Boolean state | `useState`, `useCallback` |
| `useCounter` | Counter state | `useState`, `useCallback` |
| `useHover` | Hover state | `useState`, `useEffect` |
| `useFocus` | Focus state | `useState`, `useEffect` |
| `useKeyPress` | Keyboard shortcut | `useEffect`, `useState` |

Bu hook'larning ko'pchiligi **library**larda mavjud (`react-use`, `usehooks-ts`, `@uidotdev/usehooks`, `ahooks`). Lekin bu library'larga to'liq tayanmaslik tavsiya qilinadi — har hook implementation kichik, edge case'lar (SSR, cleanup) loyihaga xos.

NIMA UCHUN o'zingiz yozish: full control, bundle size kichik, type-safety to'liq, edge case'lar loyiha bilan moslashgan. Library hook'lari "umumiy" — sizning use case'ga 80% mos kelishi mumkin (qolgan 20% — workaround yoki library replace).

Library tanlash kriteriya:

| Faktor | O'zingiz yozish | Library |
|--------|-----------------|---------|
| Hook'lar soni | Kam | Ko'p (collection) |
| TypeScript | Custom types | Library types |
| Bundle | Minimal (faqat kerakli) | Library overhead |
| Update freq | Loyiha bilan | Library version |
| Edge cases | Loyihaga xos | Generic |

**Bu bo'limda yoziladigan hook'lar production-grade** — to'liq TypeScript, SSR-safe, cleanup, race-condition free. Har birini loyihaga `lib/hooks/` papkaga ko'chirish mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Custom hooks library bundle size taqqoslash (taxminiy):

Library'lar bundle hajmi va hook count'i versiyalar bo'yicha o'zgarib turadi. Aniq raqamlar uchun bundlephobia.com tekshiring.

Tree-shaking — modern library'larda muhim. `usehooks-ts`, `@uidotdev/usehooks` per-hook export bilan tree-shake qilinadi (faqat ishlatilgan hook'lar bundle'ga kiradi). `react-use` library'sida tree-shaking inconsistent — barrel import bilan butun library yuklanishi mumkin.

Ko'pchilik library `useSyncExternalStore` (R18+) ishlatadi (Concurrent-safe). Eski library'lar `useEffect` bilan subscribe qiladi — tearing potential R18 Concurrent rendering'da (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Toolkit organize qilish — `lib/hooks/` papka strukturasi:

```
src/
├── lib/
│   └── hooks/
│       ├── index.ts                  // re-export
│       ├── usePrevious.ts
│       ├── useDebounce.ts
│       ├── useLocalStorage.ts
│       ├── useMediaQuery.ts
│       ├── useEventListener.ts
│       ├── useOnClickOutside.ts
│       ├── useIntersectionObserver.ts
│       └── useFetch.ts
└── components/
    └── ...
```

`index.ts`:

```tsx
export { usePrevious } from './usePrevious';
export { useDebounce, useDebouncedCallback } from './useDebounce';
export { useLocalStorage } from './useLocalStorage';
export { useMediaQuery } from './useMediaQuery';
export { useEventListener } from './useEventListener';
export { useOnClickOutside } from './useOnClickOutside';
export { useIntersectionObserver } from './useIntersectionObserver';
export { useFetch } from './useFetch';
```

Komponent ichida ishlatish:

```tsx
import { useDebounce, useLocalStorage } from '@/lib/hooks';

function SearchPage() {
  const [query, setQuery] = useLocalStorage('search-query', '');
  const debouncedQuery = useDebounce(query, 300);
  
  // ... search logic
}
```

</details>

---

## `usePrevious` — Previous Value Pattern

### Nazariya

`usePrevious` — komponentning avvalgi render'idagi qiymatni saqlaydi. JavaScript'da function call'lar mustaqil — avvalgi argument'larga to'g'ridan-to'g'ri kirish yo'q. React'da bu pattern `useRef` + `useEffect` bilan qurilgan.

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;  // har render'dan keyin yangilanadi
  }, [value]);
  
  return ref.current;  // hozirgi render'da eski qiymat
}
```

QANDAY ISHLAYDI:

1. **Render N**: `ref.current` eski qiymat (initial `undefined`).
2. **Render natijasi qaytariladi** — `usePrevious` `ref.current` (eski) qaytaradi.
3. **`useEffect` ishlaydi** — `ref.current = value` (yangi).
4. **Render N+1**: `ref.current` endi N-render'dagi qiymat — return qilinadi.

Bu **bir cycle delay** — har render'da hook **avvalgi** qiymat qaytaradi.

NIMA UCHUN bu kerak: state o'zgarishini detect qilish, animation trigger, transition, comparison logic.

```tsx
function Counter({ count }: { count: number }) {
  const prevCount = usePrevious(count);
  const direction = count > (prevCount ?? 0) ? 'up' : 'down';
  
  return (
    <div>
      <span>{count}</span>
      <span className={`indicator ${direction}`}>
        {direction === 'up' ? '↑' : '↓'}
      </span>
    </div>
  );
}
```

Birinchi render'da `prevCount` `undefined` — null check yoki default qiymat.

<details>
<summary><strong>Under the Hood</strong></summary>

`usePrevious` mexanizmi — `useEffect` lifecycle bilan bog'liq:

```
Render 1: count = 0
  ├─ ref.current = undefined (initial)
  ├─ return undefined (prevCount)
  ├─ useEffect (after commit): ref.current = 0
  
Render 2: count = 1
  ├─ ref.current = 0 (avvalgi render'dan)
  ├─ return 0 (prevCount)
  ├─ useEffect: ref.current = 1

Render 3: count = 5
  ├─ ref.current = 1
  ├─ return 1 (prevCount)
  ├─ useEffect: ref.current = 5
```

`useEffect` Commit Phase'dan keyin paint phase'gacha async chaqirilmaydi — passive effect (cross-ref [`16-useeffect.md`](16-useeffect.md)). Bu sabab — render'da `ref.current` hali eski qiymat.

Alternative pattern — `useEffect` o'rniga sync `useLayoutEffect`:

```tsx
function usePreviousSync<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useLayoutEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}
```

Hech qanday vizual farq yo'q — chunki `usePrevious` natijasi render output'iga ta'sir qilmaydi (sodda qiymat qaytarish). `useEffect` standart variant.

R18+ Strict Mode 2x effect cycle'da `usePrevious` to'g'ri ishlaydi — cleanup qilmaydi (assignment idempotent).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useRef, useEffect } from 'react';

export function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}
```

Initial value bilan variant:

```tsx
export function usePreviousWithInitial<T>(value: T, initial: T): T {
  const ref = useRef<T>(initial);
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// Ishlatish — undefined check yo'q
function Component({ count }: { count: number }) {
  const prev = usePreviousWithInitial(count, 0);
  return <div>Diff: {count - prev}</div>;
}
```

Animation trigger:

```tsx
function CountUp({ value }: { value: number }) {
  const prev = usePrevious(value);
  const [displayed, setDisplayed] = useState(value);
  
  useEffect(() => {
    if (prev === undefined || prev === value) return;
    
    let frame: number;
    const start = prev;
    const end = value;
    const duration = 500;
    const startTime = performance.now();
    
    const animate = (now: number) => {
      const elapsed = now - startTime;
      const progress = Math.min(elapsed / duration, 1);
      setDisplayed(Math.round(start + (end - start) * progress));
      
      if (progress < 1) frame = requestAnimationFrame(animate);
    };
    
    frame = requestAnimationFrame(animate);
    return () => cancelAnimationFrame(frame);
  }, [value, prev]);
  
  return <span>{displayed}</span>;
}
```

State transition logging:

```tsx
function useStateLogger<T>(value: T, label: string) {
  const prev = usePrevious(value);
  
  useEffect(() => {
    if (prev !== undefined && prev !== value) {
      console.log(`[${label}] ${JSON.stringify(prev)} → ${JSON.stringify(value)}`);
    }
  }, [value, prev, label]);
}

function UserProfilePanel() {
  const [user, setUser] = useState<User | null>(null);
  useStateLogger(user, 'user');
  // user o'zgarsa console'da log
}
```

</details>

---

## `useDebounce` va `useDebouncedCallback`

### Nazariya

Debouncing — function chaqiruvni "delay" bilan kechiktirish, agar yangi chaqiruv kelsa — eski timer'ni bekor qilish va yangidan boshlash. Search input, autocomplete, resize handler — tez-tez o'zgaradigan event'lar uchun ideal.

Ikki variant:

1. **`useDebounce(value, delay)`** — value'ni debounce qiladi (state-based)
2. **`useDebouncedCallback(fn, delay)`** — function chaqiruvni debounce qiladi

`useDebounce`:

```tsx
function useDebounce<T>(value: T, delay: number = 300): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debounced;
}
```

QANDAY ISHLAYDI:

1. `value` o'zgarganda `useEffect` ishlaydi.
2. `setTimeout` schedule qiladi `delay` ms keyin `setDebounced(value)`.
3. Agar `value` yana o'zgarsa (delay tugamasidan), cleanup `clearTimeout` chaqiriladi (eski timer bekor).
4. Yangi `setTimeout` schedule qilinadi.
5. Foydalanuvchi `delay` ms davomida o'zgartirmasa — `setDebounced` chaqiriladi.

Effekt — `debounced` qiymat faqat foydalanuvchi to'xtagandan keyin yangilanadi.

`useDebouncedCallback`:

```tsx
function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number = 300
): (...args: Parameters<T>) => void {
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const callbackRef = useRef(callback);
  
  // Latest callback (closure trap'dan saqlanish)
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  return useCallback((...args: Parameters<T>) => {
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      callbackRef.current(...args);
    }, delay);
  }, [delay]);
}
```

QANDAY ISHLAYDI:

1. Har chaqiruv'da eski timer bekor.
2. Yangi timer schedule qilinadi.
3. `delay` ms o'tgach — latest callback chaqiriladi (latest closure pattern).

NIMA UCHUN ikki variant: `useDebounce` value-based — controlled input bilan ishlatiladi (input'da har keypress state, lekin search faqat debounced'da). `useDebouncedCallback` event-based — handler'larda (resize, scroll, mousemove).

> **Versiya evolyutsiyasi (Debounce):**
> - **Pre-Hooks (R16.7-):** Class komponent'da `componentDidUpdate` + `setTimeout` + `clearTimeout` lifecycle. Boilerplate.
> - **R16.8+:** `useEffect` cleanup pattern. Debounce custom hook standart.
> - **R18+:** `useTransition` va `useDeferredValue` (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)) ba'zi use case'larda debounce o'rniga (priority-based scheduling).

`useDeferredValue` vs `useDebounce` farq: `useDeferredValue` priority-based (scheduler boshqaradi, qachon idle'da update), `useDebounce` time-based (aniq `delay` ms). Search uchun `useDeferredValue` (Concurrent-aware), API call uchun `useDebounce` (network rate limit).

<details>
<summary><strong>Under the Hood</strong></summary>

`useDebounce` cleanup mexanizmi:

```
value = "h"     → schedule setTimeout('h', 300ms)
value = "he"    → cleanup ('h' bekor) → schedule ('he', 300ms)
value = "hel"   → cleanup ('he' bekor) → schedule ('hel', 300ms)
value = "hell"  → cleanup ('hel' bekor) → schedule ('hell', 300ms)
value = "hello" → cleanup ('hell' bekor) → schedule ('hello', 300ms)
                  └─ 300ms kutadi
                  └─ setDebounced('hello')
```

`clearTimeout` cleanup function ichida — `useEffect` har dependency change'da cleanup chaqiradi.

`useDebouncedCallback` latest closure pattern:

```javascript
// Klassik xato — closure'da eski callback
function useDebouncedBad(callback, delay) {
  return useCallback((...args) => {
    setTimeout(() => callback(...args), delay);
    // ❌ callback render closure'idan — eski qiymat
  }, [callback, delay]);  // har callback o'zgarsa yangi function (re-attach event listener)
}

// To'g'ri — ref orqali latest
function useDebouncedGood(callback, delay) {
  const callbackRef = useRef(callback);
  useEffect(() => {
    callbackRef.current = callback;  // har render'da yangilanadi
  });
  
  return useCallback((...args) => {
    setTimeout(() => callbackRef.current(...args), delay);
    // ✅ callbackRef.current — har gal latest
  }, [delay]);  // faqat delay o'zgarsa yangi function (kam re-attach)
}
```

Latest closure pattern (cross-ref [`18-useref.md`](18-useref.md)) — ref'da latest function saqlash, callback `ref.current` orqali latest version chaqirish.

Cleanup unmount paytida:

```tsx
useEffect(() => {
  return () => {
    if (timerRef.current) clearTimeout(timerRef.current);
  };
}, []);
```

Aks holda unmounted komponentda timer ishlasa — memory leak yoki "Can't perform a React state update on an unmounted component" warning.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`useDebounce` implementation va misol:

```tsx
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debounced, setDebounced] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debounced;
}

// Ishlatish — search input
function ProductSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  const [results, setResults] = useState<Product[]>([]);
  
  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }
    
    const controller = new AbortController();
    fetch(`/api/products/search?q=${debouncedQuery}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });
    
    return () => controller.abort();
  }, [debouncedQuery]);
  
  return (
    <div>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Mahsulot qidirish..."
      />
      <ul>
        {results.map(p => <li key={p.id}>{p.name}</li>)}
      </ul>
    </div>
  );
}
```

`useDebouncedCallback` implementation va misol:

```tsx
import { useRef, useCallback, useEffect } from 'react';

export function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number = 300
): (...args: Parameters<T>) => void {
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const callbackRef = useRef(callback);
  
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  useEffect(() => {
    return () => {
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, []);
  
  return useCallback((...args: Parameters<T>) => {
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      callbackRef.current(...args);
    }, delay);
  }, [delay]);
}

// Ishlatish — resize handler
function ResponsiveLayout() {
  const [width, setWidth] = useState(window.innerWidth);
  
  const handleResize = useDebouncedCallback(() => {
    setWidth(window.innerWidth);
  }, 200);
  
  useEffect(() => {
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, [handleResize]);
  
  return <div>Width: {width}</div>;
}
```

Cancel/flush methods (advanced):

```tsx
interface DebouncedFunction<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): void;
  cancel: () => void;
  flush: () => void;
}

export function useDebouncedCallbackAdvanced<T extends (...args: any[]) => any>(
  callback: T,
  delay: number = 300
): DebouncedFunction<T> {
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const callbackRef = useRef(callback);
  const argsRef = useRef<Parameters<T> | null>(null);
  
  useEffect(() => {
    callbackRef.current = callback;
  });
  
  const cancel = useCallback(() => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
    argsRef.current = null;
  }, []);
  
  const flush = useCallback(() => {
    if (timerRef.current && argsRef.current) {
      clearTimeout(timerRef.current);
      callbackRef.current(...argsRef.current);
      timerRef.current = null;
      argsRef.current = null;
    }
  }, []);
  
  const debounced = useCallback((...args: Parameters<T>) => {
    argsRef.current = args;
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      callbackRef.current(...args);
      timerRef.current = null;
      argsRef.current = null;
    }, delay);
  }, [delay]);
  
  useEffect(() => {
    return cancel;  // cleanup on unmount
  }, [cancel]);
  
  return Object.assign(debounced, { cancel, flush });
}

// Ishlatish — submit button avval flush
function SearchForm() {
  const [query, setQuery] = useState('');
  
  const updateQuery = useDebouncedCallbackAdvanced((value: string) => {
    console.log('Search:', value);
  }, 500);
  
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    updateQuery.flush();  // pending search'ni darhol bajarish
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          updateQuery(e.target.value);
        }}
      />
      <button>Qidirish</button>
    </form>
  );
}
```

</details>

---

## `useLocalStorage` — Persistent State (SSR-safe)

### Nazariya

`useLocalStorage` — `useState` API bilan localStorage'ga sinxron state. Komponent state localStorage'ga avtomatik saqlanadi va initial render'da o'qiladi. Sahifa qayta yuklansa — state saqlanib qoladi.

```tsx
function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  // SSR-safe initial read
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;
    try {
      const item = localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      console.warn(`useLocalStorage: Error reading "${key}":`, error);
      return initialValue;
    }
  });
  
  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    try {
      setStoredValue(prev => {
        const newValue = value instanceof Function ? value(prev) : value;
        if (typeof window !== 'undefined') {
          localStorage.setItem(key, JSON.stringify(newValue));
        }
        return newValue;
      });
    } catch (error) {
      console.warn(`useLocalStorage: Error writing "${key}":`, error);
    }
  }, [key]);
  
  return [storedValue, setValue];
}
```

QANDAY ISHLAYDI:

1. **Initial render** — `useState` lazy initializer (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)) localStorage'dan o'qiydi (yoki SSR'da initialValue).
2. **`setValue` chaqiruvi** — state yangilanadi va localStorage'ga JSON.stringify orqali yoziladi.
3. **Functional update** — `setValue(prev => ...)` — `useState` qoidasi.
4. **Try-catch** — JSON parse error, localStorage quota exceeded'larni handle qiladi.

NIMA UCHUN: theme preference, language selection, form draft, tutorial completion — bularning hammasi sahifa qayta yuklash davomida saqlanishi kerak. `localStorage` API native, lekin manual sync (read on mount, write on change) custom hook'siz boilerplate.

**SSR considerations** (Next.js, Astro):

- **Initial render server'da** — `window` mavjud emas → `typeof window === 'undefined'` check.
- **Hydration** — server initialValue'ni render qiladi, client localStorage'ni o'qiydi → mismatch potential.
- **Yechim**: `useEffect` orqali post-mount sync (R18+ hydration tolerant), yoki `'use client'` directive (Next.js App Router).

Cross-tab sync — alternative pattern:

```tsx
// `storage` event boshqa tablar localStorage o'zgarishini eshitadi
useEffect(() => {
  const handler = (e: StorageEvent) => {
    if (e.key === key && e.newValue) {
      setStoredValue(JSON.parse(e.newValue) as T);
    }
  };
  window.addEventListener('storage', handler);
  return () => window.removeEventListener('storage', handler);
}, [key]);
```

`storage` event faqat **boshqa tab** localStorage'ni o'zgartirsa fire qiladi (joriy tab — yo'q). Cross-tab synchronization'ga qulay.

<details>
<summary><strong>Under the Hood</strong></summary>

localStorage API bo'yicha cheklov'lar:

| Cheklov | Tafsilot |
|---------|----------|
| Storage size | Domain bo'yicha 5-10 MB (browser farqli) |
| Synchronous | Read/write blocking (main thread) |
| String only | Boshqa turlarni JSON.stringify shart |
| Quota exceeded | Catch qilinishi shart (Safari Private mode) |
| Same-origin | Cross-domain access yo'q |

Performance — localStorage sinxron API, har read/write main thread'ni bloklaydi. Kichik value'lar uchun sezilmas, katta value'lar (ko'p kilobaytli JSON) uchun read/write blocking sezilarli kechikish berishi mumkin.

Modern alternative — **IndexedDB** (asynchronous, larger storage). Lekin API verbose — `idb-keyval` library yoki `Dexie.js` yordam beradi.

`useLocalStorage` Concurrent rendering bilan moslashuv: `useSyncExternalStore` (R18+, cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)) bilan tearing-safe yechim. Lekin `getSnapshot` reference barqarorligi kritik — quyidagi sodda variant object/array qiymatlarda **buziladi**:

```tsx
// ❌ Anti-pattern — getSnapshot har chaqiruvda yangi reference
function useLocalStorageSyncNaive<T>(key: string, initialValue: T): T {
  const subscribe = useCallback((callback: () => void) => {
    const handler = (e: StorageEvent) => {
      if (e.key === key) callback();
    };
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, [key]);
  
  const getSnapshot = useCallback((): T => {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : initialValue;  // ❌ har gal yangi object
  }, [key, initialValue]);
  
  const getServerSnapshot = useCallback(() => initialValue, [initialValue]);
  
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

KRITIK muammo — `getSnapshot` har chaqiruvda `JSON.parse` qiladi, ya'ni saqlangan qiymat object/array bo'lsa har safar **yangi reference** qaytadi. `useSyncExternalStore` snapshot'ni `Object.is` bilan solishtiradi: har gal yangi reference → React "getSnapshot should be cached" warning beradi va render loop'iga tushishi mumkin. To'g'ri yechim — oxirgi raw string va parse qilingan natijani cache qilish, string o'zgarmasa eski reference'ni qaytarish (string primitive bo'lgani uchun `Object.is` barqaror):

```tsx
// ✅ getSnapshot raw string o'zgarmasa cache'langan reference qaytaradi
function useLocalStorageSync<T>(key: string, initialValue: T): T {
  const cacheRef = useRef<{ raw: string | null; parsed: T }>({
    raw: null,
    parsed: initialValue,
  });

  const subscribe = useCallback((callback: () => void) => {
    const handler = (e: StorageEvent) => {
      if (e.key === key) callback();
    };
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, [key]);

  const getSnapshot = useCallback((): T => {
    const raw = localStorage.getItem(key);
    if (raw === cacheRef.current.raw) return cacheRef.current.parsed; // barqaror reference
    const parsed = raw !== null ? (JSON.parse(raw) as T) : initialValue;
    cacheRef.current = { raw, parsed };
    return parsed;
  }, [key, initialValue]);

  const getServerSnapshot = useCallback(() => initialValue, [initialValue]);

  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useState, useCallback, useEffect } from 'react';

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      console.warn(`useLocalStorage: Error reading "${key}":`, error);
      return initialValue;
    }
  });
  
  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    try {
      setStoredValue(prev => {
        const newValue = value instanceof Function ? value(prev) : value;
        if (typeof window !== 'undefined') {
          window.localStorage.setItem(key, JSON.stringify(newValue));
        }
        return newValue;
      });
    } catch (error) {
      console.warn(`useLocalStorage: Error writing "${key}":`, error);
    }
  }, [key]);
  
  // Cross-tab sync
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    const handler = (e: StorageEvent) => {
      if (e.key === key && e.newValue !== null) {
        try {
          setStoredValue(JSON.parse(e.newValue) as T);
        } catch (error) {
          console.warn(`useLocalStorage: Error parsing "${key}":`, error);
        }
      }
    };
    
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, [key]);
  
  return [storedValue, setValue];
}
```

Theme preference:

```tsx
type Theme = 'light' | 'dark' | 'auto';

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useLocalStorage<Theme>('theme', 'auto');
  
  useEffect(() => {
    if (theme === 'auto') {
      const mql = window.matchMedia('(prefers-color-scheme: dark)');
      document.documentElement.dataset.theme = mql.matches ? 'dark' : 'light';
    } else {
      document.documentElement.dataset.theme = theme;
    }
  }, [theme]);
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

Form draft:

```tsx
interface ContactDraft {
  name: string;
  email: string;
  message: string;
}

function ContactForm() {
  const [draft, setDraft] = useLocalStorage<ContactDraft>('contact-draft', {
    name: '',
    email: '',
    message: '',
  });
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await sendContact(draft);
    setDraft({ name: '', email: '', message: '' });  // clear draft
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={draft.name}
        onChange={(e) => setDraft(d => ({ ...d, name: e.target.value }))}
      />
      <input
        value={draft.email}
        onChange={(e) => setDraft(d => ({ ...d, email: e.target.value }))}
      />
      <textarea
        value={draft.message}
        onChange={(e) => setDraft(d => ({ ...d, message: e.target.value }))}
      />
      <button>Yuborish</button>
    </form>
  );
}
```

`useSessionStorage` variant:

```tsx
export function useSessionStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  // localStorage o'rniga sessionStorage
  // sessionStorage tab close qilinsa tozalanadi
  // ... aynan useLocalStorage logic, faqat localStorage → sessionStorage
}
```

</details>

---

## `useMediaQuery` — Responsive Design

### Nazariya

`useMediaQuery` — CSS media query'ni JavaScript'da kuzatish. Responsive design uchun (mobile/desktop layout, dark mode preference, reduced motion).

```tsx
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia(query).matches;
  });
  
  useEffect(() => {
    const mql = window.matchMedia(query);
    setMatches(mql.matches);
    
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, [query]);
  
  return matches;
}
```

QANDAY ISHLAYDI:

1. `window.matchMedia(query)` `MediaQueryList` obyektni qaytaradi.
2. `mql.matches` boolean — query mos kelyaptimi.
3. `change` event listener orqali real-time update.
4. Cleanup'da listener remove.

NIMA UCHUN: CSS media query'lar UI'ga ta'sir qiladi, lekin JavaScript logic'ga kirmaydi (`if (isMobile) ...` shaklidagi tekshiruv'lar). `useMediaQuery` JS conditional rendering'ni media query bilan sinxronlashtiradi.

R18+ Concurrent-safe variant — `useSyncExternalStore` (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
function useMediaQuerySync(query: string): boolean {
  const subscribe = useCallback((callback: () => void) => {
    const mql = window.matchMedia(query);
    mql.addEventListener('change', callback);
    return () => mql.removeEventListener('change', callback);
  }, [query]);
  
  const getSnapshot = useCallback(() => {
    return window.matchMedia(query).matches;
  }, [query]);
  
  const getServerSnapshot = useCallback(() => false, []);
  
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

`useSyncExternalStore` — Concurrent rendering tearing prevention (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)).

Common queries:

```tsx
useMediaQuery('(max-width: 768px)');           // mobile
useMediaQuery('(min-width: 1024px)');          // desktop
useMediaQuery('(prefers-color-scheme: dark)'); // dark mode
useMediaQuery('(prefers-reduced-motion: reduce)'); // reduced motion
useMediaQuery('(orientation: portrait)');      // portrait
useMediaQuery('(hover: hover)');               // hover-capable device
```

<details>
<summary><strong>Under the Hood</strong></summary>

`MediaQueryList` API:

```typescript
interface MediaQueryList extends EventTarget {
  matches: boolean;
  media: string;
  addEventListener(type: 'change', listener: (e: MediaQueryListEvent) => void): void;
  removeEventListener(type: 'change', listener: (e: MediaQueryListEvent) => void): void;
}
```

Browser support:

| API | Chrome | Firefox | Safari | Edge |
|-----|--------|---------|--------|------|
| `matchMedia` | 9+ | 6+ | 5.1+ | 12+ |
| `addEventListener('change')` | 14+ | 55+ | 14+ | 16+ |
| `addListener` (legacy) | hammasi | hammasi | hammasi | hammasi |

Eski Safari'larda `addEventListener('change')` ishlamaydi — `addListener`/`removeListener` (deprecated) ishlatilishi kerak. Polyfill:

```javascript
const handler = (e) => setMatches(e.matches);
if (mql.addEventListener) {
  mql.addEventListener('change', handler);
} else {
  // Legacy Safari < 14
  mql.addListener(handler);
}

return () => {
  if (mql.removeEventListener) {
    mql.removeEventListener('change', handler);
  } else {
    mql.removeListener(handler);
  }
};
```

SSR'da `window.matchMedia` mavjud emas — `typeof window === 'undefined'` check yoki initial value default. Hydration mismatch oldini olish uchun `useEffect` orqali post-mount.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useState, useEffect } from 'react';

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState<boolean>(() => {
    if (typeof window === 'undefined') return false;
    return window.matchMedia(query).matches;
  });
  
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    const mql = window.matchMedia(query);
    setMatches(mql.matches);  // initial sync (SSR mismatch fix)
    
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, [query]);
  
  return matches;
}
```

Predefined breakpoints:

```tsx
export function useIsMobile() {
  return useMediaQuery('(max-width: 768px)');
}

export function useIsTablet() {
  return useMediaQuery('(min-width: 769px) and (max-width: 1023px)');
}

export function useIsDesktop() {
  return useMediaQuery('(min-width: 1024px)');
}

export function usePrefersDarkMode() {
  return useMediaQuery('(prefers-color-scheme: dark)');
}

export function usePrefersReducedMotion() {
  return useMediaQuery('(prefers-reduced-motion: reduce)');
}
```

Responsive layout:

```tsx
function ResponsiveLayout({ children }: { children: React.ReactNode }) {
  const isMobile = useIsMobile();
  const prefersDark = usePrefersDarkMode();
  const reduceMotion = usePrefersReducedMotion();
  
  return (
    <div 
      className={`layout ${isMobile ? 'mobile' : 'desktop'} ${prefersDark ? 'dark' : 'light'}`}
      style={{ '--motion': reduceMotion ? '0s' : '0.3s' } as React.CSSProperties}
    >
      {isMobile ? <MobileNav /> : <DesktopSidebar />}
      <main>{children}</main>
    </div>
  );
}
```

Conditional component render:

```tsx
function PostCard({ post }: { post: Post }) {
  const isMobile = useIsMobile();
  
  if (isMobile) {
    return <MobilePostCard post={post} />;
  }
  
  return <DesktopPostCard post={post} />;
}
```

Multiple breakpoints with Map:

```tsx
type Breakpoint = 'sm' | 'md' | 'lg' | 'xl';

const BREAKPOINTS: Record<Breakpoint, string> = {
  sm: '(max-width: 640px)',
  md: '(min-width: 641px) and (max-width: 1024px)',
  lg: '(min-width: 1025px) and (max-width: 1280px)',
  xl: '(min-width: 1281px)',
};

export function useBreakpoint(): Breakpoint {
  const isSm = useMediaQuery(BREAKPOINTS.sm);
  const isMd = useMediaQuery(BREAKPOINTS.md);
  const isLg = useMediaQuery(BREAKPOINTS.lg);
  
  if (isSm) return 'sm';
  if (isMd) return 'md';
  if (isLg) return 'lg';
  return 'xl';
}

function Grid({ items }: { items: Item[] }) {
  const breakpoint = useBreakpoint();
  const columns = { sm: 1, md: 2, lg: 3, xl: 4 }[breakpoint];
  
  return (
    <div style={{ display: 'grid', gridTemplateColumns: `repeat(${columns}, 1fr)` }}>
      {items.map(item => <Card key={item.id} item={item} />)}
    </div>
  );
}
```

</details>

---

## `useWindowSize` — Window Dimensions

### Nazariya

`useWindowSize` — `window.innerWidth` va `window.innerHeight` qiymatlarini real-time kuzatish.

```tsx
interface WindowSize {
  width: number;
  height: number;
}

function useWindowSize(): WindowSize {
  const [size, setSize] = useState<WindowSize>(() => {
    if (typeof window === 'undefined') return { width: 0, height: 0 };
    return { width: window.innerWidth, height: window.innerHeight };
  });
  
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    const handler = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    
    handler();  // initial sync (SSR mismatch fix)
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  
  return size;
}
```

NIMA UCHUN: canvas, charts, virtualized lists, conditional rendering — pixel-perfect dimensions kerak. CSS units (`vw`, `vh`) statik, JS dimensions dynamic.

QANDAY ISHLAYDI:

1. Initial — `window.innerWidth/innerHeight` (SSR'da 0).
2. `resize` event'da `setSize` chaqiriladi.
3. Cleanup'da listener remove.

**Performance issue**: `resize` event tez-tez fire qiladi (drag paytida har frame). Har resize state update + re-render = scroll/animation jankiness. Yechim — debounce yoki throttle:

```tsx
function useWindowSizeDebounced(delay: number = 100): WindowSize {
  const [size, setSize] = useState<WindowSize>(() => {
    if (typeof window === 'undefined') return { width: 0, height: 0 };
    return { width: window.innerWidth, height: window.innerHeight };
  });
  
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    let timer: ReturnType<typeof setTimeout> | null = null;
    
    const handler = () => {
      if (timer) clearTimeout(timer);
      timer = setTimeout(() => {
        setSize({ width: window.innerWidth, height: window.innerHeight });
      }, delay);
    };
    
    handler();
    window.addEventListener('resize', handler);
    
    return () => {
      window.removeEventListener('resize', handler);
      if (timer) clearTimeout(timer);
    };
  }, [delay]);
  
  return size;
}
```

`ResizeObserver` API alternativa — element-specific resize tracking (window-wide emas). `useElementSize` pattern:

```tsx
function useElementSize<T extends HTMLElement = HTMLDivElement>(): [
  React.RefObject<T>,
  WindowSize
] {
  const ref = useRef<T>(null);
  const [size, setSize] = useState<WindowSize>({ width: 0, height: 0 });
  
  useEffect(() => {
    if (!ref.current) return;
    
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return [ref, size];
}

// Ishlatish
function ResizableBox() {
  const [boxRef, { width, height }] = useElementSize<HTMLDivElement>();
  return <div ref={boxRef}>{width}x{height}</div>;
}
```

`ResizeObserver` Chrome 64+, Firefox 69+, Safari 13.1+ — universal modern support.

<details>
<summary><strong>Under the Hood</strong></summary>

`window.innerWidth/innerHeight` vs alternatives:

| API | Tafsilot | Use case |
|-----|----------|----------|
| `window.innerWidth` | Viewport width (scrollbar bilan) | Window size |
| `document.documentElement.clientWidth` | Viewport width (scrollbar siz) | Layout calculation |
| `document.body.clientWidth` | Body width | Body-specific |
| `window.outerWidth` | Browser window total | Multi-monitor |
| `screen.width` | Display screen | Hardware info |

`window.innerWidth` ko'pchilik holatda to'g'ri tanlov.

`resize` event window o'lcham o'zgarishi davomida ko'p marta fire qiladi — browser uni odatda repaint frame'lariga moslab yuboradi (drag paytida har frame atrofida). Har firing'da state update + re-render bo'lsa — jankiness. Manual debounce yoki throttle qo'shimcha optimization beradi.

R18 Concurrent rendering — `useWindowSize` priority pastroq Lane'ga ko'chirish mumkin (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
function useWindowSizeTransition(): WindowSize {
  const [size, setSize] = useState<WindowSize>(/* ... */);
  
  useEffect(() => {
    const handler = () => {
      startTransition(() => {
        setSize({ width: window.innerWidth, height: window.innerHeight });
      });
    };
    
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  
  return size;
}
```

`startTransition` resize update'larni Transition lane'ga ko'chiradi — urgent input (typing, click) bloklanmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useState, useEffect } from 'react';

interface WindowSize {
  width: number;
  height: number;
}

export function useWindowSize(): WindowSize {
  const [size, setSize] = useState<WindowSize>(() => {
    if (typeof window === 'undefined') return { width: 0, height: 0 };
    return { width: window.innerWidth, height: window.innerHeight };
  });
  
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    const handler = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    
    handler();
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  
  return size;
}
```

Canvas with dynamic size:

```tsx
function ResponsiveCanvas() {
  const { width, height } = useWindowSize();
  const canvasRef = useRef<HTMLCanvasElement>(null);
  
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    
    const ctx = canvas.getContext('2d');
    if (!ctx) return;
    
    canvas.width = width;
    canvas.height = height;
    
    // Draw something
    ctx.fillStyle = 'blue';
    ctx.fillRect(0, 0, width, height);
  }, [width, height]);
  
  return <canvas ref={canvasRef} />;
}
```

Conditional layout based on size:

```tsx
function Dashboard() {
  const { width } = useWindowSize();
  const showSidebar = width >= 1024;
  
  return (
    <div className="dashboard">
      {showSidebar && <Sidebar />}
      <main>Content</main>
    </div>
  );
}
```

</details>

---

## `useEventListener` — Event Subscription

### Nazariya

`useEventListener` — DOM event'larga subscribe qilish uchun universal hook. `addEventListener` + cleanup pattern'ni qadoqlaydi.

```tsx
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window = window
): void;

function useEventListener<K extends keyof DocumentEventMap>(
  eventName: K,
  handler: (event: DocumentEventMap[K]) => void,
  element: Document
): void;

function useEventListener<K extends keyof HTMLElementEventMap, T extends HTMLElement>(
  eventName: K,
  handler: (event: HTMLElementEventMap[K]) => void,
  element: React.RefObject<T>
): void;

function useEventListener(
  eventName: string,
  handler: (event: Event) => void,
  element: any = window
): void {
  const handlerRef = useRef(handler);
  
  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);
  
  useEffect(() => {
    const target = 'current' in element ? element.current : element;
    if (!target?.addEventListener) return;
    
    const listener = (e: Event) => handlerRef.current(e);
    target.addEventListener(eventName, listener);
    return () => target.removeEventListener(eventName, listener);
  }, [eventName, element]);
}
```

QANDAY ISHLAYDI:

1. **Latest closure pattern** — `handlerRef.current = handler` har render'da yangilanadi.
2. **Event listener stable** — `addEventListener` faqat `eventName`/`element` o'zgarsa qayta attach qilinadi (re-attach paytida memory churn).
3. **TypeScript overloads** — element turiga qarab event types narrow qilinadi.

NIMA UCHUN: har joyda `useEffect` + `addEventListener` + `removeEventListener` boilerplate yozish o'rniga, hook bir xil pattern beradi. TypeScript event types avtomatik (`MouseEvent`, `KeyboardEvent`, va h.k.).

Element variantlari:

```tsx
// Window event
useEventListener('resize', handleResize);

// Document event
useEventListener('keydown', handleKeyDown, document);

// Element ref
const buttonRef = useRef<HTMLButtonElement>(null);
useEventListener('click', handleClick, buttonRef);
```

Latest closure muhim — handler render closure'idan emas, ref orqali latest version chaqiriladi:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  
  // ❌ Anti-pattern (handler stale)
  useEffect(() => {
    const handler = () => console.log(count);  // closure'da count
    window.addEventListener('click', handler);
    return () => window.removeEventListener('click', handler);
  }, []);  // count deps yo'q → handler stale
  
  // ✅ useEventListener latest closure
  useEventListener('click', () => console.log(count));  // har gal latest count
  
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Latest closure pattern internal:

```tsx
// Render N: handler = () => console.log(0)
//   handlerRef.current = handler  // after useEffect
//
// Render N+1: handler = () => console.log(1)
//   handlerRef.current = handler  // updated
//
// Listener (stable, attached once):
//   listener = (e) => handlerRef.current(e)  // har gal latest
```

`addEventListener` faqat bir marta chaqiriladi (eventName/element o'zgarmaganda). Listener inline function — `handlerRef.current` orqali latest handler chaqiriladi. Bu pattern memory-efficient (event listener'lar re-attach qilinmaydi).

`useLayoutEffect` vs `useEffect` for handlerRef sync:

```tsx
// useEffect — async (paint'dan keyin)
useEffect(() => {
  handlerRef.current = handler;
});

// useLayoutEffect — sync (paint'dan oldin)
useLayoutEffect(() => {
  handlerRef.current = handler;
});
```

`useLayoutEffect` ref sync uchun ba'zan afzal — u Commit Phase'da paint'dan oldin sync ishlaydi, ya'ni handlerRef paint'dan oldin yangilanadi. Lekin amaliy farq deyarli yo'q (`useEffect` ham aksariyat holatda yetadi, chunki event'lar paint'dan keyin firing bo'ladi).

`useInsertionEffect` (R18+, cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)) — layout effect'lardan oldin ishlaydi va CSS-in-JS kutubxonalari uchun mo'ljallangan (`<style>` tag inject qilish). Bu hook ichida ref hali attach qilinmagan va state update taqiqlangan — event listener subscription uchun mos emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation (overloaded TypeScript):

```tsx
import { useEffect, useRef } from 'react';

// Overload 1: Window
export function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element?: undefined
): void;

// Overload 2: Document
export function useEventListener<K extends keyof DocumentEventMap>(
  eventName: K,
  handler: (event: DocumentEventMap[K]) => void,
  element: Document
): void;

// Overload 3: HTMLElement ref
export function useEventListener<
  K extends keyof HTMLElementEventMap,
  T extends HTMLElement = HTMLElement
>(
  eventName: K,
  handler: (event: HTMLElementEventMap[K]) => void,
  element: React.RefObject<T>
): void;

// Implementation
export function useEventListener(
  eventName: string,
  handler: (event: any) => void,
  element?: any
): void {
  const handlerRef = useRef(handler);
  
  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);
  
  useEffect(() => {
    const target = element?.current ?? element ?? window;
    if (!target?.addEventListener) return;
    
    const listener: EventListener = (e) => handlerRef.current(e);
    target.addEventListener(eventName, listener);
    return () => target.removeEventListener(eventName, listener);
  }, [eventName, element]);
}
```

Window event:

```tsx
function ScrollToTop() {
  const [showButton, setShowButton] = useState(false);
  
  useEventListener('scroll', () => {
    setShowButton(window.scrollY > 500);
  });
  
  if (!showButton) return null;
  
  return (
    <button onClick={() => window.scrollTo({ top: 0, behavior: 'smooth' })}>
      ↑ Top
    </button>
  );
}
```

Document event — keyboard shortcut:

```tsx
function CommandPalette() {
  const [isOpen, setIsOpen] = useState(false);
  
  useEventListener('keydown', (e) => {
    if (e.key === 'k' && (e.ctrlKey || e.metaKey)) {
      e.preventDefault();
      setIsOpen(o => !o);
    } else if (e.key === 'Escape') {
      setIsOpen(false);
    }
  }, document);
  
  if (!isOpen) return null;
  return <div className="palette">Command palette</div>;
}
```

Element-specific event:

```tsx
function VideoControls({ videoRef }: { videoRef: React.RefObject<HTMLVideoElement> }) {
  const [isPlaying, setIsPlaying] = useState(false);
  const [currentTime, setCurrentTime] = useState(0);
  
  useEventListener('play', () => setIsPlaying(true), videoRef);
  useEventListener('pause', () => setIsPlaying(false), videoRef);
  useEventListener('timeupdate', () => {
    if (videoRef.current) setCurrentTime(videoRef.current.currentTime);
  }, videoRef);
  
  return (
    <div>
      <span>{isPlaying ? '▶' : '⏸'}</span>
      <span>{currentTime.toFixed(2)}s</span>
    </div>
  );
}
```

</details>

---

## `useOnClickOutside` — Click Outside Detection

### Nazariya

`useOnClickOutside` — element tashqarisida click qilinganida callback chaqiradi. Modal, dropdown, popover yopish uchun.

```tsx
function useOnClickOutside<T extends HTMLElement = HTMLElement>(
  ref: React.RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
): void {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      const target = event.target as Node;
      if (!ref.current || ref.current.contains(target)) return;
      handler(event);
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}
```

QANDAY ISHLAYDI:

1. `document` ga `mousedown` va `touchstart` listener qo'shiladi.
2. Click event firing — `event.target` Node.
3. `ref.current.contains(target)` — element ichida click bo'lganmi?
4. Tashqarida bo'lsa — `handler` chaqiriladi.

NIMA UCHUN: dropdown menu, popover, modal — UX'da tashqari click yopish kutiladi. `Element.contains()` API bilan element hierarchy traversed.

**Mouse vs Touch** — mobile device'larda touch event'lar (`touchstart`) qo'shimcha. Mouse event ham yetadi (browser touch'ni mouse'ga emulate qiladi), lekin tap yumshoqroq response uchun touch'ni alohida ushlash.

**`mousedown` vs `click`** — `mousedown` faster (button press onset), `click` complete (press + release). Modal yopish uchun `mousedown` UX bo'yicha tezroq.

**Edge case** — modal portal'ga rendered (cross-ref [`28-portals.md`](28-portals.md)). DOM tree va React tree farqlanadi. `ref.current.contains(target)` DOM bo'yicha tekshiradi — portal element o'z DOM joyida. Bu odatda to'g'ri (modal portal `<body>` ostida, lekin click target portal element'ning ichida).

Latest closure pattern qo'shish:

```tsx
function useOnClickOutsideStable<T extends HTMLElement>(
  ref: React.RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
): void {
  const handlerRef = useRef(handler);
  
  useEffect(() => {
    handlerRef.current = handler;
  });
  
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      const target = event.target as Node;
      if (!ref.current || ref.current.contains(target)) return;
      handlerRef.current(event);  // latest handler
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref]);  // handler dep yo'q
}
```

Latest closure pattern — handler dependency'siz, listener bir marta attach.

<details>
<summary><strong>Under the Hood</strong></summary>

`Element.contains()` browser API:

- DOM tree traversal — element child bo'lsa true.
- Self-check — `el.contains(el)` true.
- Disconnected node — false.
- Performance — O(depth), shallow elements uchun fast.

Alternative — `Node.compareDocumentPosition()` complex traversal pattern'lari uchun. `contains` aksariyat click outside use case'larga yetadi.

Click outside vs focus outside farq:

```tsx
// Click outside — fizik click event
useOnClickOutside(ref, () => closeModal());

// Focus outside — keyboard navigation Tab key
function useFocusOutside(ref, handler) {
  useEffect(() => {
    const listener = (e: FocusEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        handler();
      }
    };
    document.addEventListener('focusin', listener);
    return () => document.removeEventListener('focusin', listener);
  }, [ref, handler]);
}
```

Accessibility — modal yopish uchun `Escape` key + click outside + close button — uchovi kerak. Faqat click outside yetmaydi (keyboard user'lar uchun).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useEffect, useRef } from 'react';

export function useOnClickOutside<T extends HTMLElement = HTMLElement>(
  ref: React.RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
): void {
  const handlerRef = useRef(handler);
  
  useEffect(() => {
    handlerRef.current = handler;
  });
  
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      const target = event.target as Node;
      if (!ref.current || ref.current.contains(target)) return;
      handlerRef.current(event);
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref]);
}
```

Dropdown menu:

```tsx
function DropdownMenu({ items }: { items: MenuItem[] }) {
  const [isOpen, setIsOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);
  
  useOnClickOutside(menuRef, () => setIsOpen(false));
  
  return (
    <div ref={menuRef} className="dropdown">
      <button onClick={() => setIsOpen(o => !o)}>
        Menu {isOpen ? '▲' : '▼'}
      </button>
      {isOpen && (
        <ul className="menu">
          {items.map(item => (
            <li key={item.id} onClick={() => {
              item.onClick();
              setIsOpen(false);
            }}>
              {item.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

Modal with multiple close mechanisms:

```tsx
function Modal({ 
  isOpen, 
  onClose, 
  children 
}: { 
  isOpen: boolean; 
  onClose: () => void; 
  children: React.ReactNode;
}) {
  const modalRef = useRef<HTMLDivElement>(null);
  
  useOnClickOutside(modalRef, onClose);
  
  useEventListener('keydown', (e) => {
    if (e.key === 'Escape' && isOpen) onClose();
  }, document);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop">
      <div ref={modalRef} className="modal-content">
        <button onClick={onClose}>×</button>
        {children}
      </div>
    </div>
  );
}
```

Popover with conditional listener:

```tsx
function Popover({ trigger, content }: { trigger: React.ReactNode; content: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const popoverRef = useRef<HTMLDivElement>(null);
  
  useOnClickOutside(popoverRef, () => {
    if (isOpen) setIsOpen(false);
  });
  
  return (
    <div ref={popoverRef} className="popover">
      <div onClick={() => setIsOpen(o => !o)}>{trigger}</div>
      {isOpen && <div className="popover-content">{content}</div>}
    </div>
  );
}
```

</details>

---

## `useIntersectionObserver` — Visibility Detection

### Nazariya

`useIntersectionObserver` — element viewport yoki parent element ichida ko'rinishi (`isIntersecting`)ni kuzatadi. Lazy loading, infinite scroll, animation trigger uchun.

```tsx
interface UseIntersectionObserverOptions extends IntersectionObserverInit {
  freezeOnceVisible?: boolean;
}

function useIntersectionObserver<T extends Element = Element>(
  options: UseIntersectionObserverOptions = {}
): [React.RefObject<T>, IntersectionObserverEntry | null] {
  const { freezeOnceVisible = false, ...observerOptions } = options;
  const ref = useRef<T>(null);
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null);
  const frozen = entry?.isIntersecting && freezeOnceVisible;
  
  useEffect(() => {
    if (!ref.current || frozen) return;
    
    const observer = new IntersectionObserver(([entry]) => {
      setEntry(entry);
    }, observerOptions);
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [
    observerOptions.threshold, 
    observerOptions.root, 
    observerOptions.rootMargin, 
    frozen
  ]);
  
  return [ref, entry];
}
```

QANDAY ISHLAYDI:

1. `IntersectionObserver` API — viewport bilan element kesishishini async kuzatadi.
2. `observe(element)` element'ni track qilishni boshlaydi.
3. Callback har visibility change'da chaqiriladi.
4. `entry.isIntersecting` — element ko'rinmoqdami.
5. `freezeOnceVisible` — element bir marta ko'ringach observer to'xtaydi (animation faqat bir marta).

NIMA UCHUN: scroll event listener O(n) — har scroll'da hammasi tekshiriladi. `IntersectionObserver` async va efficient — browser optimization (scroll'siz background'da hisoblanadi).

Use case'lar:

| Use case | Misol | Sozlama |
|----------|-------|---------|
| Lazy image | Image src visible bo'lganda set | `threshold: 0.1` |
| Infinite scroll | Last item visible — load more | `threshold: 1`, `rootMargin: '100px'` |
| Animation trigger | Element ko'ringach fade-in | `freezeOnceVisible: true` |
| Sticky header | Header chiqsa style change | `threshold: 0` |

`IntersectionObserver` options:

- **`threshold`** — `0` (1px ham yetadi) → `1` (100% visible). Array `[0, 0.5, 1]` — har step'da callback.
- **`root`** — observer relative to which element (default: viewport).
- **`rootMargin`** — root atrofiga margin (`'100px 0px'` — 100px tepa/past).

<details>
<summary><strong>Under the Hood</strong></summary>

`IntersectionObserver` async — callback'lar render pipeline'ida (layout'dan keyin) browser tomonidan asinxron yetkaziladi, sync emas. Bu sabab — sync `getBoundingClientRect()` bilan farq:

| API | Sync/Async | Performance | Layout thrashing |
|-----|-----------|-------------|------------------|
| `getBoundingClientRect` | Sync | Slow (forces layout) | Yes |
| `IntersectionObserver` | Async | Fast (browser optimized) | No |

Browser support — Chrome 51+, Firefox 55+, Safari 12.1+. Universal modern.

`IntersectionObserver` performance — multiple elements bir observer bilan track qilish optimal:

```tsx
function useIntersectionObserverMulti<T extends Element>(
  callback: (entries: IntersectionObserverEntry[]) => void,
  options?: IntersectionObserverInit
) {
  const observerRef = useRef<IntersectionObserver | null>(null);
  
  useEffect(() => {
    observerRef.current = new IntersectionObserver(callback, options);
    return () => observerRef.current?.disconnect();
  }, [callback, options]);
  
  const observe = useCallback((element: T) => {
    observerRef.current?.observe(element);
  }, []);
  
  return observe;
}
```

Bir IntersectionObserver instance bir nechta element'ni kuzata oladi (har element uchun alohida observer yaratishdan ko'ra arzonroq) — uzun ro'yxatlarda (infinite scroll item'lari) bitta observer'ni qayta ishlatish tavsiya etiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useState, useEffect, useRef } from 'react';

interface UseIntersectionObserverOptions extends IntersectionObserverInit {
  freezeOnceVisible?: boolean;
}

export function useIntersectionObserver<T extends Element = Element>(
  options: UseIntersectionObserverOptions = {}
): [React.RefObject<T>, IntersectionObserverEntry | null] {
  const { freezeOnceVisible = false, threshold = 0, root = null, rootMargin = '0px' } = options;
  const ref = useRef<T>(null);
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null);
  const frozen = entry?.isIntersecting && freezeOnceVisible;
  
  useEffect(() => {
    const node = ref.current;
    if (!node || frozen) return;
    
    const hasIOSupport = !!window.IntersectionObserver;
    if (!hasIOSupport) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => setEntry(entry),
      { threshold, root, rootMargin }
    );
    
    observer.observe(node);
    return () => observer.disconnect();
  }, [threshold, root, rootMargin, frozen]);
  
  return [ref, entry];
}
```

Lazy image:

```tsx
function LazyImage({ src, alt, placeholder }: { src: string; alt: string; placeholder: string }) {
  const [imgRef, entry] = useIntersectionObserver<HTMLImageElement>({
    threshold: 0.1,
    freezeOnceVisible: true,
  });
  const isVisible = entry?.isIntersecting;
  
  return (
    <img
      ref={imgRef}
      src={isVisible ? src : placeholder}
      alt={alt}
      loading="lazy"  // Native lazy loading fallback
    />
  );
}
```

Fade-in animation:

```tsx
function FadeInSection({ children }: { children: React.ReactNode }) {
  const [sectionRef, entry] = useIntersectionObserver<HTMLDivElement>({
    threshold: 0.15,
    freezeOnceVisible: true,
  });
  const isVisible = entry?.isIntersecting;
  
  return (
    <div
      ref={sectionRef}
      style={{
        opacity: isVisible ? 1 : 0,
        transform: isVisible ? 'translateY(0)' : 'translateY(40px)',
        transition: 'opacity 0.5s, transform 0.5s',
      }}
    >
      {children}
    </div>
  );
}
```

Infinite scroll:

```tsx
function InfiniteList({ fetchMore }: { fetchMore: () => Promise<void> }) {
  const [items, setItems] = useState<Item[]>([]);
  const [loadingMoreRef, entry] = useIntersectionObserver<HTMLDivElement>({
    threshold: 1,
    rootMargin: '100px',
  });
  const isVisible = entry?.isIntersecting;
  
  useEffect(() => {
    if (isVisible) fetchMore();
  }, [isVisible, fetchMore]);
  
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
      <li ref={loadingMoreRef}>Loading more...</li>
    </ul>
  );
}
```

</details>

---

## `useFetch` — Async Data (AbortController, race-safe)

### Nazariya

`useFetch` — async data fetching uchun custom hook. State management (data, loading, error), AbortController bilan race condition prevention, refetch capability.

```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string | null, options?: RequestInit): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(!!url);
  const [error, setError] = useState<Error | null>(null);
  const [trigger, setTrigger] = useState(0);
  
  // options reference stable bo'lmasa (caller inline `{}` uzatadi),
  // har render yangi reference → infinite re-fetch.
  // To'g'ri yechim: optionsRef bilan latest closure (pastdagi to'liq versiyaga qarang).
  const optionsRef = useRef(options);
  useEffect(() => { optionsRef.current = options; });
  
  useEffect(() => {
    if (!url) {
      setData(null);
      setLoading(false);
      return;
    }
    
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    
    fetch(url, { ...optionsRef.current, signal: controller.signal })
      .then(async (response) => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json() as Promise<T>;
      })
      .then((result) => {
        setData(result);
        setLoading(false);
      })
      .catch((err) => {
        if (err.name === 'AbortError') return;
        setError(err);
        setLoading(false);
      });
    
    return () => controller.abort();
  }, [url, trigger]);  // options deps'da YO'Q (ref orqali latest)
  
  const refetch = useCallback(() => {
    setTrigger(t => t + 1);
  }, []);
  
  return { data, loading, error, refetch };
}
```

QANDAY ISHLAYDI:

1. `url` o'zgarsa — yangi fetch boshlanadi.
2. `AbortController` — eski fetch'ni cancel qiladi (race condition prevention).
3. Cleanup `controller.abort()` chaqiradi (yangi url yoki unmount).
4. `refetch` — `trigger` increment qiladi → useEffect qayta ishlaydi.
5. `null` url — fetch skip (conditional fetch).

NIMA UCHUN: react developerlar ko'pincha fetch logic'ni manual yozadi va race condition'ni unutadi. AbortController bilan eski fetch cancel qilinadi — komponent state correct.

**`useFetch` cheklov'lar** — production'da TanStack Query yoki SWR yaxshi:

| Feature | `useFetch` (custom) | TanStack Query |
|---------|---------------------|----------------|
| Caching | Yo'q | Built-in |
| Stale-while-revalidate | Yo'q | Built-in |
| Deduplication | Yo'q | Built-in |
| Retry | Manual | Configurable |
| Optimistic updates | Manual | Built-in |
| Window focus refetch | Manual | Built-in |
| Pagination | Manual | Built-in |
| Bundle size | Minimal (faqat shu hook) | Library overhead (versiya bo'yicha o'zgaradi) |

**Custom `useFetch`** — kichik proyektlar yoki learning uchun. Production'da TanStack Query yoki SWR ishlatish — wheel reinvent qilmaslik.

R19+ alternative — `use(promise)` (cross-ref [`23-r19-hooks.md`](23-r19-hooks.md)) Suspense bilan declarative.

<details>
<summary><strong>Under the Hood</strong></summary>

`AbortController` API:

```typescript
const controller = new AbortController();
controller.signal;     // AbortSignal — fetch'ga uzatiladi
controller.abort();    // signal abort qiladi → fetch reject AbortError
```

Browser support — Chrome 66+, Firefox 57+, Safari 11.1+. Universal modern.

`AbortError` — `error.name === 'AbortError'` check shart (catch block'da). Boshqa error'lar (network, parse) handle qilinadi.

Race condition scenario:

```
t=0:  url = 'A' → fetch A starts (controllerA)
t=10: url = 'B' → fetch B starts, cleanup → controllerA.abort()
t=20: A rejects (AbortError) — silent
t=50: B resolves → setData(B)

Without abort:
t=0:  url = 'A' → fetch A starts
t=10: url = 'B' → fetch B starts (no cancel)
t=20: A response arrives slow → setData(A) ❌ stale
t=50: B response arrives → setData(B)
```

Slow A response after B — stale data overrides correct B. AbortController bilan oldini olinadi.

`StrictMode` 2x cycle (R18+) `useFetch`'ni 2 marta chaqiradi (mount → unmount → mount). AbortController birinchi cleanup'da abort qiladi → ikkinchi mount yangi fetch.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Implementation:

```tsx
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

export function useFetch<T>(
  url: string | null,
  options?: RequestInit
): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(!!url);
  const [error, setError] = useState<Error | null>(null);
  const [trigger, setTrigger] = useState(0);
  
  // Caller inline `{}` uzatsa har render yangi reference bo'ladi.
  // optionsRef latest options'ni dep'siz saqlaydi — infinite re-fetch yo'q.
  const optionsRef = useRef(options);
  useEffect(() => {
    optionsRef.current = options;
  });
  
  useEffect(() => {
    if (!url) {
      setData(null);
      setLoading(false);
      return;
    }
    
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    
    fetch(url, { ...optionsRef.current, signal: controller.signal })
      .then(async (response) => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json() as Promise<T>;
      })
      .then((result) => {
        setData(result);
        setLoading(false);
      })
      .catch((err: unknown) => {
        if (err instanceof Error && err.name === 'AbortError') return;
        setError(err instanceof Error ? err : new Error(String(err)));
        setLoading(false);
      });
    
    return () => controller.abort();
  }, [url, trigger]);
  
  const refetch = useCallback(() => {
    setTrigger(t => t + 1);
  }, []);
  
  return { data, loading, error, refetch };
}
```

Basic usage:

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error, refetch } = useFetch<User>(`/api/users/${userId}`);
  
  if (loading) return <div>Loading...</div>;
  if (error) return (
    <div>
      Error: {error.message}
      <button onClick={refetch}>Retry</button>
    </div>
  );
  if (!user) return null;
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

Conditional fetch:

```tsx
function UserPostsPage({ userId }: { userId: string | null }) {
  const { data: user } = useFetch<User>(userId ? `/api/users/${userId}` : null);
  const { data: posts } = useFetch<Post[]>(user ? `/api/users/${user.id}/posts` : null);
  
  if (!userId) return <p>User tanlanmagan</p>;
  if (!user) return <p>Loading user...</p>;
  if (!posts) return <p>Loading posts...</p>;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
    </div>
  );
}
```

Search with debounce:

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  const { data, loading, error } = useFetch<Product[]>(
    debouncedQuery ? `/api/search?q=${debouncedQuery}` : null
  );
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      {loading && <div>Searching...</div>}
      {error && <div>Error: {error.message}</div>}
      {data && (
        <ul>
          {data.map(p => <li key={p.id}>{p.name}</li>)}
        </ul>
      )}
    </div>
  );
}
```

</details>

---

## TypeScript Generic Patterns

### Nazariya

Custom hook'lar uchun TypeScript generics — type-safe API yaratishga imkon beradi. Generic parameter'lar caller tomonidan inferred yoki explicit beriladi.

Generic naming convention (cross-ref [`10-props.md`](10-props.md)):

| Parameter | Maqsad | Misol |
|-----------|--------|-------|
| `T` | Generic data | `useFetch<T>(url): T \| null` |
| `K` | Key (object property) | `useField<T, K extends keyof T>(field: K)` |
| `E` | Element type | `useEventListener<E extends HTMLElement>` |
| `S`, `A` | State, Action | `useReducer<S, A>(reducer, initial)` |
| `R` | Return type | `useResource<T, R>(fetcher: () => T): R` |
| `P` | Props | `withHook<P>(Component)` |

**Generic constraints** — type narrowing:

```tsx
// T must be JSON-serializable
function useLocalStorage<T extends JSONValue>(key: string, initial: T) {
  // ...
}

type JSONValue = 
  | string 
  | number 
  | boolean 
  | null 
  | { [key: string]: JSONValue } 
  | JSONValue[];
```

**Generic with default**:

```tsx
function useToggle<T = boolean>(initial: T) {
  // T defaults to boolean
}

useToggle();           // T = boolean
useToggle('on');       // T = string
useToggle(0);          // T = number
```

**Multiple generics**:

```tsx
function useReducer<S, A>(
  reducer: (state: S, action: A) => S,
  initial: S
): [S, (action: A) => void] {
  // ...
}

// Caller infers from initial value
const [count, dispatch] = useReducer(
  (s: number, a: 'inc' | 'dec') => s + (a === 'inc' ? 1 : -1),
  0
);
// S = number, A = 'inc' | 'dec'
```

**Conditional types**:

```tsx
type UseFetchReturn<T, E = Error> = 
  | { status: 'loading'; data: null; error: null }
  | { status: 'success'; data: T; error: null }
  | { status: 'error'; data: null; error: E };

function useFetch<T, E = Error>(url: string): UseFetchReturn<T, E> {
  // Discriminated union return
}

// Usage — TypeScript narrows based on status
const result = useFetch<User>('/api/user');
if (result.status === 'success') {
  result.data;  // User (not null)
}
```

**`as const` for tuple inference**:

```tsx
function useToggle(initial: boolean = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle] as const;  // ← muhim
}

// Without as const: (boolean | (() => void))[]
// With as const: readonly [boolean, () => void]
```

`as const` siz TypeScript loose array type infer qiladi. `as const` tuple readonly type ko'rsatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript inference algorithms:

1. **Direct argument inference** — function call argument'lardan generic infer qilinadi.
2. **Return type inference** — function body'dan return type infer qilinadi.
3. **Constraint matching** — `T extends X` constraint'ga mos qiymat shart.
4. **Default fallback** — generic default qo'llaniladi (caller'da explicit yo'q).

```tsx
function useFetch<T>(url: string): { data: T | null } {
  return { data: null as T | null };
}

// Inference yo'q — explicit kerak
const result1 = useFetch('/api/user');     // T = unknown, data: unknown | null
const result2 = useFetch<User>('/api/user'); // T = User, data: User | null
```

`useFetch` return type — `T` argument'lardan inferred bo'lmaydi (URL string bilan T orasida bog'liqlik yo'q). Caller explicit type ber'shi kerak.

Inference kelishi mumkin holatlar:

```tsx
function useFetchTyped<T>(url: string, transform: (raw: any) => T): { data: T | null } {
  // ...
}

// transform callback'dan T inferred
const result = useFetchTyped('/api/user', (raw) => raw as User);
// T = User (transform return type)
```

React Compiler runtime memoization kod'ini generatsiya qiladi, TypeScript type'lariga tegmaydi — type information build pipeline'da o'zgarmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic `useToggle`:

```tsx
import { useState, useCallback } from 'react';

export function useToggle<T = boolean>(
  initial: T,
  ...values: T[]
): readonly [T, () => void] {
  const [value, setValue] = useState<T>(initial);
  
  const toggle = useCallback(() => {
    setValue((current) => {
      if (values.length === 0) {
        // Boolean default
        return !current as T;
      }
      // Cycle through values
      const allValues = [initial, ...values];
      const currentIndex = allValues.indexOf(current);
      const nextIndex = (currentIndex + 1) % allValues.length;
      return allValues[nextIndex];
    });
  }, [initial, values]);
  
  return [value, toggle] as const;
}

// Usage
const [isOpen, toggleOpen] = useToggle();              // T = boolean
const [theme, cycleTheme] = useToggle('light', 'dark', 'auto');  // T = string
const [count, cycleCount] = useToggle(1, 2, 3);        // T = number
```

Generic `useFetch`:

```tsx
type FetchState<T, E = Error> =
  | { status: 'idle'; data: null; error: null }
  | { status: 'loading'; data: null; error: null }
  | { status: 'success'; data: T; error: null }
  | { status: 'error'; data: null; error: E };

export function useFetch<T, E = Error>(
  url: string | null
): FetchState<T, E> & { refetch: () => void } {
  const [state, setState] = useState<FetchState<T, E>>({
    status: 'idle',
    data: null,
    error: null,
  });
  
  // ... fetch logic
  
  const refetch = useCallback(() => { /* ... */ }, []);
  
  return { ...state, refetch };
}

// Usage — discriminated union narrowing
function UserCard({ userId }: { userId: string }) {
  const result = useFetch<User>(`/api/users/${userId}`);
  
  switch (result.status) {
    case 'idle':
    case 'loading':
      return <Spinner />;
    case 'error':
      return <ErrorMsg error={result.error} />;  // E type
    case 'success':
      return <Profile user={result.data} />;     // T type, not null
  }
}
```

Generic `useFormField`:

```tsx
function useFormField<T extends Record<string, unknown>, K extends keyof T>(
  initial: T,
  field: K
): {
  value: T[K];
  setValue: (value: T[K]) => void;
  reset: () => void;
} {
  const [value, setValueState] = useState<T[K]>(initial[field]);
  
  const setValue = useCallback((newValue: T[K]) => {
    setValueState(newValue);
  }, []);
  
  const reset = useCallback(() => {
    setValueState(initial[field]);
  }, [initial, field]);
  
  return { value, setValue, reset };
}

// Usage
interface ContactFormData {
  name: string;
  email: string;
  age: number;
}

const initial: ContactFormData = { name: '', email: '', age: 0 };

const nameField = useFormField(initial, 'name');     // value: string
const emailField = useFormField(initial, 'email');   // value: string
const ageField = useFormField(initial, 'age');       // value: number
```

Generic `useEventListener` (overloads):

```tsx
import { useEffect, useRef, RefObject } from 'react';

// Overload signatures
export function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void
): void;

export function useEventListener<K extends keyof DocumentEventMap>(
  eventName: K,
  handler: (event: DocumentEventMap[K]) => void,
  element: Document
): void;

export function useEventListener<
  K extends keyof HTMLElementEventMap,
  T extends HTMLElement = HTMLElement
>(
  eventName: K,
  handler: (event: HTMLElementEventMap[K]) => void,
  element: RefObject<T>
): void;

// Implementation
export function useEventListener(
  eventName: string,
  handler: (event: any) => void,
  element?: any
): void {
  // ...
}

// Usage — TypeScript event types narrowed
useEventListener('keydown', (e) => {
  e.key;        // string
  e.ctrlKey;    // boolean
});

useEventListener('click', (e) => {
  e.clientX;    // number
}, buttonRef);
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Custom hook conditional ichida — Rules of Hooks

Custom hook ham hook — Rules of Hooks qo'llaniladi. `if`/`for`/early return ichida chaqirish — ESLint error va silent state corruption.

```tsx
// ❌ Anti-pattern
function Component({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const data = useFetch('/api/data');  // ❌ conditional custom hook
    return <div>{data?.name}</div>;
  }
  return null;
}

// ✅ To'g'ri — har doim chaqirish, internal conditional
function Component({ enabled }: { enabled: boolean }) {
  const data = useFetch(enabled ? '/api/data' : null);  // ✅ null parameter
  return enabled && data ? <div>{data.name}</div> : null;
}
```

Custom hook'lar **`null` parameter** orqali "disabled" holatni qo'llab-quvvatlashi tavsiya qilinadi.

### Gotcha 2: Custom hook return value har render'da yangi reference

Object return — har render yangi reference. Consumer `React.memo` bilan ham re-render bo'ladi:

```tsx
// ❌ Har render yangi object
function useCounter(initial: number) {
  const [count, setCount] = useState(initial);
  return { count, setCount };  // har render new object
}

// Consumer
const Counter = React.memo(({ counter }: { counter: ReturnType<typeof useCounter> }) => {
  return <div>{counter.count}</div>;
});

function App() {
  const counter = useCounter(0);
  return <Counter counter={counter} />;  // ❌ har render re-render
}
```

**Yechim 1** — useMemo:

```tsx
function useCounter(initial: number) {
  const [count, setCount] = useState(initial);
  return useMemo(() => ({ count, setCount }), [count]);
}
```

**Yechim 2** — destructure consumer'da:

```tsx
function App() {
  const { count } = useCounter(0);  // ← faqat primitive
  return <Counter count={count} />;
}
```

**Yechim 3** — React Compiler auto-memoization (manual `useMemo` kerak emas).

### Gotcha 3: Latest closure trap event handler'larda

Custom hook event handler'da render closure variable ishlatsa — stale value:

```tsx
// ❌ Stale closure
function useKeyPress(targetKey: string, handler: () => void) {
  useEffect(() => {
    const listener = (e: KeyboardEvent) => {
      if (e.key === targetKey) handler();  // ❌ handler stale
    };
    window.addEventListener('keydown', listener);
    return () => window.removeEventListener('keydown', listener);
  }, [targetKey]);  // handler dep yo'q → stale
}

// ✅ Latest closure
function useKeyPress(targetKey: string, handler: () => void) {
  const handlerRef = useRef(handler);
  
  useEffect(() => {
    handlerRef.current = handler;
  });
  
  useEffect(() => {
    const listener = (e: KeyboardEvent) => {
      if (e.key === targetKey) handlerRef.current();  // ✅ latest
    };
    window.addEventListener('keydown', listener);
    return () => window.removeEventListener('keydown', listener);
  }, [targetKey]);
}
```

### Gotcha 4: SSR — `window` access initial render'da

`window`/`document`/`localStorage` server'da mavjud emas. Direct access — runtime error:

```tsx
// ❌ SSR'da error: ReferenceError: window is not defined
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);  // ❌ initial render
  // ...
}

// ✅ SSR-safe
function useWindowWidth() {
  const [width, setWidth] = useState<number | null>(null);
  
  useEffect(() => {
    setWidth(window.innerWidth);
    // listener...
  }, []);
  
  return width;  // null'gacha SSR
}

// Yoki lazy initializer
function useWindowWidth() {
  const [width, setWidth] = useState(() => 
    typeof window !== 'undefined' ? window.innerWidth : 0
  );
  // ...
}
```

### Gotcha 5: `useEffect` cleanup — old reference saqlash

Cleanup function render closure'idan reference saqlaydi. Async cleanup'da bu muammo bo'lishi mumkin:

```tsx
function useTimer(callback: () => void, delay: number) {
  useEffect(() => {
    const id = setTimeout(callback, delay);
    return () => clearTimeout(id);  // ✅ id closure'da
  }, [callback, delay]);
}

// ❌ Anti-pattern — id ref'da
function useTimerBad(callback: () => void, delay: number) {
  const idRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  
  useEffect(() => {
    idRef.current = setTimeout(callback, delay);
    return () => {
      if (idRef.current) clearTimeout(idRef.current);  // ❌ old id race
    };
  }, [callback, delay]);
}
```

Cleanup function variable closure orqali — bu correct pattern. Ref orqali — race condition (yangi mount eski cleanup'gacha set qilishi mumkin).

---

## Common Mistakes

### ❌ Xato 1: `use` prefix sodda function'da

```tsx
// ❌ Noto'g'ri — hook emas, lekin `use` prefix
function useFormatPrice(price: number): string {
  return `${price} so'm`;  // sodda formatter
}

// ✅ To'g'ri — sodda function
function formatPrice(price: number): string {
  return `${price} so'm`;
}
```

**Sabab:** ESLint Rules of Hooks `use*` function'da static analysis qiladi. Sodda function'da hook chaqirilmasa — keraksiz check.

### ❌ Xato 2: Hook'siz function'da `use` prefix yo'q

```tsx
// ❌ Noto'g'ri — hook ichida useState lekin use prefix yo'q
function counter() {
  const [count, setCount] = useState(0);  // ESLint warning
  return count;
}

// ✅ To'g'ri
function useCounter() {
  const [count, setCount] = useState(0);
  return count;
}
```

**Sabab:** `use` prefix ESLint plugin'ga signal — function hook ekanligini bildiradi. Aks holda Rules of Hooks check qilinmaydi.

### ❌ Xato 3: Custom hook conditional chaqirish

```tsx
// ❌ Noto'g'ri
function Component({ flag }: { flag: boolean }) {
  if (flag) {
    const value = useCustomHook();  // ❌ conditional
    return <div>{value}</div>;
  }
  return null;
}

// ✅ To'g'ri — har doim chaqirish, internal conditional
function Component({ flag }: { flag: boolean }) {
  const value = useCustomHook(flag);  // hook accepts disabled flag
  return flag ? <div>{value}</div> : null;
}
```

**Sabab:** Hook count buzilsa — silent state corruption yoki "Rendered fewer/more hooks" runtime error.

### ❌ Xato 4: SSR'da `window`/`localStorage` access

```tsx
// ❌ Noto'g'ri — Next.js'da error
function useTheme() {
  const [theme, setTheme] = useState(localStorage.getItem('theme'));  // ❌ SSR
  // ...
}

// ✅ To'g'ri
function useTheme() {
  const [theme, setTheme] = useState<string | null>(null);
  
  useEffect(() => {
    setTheme(localStorage.getItem('theme'));
  }, []);
}
```

**Sabab:** Server'da `window`/`localStorage` mavjud emas. Initial render server-side bajarilsa — runtime error.

### ❌ Xato 5: Cleanup yo'q — memory leak

```tsx
// ❌ Cleanup yo'q
function useTimer(callback: () => void) {
  useEffect(() => {
    const id = setInterval(callback, 1000);
    // ❌ cleanup yo'q
  }, [callback]);
}

// ✅ Cleanup
function useTimer(callback: () => void) {
  useEffect(() => {
    const id = setInterval(callback, 1000);
    return () => clearInterval(id);
  }, [callback]);
}
```

**Sabab:** Komponent unmount bo'lsa, interval ishlashda davom etadi → memory leak + unmounted state update warning.

---

## Amaliy Mashqlar

### Mashq 1: `useToggle` (Oson)

`useToggle(initial)` hook yarating — boolean state va toggle function qaytaradi. `setValue` ham qaytaradi (manual set imkoni).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useCallback } from 'react';

export function useToggle(
  initial: boolean = false
): readonly [boolean, () => void, (value: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  const set = useCallback((newValue: boolean) => setValue(newValue), []);
  
  return [value, toggle, set] as const;
}

// Usage
function Sidebar() {
  const [isOpen, toggleSidebar, setIsOpen] = useToggle();
  
  return (
    <>
      <button onClick={toggleSidebar}>Toggle</button>
      <button onClick={() => setIsOpen(false)}>Close</button>
      <button onClick={() => setIsOpen(true)}>Open</button>
      {isOpen && <aside>Sidebar content</aside>}
    </>
  );
}
```

**Tushuntirish:**
- `as const` tuple readonly type infer qiladi.
- `useCallback` toggle/set referentlarini stable qiladi.
- Ko'p ishlatiladi — modal, dropdown, sidebar.

</details>

### Mashq 2: `usePrevious` + warning (Oson)

`usePrevious` standart implementation. `useStateLogger` qo'shing — state o'zgarsa console'da log.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useRef, useEffect } from 'react';

export function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

export function useStateLogger<T>(value: T, label: string = 'state') {
  const prev = usePrevious(value);
  
  useEffect(() => {
    if (prev !== undefined && prev !== value) {
      console.log(
        `[${label}]`,
        JSON.stringify(prev, null, 2),
        '→',
        JSON.stringify(value, null, 2)
      );
    }
  }, [value, prev, label]);
}

// Usage
function UserProfile({ user }: { user: User }) {
  useStateLogger(user, 'user');
  // user o'zgarsa console'da log: [user] {old} → {new}
  
  return <div>{user.name}</div>;
}
```

**Tushuntirish:**
- `usePrevious` `useRef` + `useEffect` pattern.
- `useStateLogger` `usePrevious`'ni compose qiladi.
- Composition pattern misol — primitive hook (usePrevious) → feature hook (useStateLogger).

</details>

### Mashq 3: `useDebouncedSearch` (O'rta)

`useDebouncedSearch<T>(searchFn, delay)` hook — query state, debounced search, results, loading state. Race condition prevention `AbortController` bilan.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect, useRef } from 'react';

interface UseDebouncedSearchOptions<T> {
  searchFn: (query: string, signal: AbortSignal) => Promise<T[]>;
  delay?: number;
  minQueryLength?: number;
}

interface UseDebouncedSearchResult<T> {
  query: string;
  setQuery: (q: string) => void;
  results: T[];
  loading: boolean;
  error: Error | null;
}

export function useDebouncedSearch<T>({
  searchFn,
  delay = 300,
  minQueryLength = 2,
}: UseDebouncedSearchOptions<T>): UseDebouncedSearchResult<T> {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const searchFnRef = useRef(searchFn);
  
  useEffect(() => {
    searchFnRef.current = searchFn;
  });
  
  useEffect(() => {
    if (query.length < minQueryLength) {
      setResults([]);
      setLoading(false);
      setError(null);
      return;
    }
    
    const controller = new AbortController();
    const timer = setTimeout(async () => {
      setLoading(true);
      setError(null);
      try {
        const data = await searchFnRef.current(query, controller.signal);
        setResults(data);
      } catch (err) {
        if (err instanceof Error && err.name !== 'AbortError') {
          setError(err);
        }
      } finally {
        setLoading(false);
      }
    }, delay);
    
    return () => {
      clearTimeout(timer);
      controller.abort();
    };
  }, [query, delay, minQueryLength]);
  
  return { query, setQuery, results, loading, error };
}

// Usage
interface Product {
  id: string;
  name: string;
  price: number;
}

function ProductSearch() {
  const { query, setQuery, results, loading, error } = useDebouncedSearch<Product>({
    searchFn: async (q, signal) => {
      const res = await fetch(`/api/products/search?q=${encodeURIComponent(q)}`, { signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    },
    delay: 300,
    minQueryLength: 2,
  });
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Mahsulot qidirish..."
      />
      {loading && <span>Qidirilmoqda...</span>}
      {error && <span>Error: {error.message}</span>}
      <ul>
        {results.map(p => (
          <li key={p.id}>{p.name} — {p.price} so'm</li>
        ))}
      </ul>
    </div>
  );
}
```

**Tushuntirish:**
- Debounce + AbortController + latest closure pattern.
- `searchFnRef` — function dep'siz stable listener.
- `minQueryLength` UX optimization — short query'larda search yo'q.
- `controller.abort()` cleanup — pending fetch cancel.

</details>

### Mashq 4: `useLocalStorage` cross-tab sync (O'rta)

`useLocalStorage<T>` hook — full implementation. Cross-tab sync `storage` event orqali. Type-safe `JSON.parse` (try/catch).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect, useCallback } from 'react';

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): readonly [T, (value: T | ((prev: T) => T)) => void, () => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;
    try {
      const item = window.localStorage.getItem(key);
      return item !== null ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      console.warn(`useLocalStorage: Error reading "${key}":`, error);
      return initialValue;
    }
  });
  
  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      try {
        setStoredValue(prev => {
          const newValue = value instanceof Function ? value(prev) : value;
          if (typeof window !== 'undefined') {
            window.localStorage.setItem(key, JSON.stringify(newValue));
          }
          return newValue;
        });
      } catch (error) {
        console.warn(`useLocalStorage: Error writing "${key}":`, error);
      }
    },
    [key]
  );
  
  const remove = useCallback(() => {
    try {
      if (typeof window !== 'undefined') {
        window.localStorage.removeItem(key);
        setStoredValue(initialValue);
      }
    } catch (error) {
      console.warn(`useLocalStorage: Error removing "${key}":`, error);
    }
  }, [key, initialValue]);
  
  // Cross-tab sync
  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    const handler = (e: StorageEvent) => {
      if (e.key !== key) return;
      
      if (e.newValue === null) {
        // Removed in another tab
        setStoredValue(initialValue);
        return;
      }
      
      try {
        setStoredValue(JSON.parse(e.newValue) as T);
      } catch (error) {
        console.warn(`useLocalStorage: Error parsing "${key}" from event:`, error);
      }
    };
    
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, [key, initialValue]);
  
  return [storedValue, setValue, remove] as const;
}

// Usage
function ThemeToggle() {
  const [theme, setTheme, removeTheme] = useLocalStorage<'light' | 'dark'>(
    'theme',
    'light'
  );
  
  return (
    <div>
      <p>Current theme: {theme}</p>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <button onClick={removeTheme}>Reset</button>
    </div>
  );
}
```

**Tushuntirish:**
- Lazy initializer — SSR-safe + JSON.parse error handle.
- Functional updates support (`(prev) => next`).
- `storage` event — cross-tab sync.
- `remove` function — explicit cleanup.
- Tuple return readonly.

</details>

### Mashq 5: `useFetch` with cache + retry (Qiyin)

`useFetch<T>(url, options)` hook — cache (Map by url), retry (max attempts), AbortController. Concurrent rendering safe.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseFetchOptions extends RequestInit {
  retry?: number;
  retryDelay?: number;
  cache?: 'memory' | 'none';
  cacheTTL?: number;  // ms
}

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

interface CacheEntry<T> {
  data: T;
  timestamp: number;
}

const memoryCache = new Map<string, CacheEntry<unknown>>();

export function useFetch<T>(
  url: string | null,
  options: UseFetchOptions = {}
): UseFetchResult<T> {
  const {
    retry = 0,
    retryDelay = 1000,
    cache = 'memory',
    cacheTTL = 60_000,  // 1 minute
    ...fetchOptions
  } = options;
  
  const [data, setData] = useState<T | null>(() => {
    if (url && cache === 'memory') {
      const entry = memoryCache.get(url) as CacheEntry<T> | undefined;
      if (entry && Date.now() - entry.timestamp < cacheTTL) {
        return entry.data;
      }
    }
    return null;
  });
  
  const [loading, setLoading] = useState(!!url && !data);
  const [error, setError] = useState<Error | null>(null);
  const [trigger, setTrigger] = useState(0);
  const optionsRef = useRef(fetchOptions);
  
  useEffect(() => {
    optionsRef.current = fetchOptions;
  });
  
  useEffect(() => {
    if (!url) {
      setData(null);
      setLoading(false);
      setError(null);
      return;
    }
    
    // Check cache
    if (cache === 'memory') {
      const entry = memoryCache.get(url) as CacheEntry<T> | undefined;
      if (entry && Date.now() - entry.timestamp < cacheTTL) {
        setData(entry.data);
        setLoading(false);
        return;
      }
    }
    
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    
    const attemptFetch = async (attempt: number): Promise<void> => {
      try {
        const response = await fetch(url, {
          ...optionsRef.current,
          signal: controller.signal,
        });
        
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        const result = (await response.json()) as T;
        
        if (cache === 'memory') {
          memoryCache.set(url, { data: result, timestamp: Date.now() });
        }
        
        setData(result);
        setLoading(false);
      } catch (err) {
        if (err instanceof Error && err.name === 'AbortError') return;
        
        if (attempt < retry) {
          // Retry with exponential backoff
          const delay = retryDelay * Math.pow(2, attempt);
          await new Promise(resolve => setTimeout(resolve, delay));
          if (!controller.signal.aborted) {
            return attemptFetch(attempt + 1);
          }
          return;
        }
        
        setError(err instanceof Error ? err : new Error(String(err)));
        setLoading(false);
      }
    };
    
    attemptFetch(0);
    
    return () => controller.abort();
  }, [url, trigger, retry, retryDelay, cache, cacheTTL]);
  
  const refetch = useCallback(() => {
    if (url && cache === 'memory') {
      memoryCache.delete(url);  // Cache invalidate
    }
    setTrigger(t => t + 1);
  }, [url, cache]);
  
  return { data, loading, error, refetch };
}

// Usage
interface User {
  id: string;
  name: string;
  email: string;
}

function UserCard({ userId }: { userId: string }) {
  const { data: user, loading, error, refetch } = useFetch<User>(
    `/api/users/${userId}`,
    { retry: 3, retryDelay: 500, cacheTTL: 30_000 }
  );
  
  if (loading) return <Spinner />;
  if (error) return (
    <div>
      Error: {error.message}
      <button onClick={refetch}>Retry</button>
    </div>
  );
  if (!user) return null;
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={refetch}>Refresh</button>
    </div>
  );
}
```

**Tushuntirish:**
- Module-level `Map` — komponent instance'lar orasida share.
- Cache TTL — eski data'ni avtomatik invalidate.
- Exponential backoff — retry delay 500ms → 1000ms → 2000ms.
- AbortController + retry — controller signal har attempt'da check.
- `refetch` cache invalidate.
- **Production warning:** TanStack Query yaxshi (deduplication, window focus refetch, devtools, va h.k.).

</details>

---

## Xulosa

Custom hooks — React'ning eng kuchli pattern'larini ifodalovchi konstruktsiya. Asosiy fikrlar:

- **Custom hook = `use*` function** — boshqa hook'lar ichida chaqirilishi mumkin. React feature emas, **konvensiya**. Hech qanday "registratsiya" yo'q — function yozasiz va `use` prefix bilan nomlaysiz.
- **`use*` prefix functional ahamiyatga ega** — ESLint plugin (`react-hooks/rules-of-hooks`) `/^use[A-Z0-9]/` regex orqali function'ni hook deb identify qiladi va Rules of Hooks tekshiradi. React Compiler (1.0 stable 2025-oktyabr; React'ga bundle qilinmagan opt-in Babel plugin; React 17/18/19 mos) ham shu konvensiyaga tayanib auto-memoization qiladi.
- **Logic extraction pattern** — komponent ichidagi takrorlanadigan logic'ni custom hook'ga ko'chirish. Single Responsibility, reusability, testability. Premature abstraction xavfi — faqat 2+ joyda ishlatilsa yoki concern alohida bo'lsa extract qilinadi.
- **Hook composition** — custom hook'lar boshqa custom hook'lar ichida. Layered architecture (primitive → domain → feature → page). Hook linked list flat — composition layer'lari React internal'da farqlanmaydi.
- **Parameters va Return Types** — single value (1) / tuple (2) / object (3+). Tuple — destructuring rename oson (`const [count, setCount]`). Object — partial destructuring + named property. `as const` tuple readonly type infer qiladi.
- **`useDebugValue`** — DevTools-only, custom hook label/value qo'shish. Lazy formatter (production'da no-op). Faqat custom hook ichida (komponent'da silent no-op).
- **Common Toolkit** — `usePrevious`, `useDebounce`, `useDebouncedCallback`, `useLocalStorage`, `useMediaQuery`, `useWindowSize`, `useEventListener`, `useOnClickOutside`, `useIntersectionObserver`, `useFetch`. Har biri production-grade implementation (SSR-safe, cleanup, race-condition free).
- **`usePrevious`** — `useRef` + `useEffect` pattern. Bir cycle delay (har render eski qiymat). Animation trigger, state transition logging.
- **`useDebounce` va `useDebouncedCallback`** — value-based vs function-based. Cleanup `clearTimeout`. Latest closure pattern (callback ref). `useDeferredValue` (R18+) priority-based alternative.
- **`useLocalStorage`** — `useState` API + localStorage sync. SSR-safe (lazy initializer + `typeof window` check). Cross-tab sync `storage` event. Try-catch JSON parse error.
- **`useMediaQuery`** — `MediaQueryList` API + `change` event. Responsive design (mobile/desktop, dark mode, reduced motion). `useSyncExternalStore` (R18+) Concurrent-safe variant.
- **`useWindowSize`** — `window.innerWidth/innerHeight` + `resize` event. Performance issue (har resize re-render) — debounce yoki `ResizeObserver` (element-specific).
- **`useEventListener`** — universal event subscription. Latest closure pattern (handler ref). TypeScript overloads (Window, Document, HTMLElement). Stable listener (re-attach kam).
- **`useOnClickOutside`** — `Element.contains()` + `mousedown`/`touchstart`. Modal, dropdown, popover. Modal yopish uchun `Escape` key + click outside + close button — uchovi accessibility.
- **`useIntersectionObserver`** — async visibility detection. Lazy image, infinite scroll, animation trigger. `freezeOnceVisible` — bir marta animatsiya. Browser optimized — `getBoundingClientRect` sync alternative emas.
- **`useFetch`** — async data + AbortController + retry. Production'da TanStack Query yoki SWR yaxshi (caching, deduplication, devtools). R19+ `use(promise)` Suspense bilan declarative alternativa.
- **TypeScript Generic Patterns** — `T` (data), `K` (key), `E` (element), `R` (return). Constraints (`extends`), defaults (`T = boolean`), inference (function arguments → return type), `as const` tuple, discriminated union return type.
- **Edge Cases** — Custom hook conditional ichida TAQIQ, return value memoization, latest closure trap, SSR `window` access, cleanup lifecycle.
- **Common Mistakes** — `use` prefix sodda function'da, hook'siz function'da `use` yo'q, custom hook conditional, SSR `window`, cleanup yo'q.

Cross-references:

- [`12-state-and-usestate.md`](12-state-and-usestate.md) — `useState` lazy initializer pattern
- [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) — Hook linked list, Rules of Hooks
- [`16-useeffect.md`](16-useeffect.md) — `useEffect` cleanup, AbortController
- [`17-uselayouteffect.md`](17-uselayouteffect.md) — Layout vs Effect timing
- [`18-useref.md`](18-useref.md) — Latest closure pattern, mutable container
- [`21-usememo-usecallback.md`](21-usememo-usecallback.md) — Return value memoization
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useSyncExternalStore` Concurrent-safe
- [`23-r19-hooks.md`](23-r19-hooks.md) — `use(promise)` declarative alternative
- [`28-portals.md`](28-portals.md) — Portal va event handling
- [`31-react-compiler.md`](31-react-compiler.md) — Auto-memoization, Rules of React

---

**Keyingi bo'lim:** [25-legacy-patterns.md](25-legacy-patterns.md) — Legacy patterns (Render Props, Higher-Order Components). Hooks oldin dominant pattern'lar, hozir kamdan-kam ishlatiladi, lekin eski codebase'larda mavjud. Render Props (`<Provider>{(value) => ...}</Provider>` function-as-children), HOC (`withAuth(Component)` enhancer pattern), inversion of control, prop collision muammosi, hooks bilan equivalent migration. QISM 7 (Advanced Patterns) boshlanishi.
