# Interview: Design Patterns TypeScript da

> Factory, Strategy, Adapter, Result/Either, EventEmitter, Repository, Builder, Middleware Pipeline — type-safe design patterns bo'yicha interview savollari. Singleton — [interview/10](10-classes.md), State Machine — [interview/05](05-unions-intersections.md).

---

## Nazariy savollar

### 1. Factory pattern — generic factory qanday yoziladi?

<details>
<summary>Javob</summary>

Factory — object yaratish logikasini yashiradi. TypeScript da generic factory type inference bilan kuchli:

```typescript
interface NotificationMap {
  email: EmailNotification;
  sms: SMSNotification;
}

function createNotification<K extends keyof NotificationMap>(
  type: K
): NotificationMap[K] {
  const map: { [P in keyof NotificationMap]: new () => NotificationMap[P] } = {
    email: EmailNotification,
    sms: SMSNotification,
  };
  return new map[type]() as NotificationMap[K];
}

const email = createNotification("email");
//    ^? EmailNotification — auto-inferred!
// createNotification("telegram"); // ❌ Compile error
```

Generic factory afzalligi — return type avtomatik aniqlanadi, type assertion kerak emas.

</details>

### 2. Strategy pattern nima? Qachon ishlatiladi?

<details>
<summary>Javob</summary>

Strategy — algoritmni **interface orqali abstract** qilib, runtime da almashtirishga imkon beradi:

```typescript
interface SortStrategy<T> {
  sort(data: T[]): T[];
}

class BubbleSort<T> implements SortStrategy<T> {
  sort(data: T[]): T[] { /* bubble sort */ return [...data].sort(); }
}
class QuickSort<T> implements SortStrategy<T> {
  sort(data: T[]): T[] { /* quick sort */ return [...data].sort(); }
}

class DataProcessor<T> {
  constructor(private strategy: SortStrategy<T>) {}
  setStrategy(s: SortStrategy<T>) { this.strategy = s; }
  process(data: T[]): T[] { return this.strategy.sort(data); }
}

const processor = new DataProcessor<number>(new BubbleSort());
processor.process([5, 3, 1]); // BubbleSort
processor.setStrategy(new QuickSort());
processor.process([5, 3, 1]); // QuickSort — runtime da almashdi
```

**Open/Closed Principle** — yangi strategy qo'shish uchun mavjud kod o'zgarmaydi. Validation, formatting, pricing, sorting — turli algoritmlar uchun.

</details>

### 3. Adapter pattern — real-world misol.

<details>
<summary>Javob</summary>

Adapter — mos kelmaydigan interface larni moslashtiradi. Legacy API ni yangi interface ga o'rab:

```typescript
interface PaymentProcessor {
  charge(amount: number, currency: string): Promise<{ transactionId: string }>;
  refund(transactionId: string): Promise<boolean>;
}

// Legacy API — boshqa interface
class LegacyPayPal {
  makePayment(cents: number, curr: string): Promise<string> { /* ... */ }
  reversePayment(id: string): Promise<{ success: boolean }> { /* ... */ }
}

// Adapter — LegacyPayPal ni PaymentProcessor ga moslashtiradi
class PayPalAdapter implements PaymentProcessor {
  constructor(private paypal: LegacyPayPal) {}

  async charge(amount: number, currency: string) {
    const id = await this.paypal.makePayment(amount * 100, currency);
    return { transactionId: id };
  }

  async refund(transactionId: string) {
    const result = await this.paypal.reversePayment(transactionId);
    return result.success;
  }
}

// Client kod faqat PaymentProcessor bilan ishlaydi
function processOrder(payment: PaymentProcessor) {
  return payment.charge(99.99, "USD");
}
processOrder(new PayPalAdapter(new LegacyPayPal())); // ✅
```

Client kod `PaymentProcessor` interface dan xabardor — legacy implementation yashirin.

</details>

### 4. Result/Either pattern nima uchun kerak?

<details>
<summary>Javob</summary>

Exception o'rniga `Ok<T> | Err<E>` discriminated union — xatolarni **type-safe** handle qilish:

```typescript
type Result<T, E> =
  | { readonly _tag: "Ok"; readonly value: T }
  | { readonly _tag: "Err"; readonly error: E };

function ok<T>(value: T): Result<T, never> { return { _tag: "Ok", value }; }
function err<E>(error: E): Result<never, E> { return { _tag: "Err", error }; }
```

Exception dan afzalligi — compiler xato holatini handle qilishni **majburlaydi**:

```typescript
function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return err("Division by zero");
  return ok(a / b);
}

const result = divide(10, 0);
if (result._tag === "Ok") {
  console.log(result.value);  // number
} else {
  console.log(result.error);  // string — handle qilish MAJBURIY
}
```

`catch` ni unutish mumkin, lekin `Result` da ikkala branch ni ko'rmaslik mumkin emas.

</details>

### 5. Middleware pipeline — onion model nima?

<details>
<summary>Javob</summary>

Middleware lar ichma-ich ishlaydi. `next()` dan oldingi kod "kirish", keyin "chiqish" (A→B→C→B→A):

```typescript
type Middleware = (ctx: { value: number }, next: () => void) => void;

const pipeline = new Pipeline()
  .use((ctx, next) => {
    console.log("A before:", ctx.value); // 1-kirish
    ctx.value *= 2;
    next();
    console.log("A after:", ctx.value);  // 5-chiqish
  })
  .use((ctx, next) => {
    console.log("B before:", ctx.value); // 2-kirish
    ctx.value += 10;
    next();
    console.log("B after:", ctx.value);  // 4-chiqish
  })
  .use((ctx) => {
    console.log("C:", ctx.value);        // 3-markaz
  });

pipeline.execute({ value: 5 });
// A before: 5 → B before: 10 → C: 20 → B after: 20 → A after: 20
```

**Chain of Responsibility** pattern ning composable versiyasi. Express, Koa middleware aynan shunday.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Type-safe `EventEmitter<Events>` implement qiling (Daraja: Senior)

**Savol:** `on`, `emit`, `once`, `removeAllListeners` — barcha event name va argument type lari compile-time da tekshirilsin:

```typescript
interface UserEvents {
  login: [userId: string, timestamp: Date];
  logout: [userId: string];
  purchase: [userId: string, product: string, amount: number];
}
// emitter.on("login", (userId, timestamp) => { ... }) — type-safe
// emitter.emit("login", "user-1", new Date()) — ✅
// emitter.emit("login", "user-1") — ❌ timestamp kerak
```

<details>
<summary>Yechim</summary>

```typescript
type EventMap = Record<string, unknown[]>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<keyof Events, Set<(...args: any[]) => void>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(handler);
    return () => { this.listeners.get(event)?.delete(handler); };
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    this.listeners.get(event)?.forEach(h => h(...args));
  }

  once<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    const unsub = this.on(event, (...args) => { handler(...args); unsub(); });
    return unsub;
  }

  removeAllListeners(event?: keyof Events): void {
    event ? this.listeners.delete(event) : this.listeners.clear();
  }
}

const emitter = new TypedEventEmitter<UserEvents>();
emitter.on("login", (userId, timestamp) => {
  console.log(`${userId} at ${timestamp.toISOString()}`); // TS infer: string, Date
});
emitter.emit("login", "user-1", new Date()); // ✅
// emitter.emit("signup", "x"); // ❌ 'signup' not in Events
```

`K extends keyof Events` + `Events[K]` tuple — har event uchun aniq argument type lar.

</details>

### 2. Generic `Result<T, E>` — utility funksiyalar bilan (Daraja: Middle+)

**Savol:** `Result<T, E>` type, `ok`/`err` constructors, `map`, `flatMap`, `unwrapOr` yozing:

```typescript
// map(ok(5), n => n * 2) → ok(10)
// map(err("fail"), n => n * 2) → err("fail")
// flatMap(ok(5), n => n > 0 ? ok(n) : err("negative")) → ok(5)
// unwrapOr(err("fail"), 0) → 0
```

<details>
<summary>Yechim</summary>

```typescript
type Ok<T> = { readonly _tag: "Ok"; readonly value: T };
type Err<E> = { readonly _tag: "Err"; readonly error: E };
type Result<T, E> = Ok<T> | Err<E>;

function ok<T>(value: T): Ok<T> { return { _tag: "Ok", value }; }
function err<E>(error: E): Err<E> { return { _tag: "Err", error }; }

function isOk<T, E>(r: Result<T, E>): r is Ok<T> { return r._tag === "Ok"; }

function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return isOk(r) ? ok(fn(r.value)) : r;
}

function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return isOk(r) ? fn(r.value) : r;
}

function unwrapOr<T, E>(r: Result<T, E>, defaultValue: T): T {
  return isOk(r) ? r.value : defaultValue;
}

// Railway-oriented programming
function validateAge(age: number): Result<number, string> {
  return age >= 0 && age <= 150 ? ok(age) : err("Invalid age");
}

const result = flatMap(validateAge(25), age =>
  age >= 18 ? ok(`Adult: ${age}`) : err("Too young")
);
// ok("Adult: 25")
```

Exception dan afzal — compiler ikkala holatni handle qilishni majburlaydi.

</details>

### 3. Generic `Repository<T>` — interface + in-memory (Daraja: Middle+)

**Savol:** CRUD repository interface va InMemoryRepository yozing. `create` da `id`/dates avtomatik bo'lsin:

```typescript
// const user = await repo.create({ name: "Ali", email: "a@b.com" });
// user.id, user.createdAt — avtomatik
// await repo.findAll({ role: "admin" }) — filter bilan
```

<details>
<summary>Yechim</summary>

```typescript
interface Entity { id: string; createdAt: Date; updatedAt: Date; }

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(id: string, data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

class InMemoryRepository<T extends Entity> implements Repository<T> {
  protected items = new Map<string, T>();

  async findById(id: string) { return this.items.get(id) ?? null; }

  async findAll(filter?: Partial<T>) {
    let results = Array.from(this.items.values());
    if (filter) {
      results = results.filter(item =>
        Object.entries(filter).every(([k, v]) => item[k as keyof T] === v)
      );
    }
    return results;
  }

  async create(data: Omit<T, "id" | "createdAt" | "updatedAt">) {
    const now = new Date();
    const entity = { ...data, id: crypto.randomUUID(), createdAt: now, updatedAt: now } as T;
    this.items.set(entity.id, entity);
    return entity;
  }

  async update(id: string, data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>) {
    const existing = this.items.get(id);
    if (!existing) throw new Error(`Not found: ${id}`);
    const updated = { ...existing, ...data, updatedAt: new Date() } as T;
    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string) { return this.items.delete(id); }
}

interface User extends Entity { name: string; email: string; role: "admin" | "user"; }
const repo: Repository<User> = new InMemoryRepository<User>();
const user = await repo.create({ name: "Ali", email: "a@b.com", role: "admin" });
// id, createdAt, updatedAt avtomatik
```

`Omit<T, "id" | "createdAt" | "updatedAt">` — compile-time da noto'g'ri data oldini oladi.

</details>

### 4. Type-safe Builder — required field lar compile-time da (Daraja: Senior)

**Savol:** `build()` faqat `url` VA `method` set bo'lganda chaqirilsin. Bo'lmasa compile error:

```typescript
// new Builder().setUrl("/api").setMethod("POST").build() → ✅
// new Builder().setUrl("/api").build() → ❌ method yo'q
```

<details>
<summary>Yechim</summary>

```typescript
interface HttpRequest {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers: Record<string, string>;
  body?: unknown;
}

type BuilderState = Partial<Record<"url" | "method", true>>;

class RequestBuilder<State extends BuilderState = {}> {
  private data: Partial<HttpRequest> = {};

  setUrl(url: string): RequestBuilder<State & { url: true }> {
    this.data.url = url;
    return this as any;
  }

  setMethod(m: HttpRequest["method"]): RequestBuilder<State & { method: true }> {
    this.data.method = m;
    return this as any;
  }

  addHeader(k: string, v: string): RequestBuilder<State> {
    this.data.headers = { ...this.data.headers, [k]: v };
    return this as any;
  }

  // build() faqat url VA method bor bo'lganda
  build(this: RequestBuilder<{ url: true; method: true }>): HttpRequest {
    return {
      url: this.data.url!, method: this.data.method!,
      headers: this.data.headers ?? {}, body: this.data.body,
    };
  }
}

new RequestBuilder().setUrl("/api").setMethod("POST").build(); // ✅
// new RequestBuilder().setUrl("/api").build();
// ❌ 'RequestBuilder<{ url: true }>' not assignable to '{ url: true; method: true }'
```

**Phantom types** — `State` generic har setter dan yangilanadi. `build()` ning `this` parametri faqat to'liq state da match bo'ladi. Runtime da overhead yo'q.

</details>

### 5. Type-safe middleware pipeline (async) (Daraja: Senior)

**Savol:** Async middleware pipeline — `use()` bilan middleware qo'shish, `execute()` bilan ishga tushirish. Onion model:

```typescript
// pipeline.use(loggingMiddleware).use(authMiddleware).use(handler);
// await pipeline.execute(ctx);
```

<details>
<summary>Yechim</summary>

```typescript
interface Context {
  request: { path: string; method: string; headers: Record<string, string> };
  response: { status: number; body?: unknown };
  state: Record<string, unknown>;
}

type NextFn = () => Promise<void>;
type Middleware = (ctx: Context, next: NextFn) => Promise<void>;

class Pipeline {
  private middlewares: Middleware[] = [];

  use(mw: Middleware): this {
    this.middlewares.push(mw);
    return this;
  }

  async execute(ctx: Context): Promise<Context> {
    let index = 0;
    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        await this.middlewares[index++](ctx, next);
      }
    };
    await next();
    return ctx;
  }
}

// Middleware lar
const logger: Middleware = async (ctx, next) => {
  const start = Date.now();
  console.log(`-> ${ctx.request.method} ${ctx.request.path}`);
  await next();
  console.log(`<- ${ctx.response.status} (${Date.now() - start}ms)`);
};

const auth: Middleware = async (ctx, next) => {
  if (!ctx.request.headers["authorization"]) {
    ctx.response = { status: 401, body: { error: "Unauthorized" } };
    return; // next() chaqirilmaydi — zanjir to'xtaydi
  }
  ctx.state["userId"] = "user-from-token";
  await next();
};

const handler: Middleware = async (ctx) => {
  ctx.response = { status: 200, body: { userId: ctx.state["userId"] } };
};

await new Pipeline().use(logger).use(auth).use(handler).execute({
  request: { path: "/api", method: "GET", headers: { authorization: "Bearer x" } },
  response: { status: 200 }, state: {},
});
```

**Onion model:** `next()` dan oldin "kirish", keyin "chiqish" (logger→auth→handler→auth→logger). Har middleware `next()` chaqirish yoki to'xtatishni o'zi hal qiladi.

</details>

---

## Xulosa

- Factory — generic + type map bilan return type auto-inferred
- Strategy — runtime da algoritm almashtirishOpen/Closed Principle
- Adapter — legacy API ni yangi interface ga moslashtirish
- Result\<T,E\> — type-safe error handling, exception o'rniga discriminated union
- EventEmitter — `Events[K]` tuple bilan har event argument type-safe
- Repository\<T\> — `Omit` bilan create/update, `Partial` bilan filter
- Builder — phantom types bilan required field compile-time check
- Middleware Pipeline — onion model, async chain of responsibility
- Singleton — batafsil [interview/10](10-classes.md)
- State Machine — batafsil [interview/05](05-unions-intersections.md)

[Asosiy bo'limga qaytish →](../21-design-patterns.md)
