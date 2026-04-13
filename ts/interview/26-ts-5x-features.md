# Interview: TypeScript 5.x Yangiliklari

> `using`/`await using` (resource management), inferred type predicates, iterator helpers va TS 5.x da qo'shilgan muhim feature lar bo'yicha interview savollari. TC39 decorators — [interview/19](19-decorators.md), const type params — [interview/08](08-generics.md), satisfies — [interview/06](06-type-narrowing.md), NoInfer — [interview/15](15-utility-types.md), isolatedDeclarations — [interview/18](18-declaration-files.md), erasableSyntaxOnly — [interview/22](22-tsconfig.md).

---

## Nazariy savollar

### 1. `using` va `await using` declarations nima?

<details>
<summary>Javob</summary>

`using` (TS 5.2+) — TC39 **Explicit Resource Management**. Resource scope tugaganda **avtomatik** dispose — `Symbol.dispose` / `Symbol.asyncDispose` chaqiriladi:

```typescript
class TempFile implements Disposable {
  #path: string;
  constructor(path: string) {
    this.#path = path;
    console.log(`Ochildi: ${path}`);
  }
  read(): string { return "data"; }
  [Symbol.dispose](): void {
    console.log(`Yopildi: ${this.#path}`);
  }
}

function processData() {
  using tmp = new TempFile("/tmp/data.tmp");
  const data = tmp.read();
  // Scope tugaganda [Symbol.dispose]() AVTOMATIK chaqiriladi
  // Hatto error bo'lsa ham!
}

// Async
async function getUsers() {
  await using db = await DBConnection.connect("postgres://...");
  return await db.query("SELECT * FROM users");
  // Avtomatik disconnect
}
```

**DisposableStack** — bir nechta resource, teskari tartibda dispose:

```typescript
await using stack = new AsyncDisposableStack();
const db = stack.use(await connectDB());
const cache = stack.use(await connectRedis());
// Scope tugaganda: cache → db (LIFO)
```

C# `using`, Python `with`, Java `try-with-resources` ga o'xshash.

</details>

### 2. Inferred type predicates (TS 5.5+) nima?

<details>
<summary>Javob</summary>

TS 5.5+ da `filter` callback uchun **avtomatik** type predicate hosil bo'ladi — manual type guard kerak emas:

```typescript
const values: (string | null | undefined)[] = ["hello", null, "world", undefined];

// TS 5.4 — manual type predicate kerak
const old = values.filter((v): v is string => v != null);

// TS 5.5+ — avtomatik inferred
const filtered = values.filter(v => v != null);
// type: string[] — TS avtomatik `v is string` hosil qildi!
```

`Array.find`, `Array.some` kabi method larda ham ishlaydi. Murakkab holatlarda manual type guard hali kerak.

</details>

### 3. TS 5.x da qo'shilgan eng muhim feature lar xronologiyasi?

<details>
<summary>Javob</summary>

| Versiya | Feature | Kategoriya |
|---------|---------|-----------|
| **4.9** | `satisfies` operator | Type narrowing |
| **5.0** | TC39 decorators, `const` type params | Decorators, Generics |
| **5.0** | `moduleResolution: "bundler"` | Modules |
| **5.0** | `verbatimModuleSyntax` | Modules |
| **5.2** | `using` / `await using` declarations | Resource management |
| **5.2** | `Symbol.metadata` (decorator metadata) | Decorators |
| **5.4** | `NoInfer<T>` utility type | Generics |
| **5.4** | Preserved narrowing in closures | Narrowing |
| **5.5** | Inferred type predicates | Narrowing |
| **5.5** | `isolatedDeclarations` | Build |
| **5.8** | `erasableSyntaxOnly` | Node.js compat |

**Eng impactful:** `satisfies` (har loyihada), `using` (resource management paradigm), `verbatimModuleSyntax` (bundler compat), TC39 decorators (framework migration).

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `using` — xatoni toping (Daraja: Middle+)

**Savol:** Bu kod nima uchun compile bo'lmaydi?

```typescript
class Connection {
  #url: string;
  constructor(url: string) { this.#url = url; }
  close() { console.log(`Closed: ${this.#url}`); }
}

function processData() {
  using conn = new Connection("postgres://...");
}
```

<details>
<summary>Yechim</summary>

**Xato:** `Connection` `Disposable` interface implement **qilmaydi** — `Symbol.dispose` yo'q. `using` faqat `Disposable` bilan ishlaydi:

```typescript
class Connection implements Disposable {
  #url: string;
  constructor(url: string) { this.#url = url; }

  [Symbol.dispose](): void {
    console.log(`Closed: ${this.#url}`);
  }
}
```

`close()` va `Symbol.dispose` **bir xil emas**. `using` faqat `Symbol.dispose` ni taniydi.

</details>

### 2. Inferred type predicates — output (Daraja: Middle)

**Savol:** TS 5.5+ da output va `filtered` type ini ayting:

```typescript
const values: (string | null | undefined)[] = [
  "hello", null, "world", undefined, "", "ts"
];

const filtered = values.filter(v => v != null);
console.log(filtered);
console.log(filtered.length);
```

<details>
<summary>Yechim</summary>

```
[ 'hello', 'world', '', 'ts' ]
4
```

`filtered` type: `string[]`. TS 5.5+ avtomatik `v is string` infer qiladi. `v != null` (loose equality) `null` va `undefined` ikkalasini olib tashlaydi. Bo'sh string `""` falsy, lekin `!= null` → **truthy** — qoladi.

</details>

### 3. `using` bilan resource management implement (Daraja: Middle+)

**Savol:** `DatabasePool` yozing — `using` bilan avtomatik release:

```typescript
// await using pool = await DatabasePool.create(5);
// const conn = await pool.acquire();
// Scope tugaganda barcha connection lar AVTOMATIK release
```

<details>
<summary>Yechim</summary>

```typescript
class PooledConnection implements Disposable {
  constructor(
    private id: number,
    private onRelease: (id: number) => void
  ) {}

  query(sql: string): string { return `[Conn ${this.id}] ${sql}`; }

  [Symbol.dispose](): void {
    this.onRelease(this.id);
    console.log(`Connection ${this.id} released`);
  }
}

class DatabasePool implements AsyncDisposable {
  private connections: number[] = [];

  private constructor(private size: number) {
    for (let i = 0; i < size; i++) this.connections.push(i);
  }

  static async create(size: number): Promise<DatabasePool> {
    console.log(`Pool created with ${size} connections`);
    return new DatabasePool(size);
  }

  acquire(): PooledConnection {
    const id = this.connections.pop();
    if (id === undefined) throw new Error("No available connections");
    return new PooledConnection(id, (id) => this.connections.push(id));
  }

  async [Symbol.asyncDispose](): Promise<void> {
    console.log(`Pool destroyed, ${this.connections.length} connections released`);
  }
}

// Ishlatish
async function main() {
  await using pool = await DatabasePool.create(3);

  {
    using conn = pool.acquire();
    console.log(conn.query("SELECT * FROM users"));
    // Scope tugaganda conn avtomatik release
  }

  // Pool scope tugaganda barcha connection lar destroy
}
```

**Kalit:** `PooledConnection` `Disposable` (sync), `DatabasePool` `AsyncDisposable`. Scope ending → avtomatik cleanup, error bo'lsa ham.

</details>

### 4. TS 5.x features quiz (Daraja: Middle+)

**Savol:** Har bir feature qaysi versiyada qo'shilgan?

```
A. satisfies operator           → TS ?.?
B. TC39 decorators              → TS ?.?
C. using / await using          → TS ?.?
D. NoInfer<T>                   → TS ?.?
E. Inferred type predicates     → TS ?.?
F. erasableSyntaxOnly           → TS ?.?
```

<details>
<summary>Yechim</summary>

```
A. satisfies          → TS 4.9
B. TC39 decorators    → TS 5.0
C. using / await using → TS 5.2
D. NoInfer<T>         → TS 5.4
E. Inferred type pred → TS 5.5
F. erasableSyntaxOnly → TS 5.8
```

</details>

---

## Xulosa

- `using` / `await using` — avtomatik resource cleanup, `Symbol.dispose`
- Inferred type predicates (5.5+) — `filter(v => v != null)` avtomatik type narrowing
- `satisfies` (4.9) — validate + literal type saqlash
- `const` type params (5.0) — literal inference generic da
- `NoInfer<T>` (5.4) — noto'g'ri inference oldini olish
- `isolatedDeclarations` (5.5) — parallel `.d.ts` emit
- `erasableSyntaxOnly` (5.8) — Node.js strip-types compat
- TC39 decorators (5.0) — JS standart, legacy o'rniga

[Asosiy bo'limga qaytish →](../26-ts-5x-features.md)
