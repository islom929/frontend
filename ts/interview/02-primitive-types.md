# Interview: Primitive Types va Type Annotations

> Type annotations, type inference, primitive types, any/unknown/never/void, literal types, as const, type assertions, widening/narrowing bo'yicha interview savollari.

---

## Mundarija

**Nazariy savollar:**
1. Type annotation va type inference farqi `[Junior+]`
2. Primitive types va `string` vs `String` `[Junior+]`
3. `null` va `undefined` + `strictNullChecks` `[Junior+]`
4. `any` vs `unknown` `[Junior+]`
5. `never` va `void` farqi `[Middle]`
6. Literal types `[Middle]`
7. `as const` va `as` type assertion `[Middle]`
8. Type assertion va type guard farqi `[Middle]`
9. Non-null assertion (`!`) — qachon xavfli `[Middle]`
10. Double assertion (`as unknown as T`) `[Middle+]`
11. Widening va narrowing `[Middle]`
12. Type inference algoritmi va contextual typing `[Senior]`

**Output prediction:**
13. `as const` va widening type quiz `[Junior+]`
14. Type inference quiz — array va `null` `[Middle]`

**Coding challenges:**
15. `as const` bilan type-safe configuration `[Middle+]`
16. User-defined type guard + `unknown` validation `[Middle+]`
17. Discriminated union va exhaustive check `[Middle+]`

**Bug fix:**
18. `fetch` kodida async xatolar `[Middle]`
19. Type assertion abuse — JSON parse `[Middle]`

---

## Nazariy savollar

### Savol 1: Type annotation va type inference farqi nima? Qachon annotation yozish kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Type annotation** — developer tipni `:` bilan aniq belgilaydi (`let name: string = "Ali"`). **Type inference** — TypeScript tipni kontekstdan avtomatik aniqlaydi (`let name = "Ali"` → `string`). Annotation funksiya parametrlari, delayed initialization, public API return type uchun zarur; initialized o'zgaruvchilar va callback parametrlar uchun shart emas.

### To'liq tushuntirish

Annotation yozish **zarur** bo'lgan holatlar:

1. **Funksiya parametrlari** — TS parametr type'ini default da inference qila olmaydi. `noImplicitAny: true` bilan annotation majburiy
2. **Public API return type** — library yoki module export qiladigan funksiyalar uchun aniq yozish best practice
3. **Delayed initialization** — `let userId: string;` keyinroq assign qilinadi
4. **Inference noto'g'ri natija berganda** — `let arr = []` → `any[]` (evolving array)
5. **Discriminated union widening** — `const status: "ok" | "error" = "ok"` aks holda `string`

Annotation **shart emas**:

1. **Initialized o'zgaruvchi** — `let x = 5` → `number`
2. **Callback parametrlar** — `names.map(name => ...)` — kontekstdan
3. **Return type** — funksiya tanasidan inference (public API'dan tashqari)
4. **Object literal property** — har property type aniq

### Kod misol

```typescript
// ✅ Annotation KERAK — funksiya parametrlari
function calculateTotal(price: number, quantity: number): number {
  return price * quantity;
}

// ✅ Annotation SHART EMAS — inference ishlaydi
let count = 10;                  // number
const userName = "Ali";          // "Ali" (literal — const)
let userName2 = "Ali";           // string (widened — let)
const items = ["a", "b", "c"];   // string[]

// ✅ Annotation KERAK — delayed initialization
let userId: string;
userId = fetchUserId();

// ⚠️ Inference noto'g'ri — annotation kerak
let arr = [];            // any[] (evolving array)
let arr2: number[] = []; // number[] — aniq
```

### Edge Cases

- **Evolving array** (`let arr = []`) — TS dastlab `any[]`, keyingi `push`/element assign'larga qarab evolve qiladi. Agar variable scope ichida bir nechta assign bo'lsa va final type aniqlanmasa, `noImplicitAny: true` "Variable 'arr' implicitly has type 'any[]' in some locations" xatosini berishi mumkin.
- **Evolving any** (`let x = null`) — dastlab `any`, assign'larga qarab evolve. Faqat `let` + initial `null`/`undefined` da.
- **Contextual typing** — `[1,2,3].map(x => x.toString())` — `x: number` kontekstdan, standalone `(x) => x.toString()` esa `noImplicitAny`'da xato (parameter type aniqlanmaydi).
- **Return type inference** — recursive funksiya'da TS implicit `any` deb hisoblashi va xato berishi mumkin, aniq return type kerak.

### Follow-up savollar

1. **"`noImplicitAny: true` bilan har parametrga annotation kerakmi?"** — Ha, agar inference kontekstdan kelmasa. Callback parametrlari odatda inference olinadi.
2. **"Public API uchun nima uchun aniq return type kerak?"** — Inference kelajakda kod o'zgarsa, return type avtomatik o'zgaradi va consumer kod buzilishi mumkin. Aniq annotation `tsc` ni majburlaydi.

</details>

---

### Savol 2: Primitive types qaysilar va `string` vs `String` farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript da 7 ta primitive type: `string`, `number`, `boolean`, `bigint`, `symbol`, `null`, `undefined`. `string` (kichik harf) — JavaScript primitive type. `String` (katta harf) — wrapper object class. Annotation da har doim kichik harfli versiya ishlatish kerak.

### To'liq tushuntirish

7 ta primitive type:

| Type | Qiymat misol | Wrapper class (TAQIQ) |
|------|--------------|-----------------------|
| `string` | `"hello"` | `String` |
| `number` | `42`, `3.14`, `NaN`, `Infinity` | `Number` |
| `boolean` | `true`, `false` | `Boolean` |
| `bigint` | `9007199254740992n` | `BigInt` |
| `symbol` | `Symbol("id")` | `Symbol` |
| `null` | `null` | — |
| `undefined` | `undefined` | — |

`string` vs `String` farqi:

- `string` — primitive type, `typeof "x" === "string"`
- `String` — wrapper object class. `new String("x")` yaratadi (`typeof === "object"`)
- Wrapper class'lar (`String`, `Number`, `Boolean`) ko'p hollarda kerak emas. JavaScript auto-boxing qiladi (`"hello".toUpperCase()` da `"hello"` vaqtinchalik `String` object'ga o'raladi)

### Kod misol

```typescript
// ✅ Primitive type — annotation'da kichik harf
let name: string = "Ali";
let age: number = 25;
let active: boolean = true;

// ❌ Wrapper class — annotation'da ishlatmang
let badName: String = "Ali"; // ⚠️ Eslatma yo'q, lekin best practice emas

// Farq runtime da
const primitive = "hello";
const wrapper = new String("hello");

console.log(typeof primitive); // → "string"
console.log(typeof wrapper);   // → "object"
console.log(primitive === "hello"); // → true
console.log(wrapper === "hello");   // → false (object reference)
```

NaN va Infinity — `number` type:

```typescript
const a: number = NaN;
const b: number = Infinity;
const c: number = -Infinity;

console.log(NaN === NaN); // → false (JavaScript quirk)
console.log(Number.isNaN(NaN)); // → true (xavfsiz check)
```

### Edge Cases

- `Number.MAX_SAFE_INTEGER` — `2^53 - 1`. Bundan katta integer uchun `bigint` kerak
- `0.1 + 0.2 !== 0.3` — IEEE 754 floating point cheklov
- `String("x")` (without `new`) — primitive `"x"` qaytaradi (type conversion)
- TypeScript `Object` type (katta harf) — `null`/`undefined` dan tashqari hamma qiymatga mos. Annotation da kerak emas

### Follow-up savollar

1. **"`bigint` qachon kerak?"** — `Number.MAX_SAFE_INTEGER` (2^53-1) dan katta integer kerak bo'lganda (financial calculation, blockchain, ID'lar). Sintaksis: `42n` yoki `BigInt(42)`. `bigint` va `number` orasida implicit conversion yo'q.
2. **"`symbol` qachon kerak?"** — Unique object key yaratish uchun (`Symbol("id")`). Library da private/internal property uchun foydali. ES iterator protocol (`Symbol.iterator`) va metaprogramming uchun.

</details>

---

### Savol 3: `null` va `undefined` farqi nima va `strictNullChecks` qanday ta'sir qiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`null` — qiymat "atayin yo'q" (explicit). `undefined` — qiymat "aniqlanmagan" (variable assign qilinmagan, function return qilmagan, property mavjud emas). `strictNullChecks: true` bilan `null` va `undefined` har tipdan ajratiladi — `string | null` deb aniq yozish kerak.

### To'liq tushuntirish

JavaScript semantic farqi:

- **`undefined`** — "qiymat hali aniqlanmagan". Default qiymat: assignsiz variable, return qilmagan function, mavjud emas property
- **`null`** — "qiymat yo'q, atayin". Developer aniq beradi: "bu qiymat hali yo'q, lekin keyinroq bo'lishi mumkin"

Tarixiy farq: `null` JavaScript da object reference uchun "yo'qlik" (`document.getElementById` topmasa `null` qaytaradi), `undefined` esa primitive "aniqlanmagan".

`typeof` xulqi:

```typescript
typeof undefined; // → "undefined"
typeof null;      // → "object" (JS legacy bug)
```

**`strictNullChecks: false` (eski xulq)** — har tip `null`/`undefined` ni qabul qiladi:

```typescript
let name: string = null;      // ✅ ruxsat (xavfli)
let age: number = undefined;  // ✅ ruxsat
```

**`strictNullChecks: true` (zamonaviy)** — `null`/`undefined` alohida type, aniq yozish kerak:

```typescript
let name: string = null;            // ❌ Type 'null' is not assignable to 'string'
let name2: string | null = null;    // ✅ Aniq belgilangan
let age: number | undefined = undefined; // ✅
```

### Kod misol

`strictNullChecks` bilan narrowing:

```typescript
function getLength(str: string | null): number {
  // str.length; ❌ 'str' is possibly 'null'

  if (str !== null) {
    return str.length; // ✅ narrowing'dan keyin xavfsiz
  }
  return 0;
}

// Optional chaining bilan soddaroq
function getLengthOpt(str: string | null): number {
  return str?.length ?? 0;
}

// Nullish coalescing
const username = userInput ?? "anonymous";
```

DOM API real-world misol:

```typescript
const input = document.getElementById("name");
// input: HTMLElement | null

input.value; // ❌ 'input' is possibly 'null'

if (input !== null && input instanceof HTMLInputElement) {
  console.log(input.value); // ✅
}
```

### Edge Cases

- `void` parameter type — `function callback(): void` — argument `undefined` qaytarmasligi mumkin (TS ignore qiladi)
- `null` JSON da bor (`JSON.stringify(null) === "null"`), `undefined` JSON da yo'q (`JSON.stringify({x: undefined}) === "{}"`)
- `==` (loose equality) — `null == undefined` `true`, `null === undefined` `false`
- `strictNullChecks: false` da `tsc` "intentional unsoundness" — type system soundness yo'q

### Follow-up savollar

1. **"Optional property (`?`) va `undefined` farqi bormi?"** — `exactOptionalPropertyTypes: true` bilan farqlanadi. `{ name?: string }` — property bo'lmasligi mumkin. `{ name: string | undefined }` — property bo'lishi shart, lekin qiymat `undefined` bo'lishi mumkin.
2. **"`null` o'rniga doim `undefined` ishlatish mumkinmi?"** — Ko'p loyihalarda ha — bitta nullish qiymat oson. Lekin DOM API, JSON, va ko'p library'lar `null` qaytaradi — convention ga moslashish kerak.

</details>

---

### Savol 4: `any` va `unknown` farqi nima? Nima uchun `unknown` afzal? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`any` type checking ni butunlay o'chiradi — har qanday operatsiya ruxsat etiladi. `unknown` — "tip noma'lum, avval tekshir" — narrowing siz hech qanday operatsiya mumkin emas. Yangi kodda doim `unknown` ishlatish kerak.

### To'liq tushuntirish

| Xususiyat | `any` | `unknown` |
|-----------|-------|-----------|
| Boshqa tipga assign | ✅ Ruxsat | ❌ Faqat `any`/`unknown` ga |
| Property/method call | ✅ Tekshirilmaydi | ❌ Narrowing kerak |
| Type safety | ❌ Yo'q | ✅ Bor |
| Asosiy use case | JS → TS migration | API boundary, `catch(e)`, tashqi data |

`any` "viral" — `any` ga assign qilingan qiymat ham `any` bo'ladi. `unknown` "non-viral" — narrowing'dan o'tmasa, operatsiya mumkin emas.

`unknown` TS 3.0 da kiritilgan. Type theory tilida "top type" — barcha tiplar `unknown` ga assignable, lekin `unknown` faqat `any` va `unknown` ga.

### Kod misol

`any` — xavfli:

```typescript
function processAny(data: any) {
  data.user.profile.getName(); // ✅ Compile da OK
  return data.toUpperCase();   // ✅ Compile da OK, lekin runtime da TypeError
}

processAny(null);    // 💥 Runtime crash
processAny(42);      // 💥 toUpperCase yo'q
```

`unknown` — xavfsiz:

```typescript
function processUnknown(data: unknown) {
  // data.name; ❌ 'data' is of type 'unknown'

  if (typeof data === "object" && data !== null && "name" in data) {
    console.log(data.name); // ✅ narrowing'dan keyin
  }
}

// catch blokda
try {
  await riskyOperation();
} catch (error) {
  // error: unknown (useUnknownInCatchVariables: true)
  if (error instanceof Error) {
    console.log(error.message);
  }
}
```

### Edge Cases

- `any` `never` dan tashqari har tipga assignable
- `unknown | string` → `unknown` (`unknown` boshqa tiplarni "yutadi")
- TypeScript ning ba'zi API'lari `any` qaytaradi (`JSON.parse`, `fetch().then(r => r.json())`) — manual `unknown` cast tavsiya
- `Function` type ham `any` ga o'xshash xavfli — aniq function signature yozish kerak

### Follow-up savollar

1. **"`unknown` ni narrowing siz qanday qilib `any` ga aylantirish mumkin?"** — `as any` cast bilan. Lekin bu type safety ni butunlay yo'qotadi. Yaxshisi: `as SomeType` aniq cast yoki manual narrowing.
2. **"`any` ni qachon ishlatish kerak?"** — Migration paytida vaqtincha, library uchun type yozishda (agar `@types` yo'q bo'lsa), test mock larda. Boshqa joyda `unknown` yoki aniq type.

<details>
<summary><strong>Deep Dive</strong></summary>

`unknown` type theory tilida "top type" — barcha tip unga assignable, lekin `unknown` faqat `any` va `unknown` ga assignable. `any` bir vaqtda top va bottom type xususiyatini birga oladi: `any <: T` va `T <: any` har `T` uchun bajariladi (faqat `never` istisno — `any` `never` ga assignable emas). Aynan shu `T <: any` yo'nalishi type system soundness'ini buzadi — `unknown` bu yo'nalishni yopib, "avval narrow qil" majburiyatini kiritadi.

`useUnknownInCatchVariables` (TS 4.4) sababi — JavaScript da `throw "string"` yoki `throw 42` qonuniy, shuning uchun `e: any` semantically noto'g'ri edi. `unknown` to'g'riroq: har `throw` qiymati `unknown` deb qaraladi.

`unknown`'ni narrowing — `typeof`, `instanceof`, `in` operator, user-defined type predicate orqali. Checker control flow graph'ni binder bosqichida quradi, keyin reference nuqtasida `getFlowTypeOfReference` orqali lazily hisoblaydi — har branch'dan keyin qiymat tipini torroq variantga qisqartiradi.

</details>

</details>

---

### Savol 5: `never` va `void` farqi nima? `never` qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`void` — funksiya normal tugaydi, qiymat qaytarmaydi (aslida `undefined`). `never` — funksiya **hech qachon** normal tugamaydi — yoki `throw`, yoki cheksiz loop, yoki impossible type. `never` exhaustive checking, error type, va impossible state'lar uchun ishlatiladi.

### To'liq tushuntirish

`void` semantic:
- Function return qiymati ahamiyatsiz
- Caller return qiymatni ishlatmasligi shart
- Aslida `undefined` qaytariladi

`never` semantic:
- Function hech qachon return qilmaydi (throw yoki infinite loop)
- Type system da "bottom type" — boshqa har tipga assignable
- Impossible state ifoda qiladi

| Xususiyat | `void` | `never` |
|-----------|--------|---------|
| Return qiladi | Normal (`undefined`) | Hech qachon |
| Boshqa tipga assign | `void` ga `any`/`undefined` mumkin | `never` har tipga assignable |
| Use case | Side effect funksiya | Exhaustive check, error, impossible |

`never` ning use case'lari:

1. **Throw funksiya** — `function fail(msg: string): never { throw new Error(msg); }`
2. **Infinite loop** — `function loop(): never { while (true) {} }`
3. **Exhaustive check** — discriminated union'da barcha holatlar qamrab olinganini tekshirish
4. **Conditional type'da impossible branch** — `T extends ... ? X : never`
5. **Filter / type subtraction** — `Exclude<T, U>` `never` orqali

### Kod misol

`void` va `never`:

```typescript
// void — normal return, qiymat yo'q
function log(message: string): void {
  console.log(message);
}

// never — hech qachon return qilmaydi
function throwError(msg: string): never {
  throw new Error(msg);
}

function infiniteLoop(): never {
  while (true) {
    process();
  }
}
```

Exhaustive check — eng muhim `never` use case:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.size ** 2;
    case "triangle": return 0.5 * shape.base * shape.height;
    default:
      // Agar yangi Shape qo'shilsa va case yozilmasa — compile error
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}
```

Conditional type'da `never`:

```typescript
type NonNull<T> = T extends null | undefined ? never : T;

type A = NonNull<string | null>;  // string
type B = NonNull<number | undefined>; // number
```

### Edge Cases

- Funksiya ba'zan throw, ba'zan return qilsa — return type `never` emas:
  ```typescript
  function divide(a: number, b: number): number {
    if (b === 0) throw new Error("Division by zero");
    return a / b; // Return type: number, never emas
  }
  ```
- `Promise<void>` va `Promise<undefined>` farq qiladi — `void` da TS caller return ni ishlatmasligi kerak
- Array `[]` ning type — `never[]` agar element bo'lmasa (TS strict mode)
- `never` `union` da yutadi: `string | never` → `string`. `intersection` da hammasini yutadi: `string & never` → `never`

### Follow-up savollar

1. **"Function `void` return qilsa, undefined qaytarish shartmi?"** — Yo'q. `void` return type bo'lgan funksiyada `return` yozmaslik mumkin yoki `return;` bo'sh. JavaScript avtomatik `undefined` qaytaradi.
2. **"`never` ni qanday qilib `void` ga aylantirish mumkin?"** — `never` har tipga assignable, shuning uchun cast kerak emas: `function f(): void { return throwError("x"); }` — `never` `void` ga assign bo'ladi.

</details>

---

### Savol 6: Literal types nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Literal type — **aniq bitta qiymat** tipga aylanadi (`"hello"`, `42`, `true`). `string`/`number`/`boolean` ning torroq versiyasi. Asosan **union** bilan ishlatiladi — cheklangan qiymatlar to'plami yaratish uchun (enum alternativasi).

### To'liq tushuntirish

Literal type uch xil:

1. **String literal** — `"active"`, `"GET"`, `"north"`
2. **Number literal** — `0`, `1`, `200`, `404`
3. **Boolean literal** — `true`, `false`

Use case'lar:

1. **Discriminated union** — switch da exhaustive check
2. **Enum alternativasi** — `type Status = "active" | "inactive"`
3. **Configuration** — `{ mode: "production" | "development" }`
4. **Type-safe routing** — `type Route = "/" | "/users" | "/posts"`
5. **HTTP method** — `type Method = "GET" | "POST" | "PUT" | "DELETE"`

**Widening qoidalari:**

- `const x = "hello"` → `"hello"` (literal — const reassignable emas)
- `let x = "hello"` → `string` (widened — let reassignable)
- `let x = "hello" as const` → `"hello"` (literal saqlanadi, lekin `let` qoldigi uchun qayta assign mumkin — faqat aynan `"hello"` qiymatga)

### Kod misol

Discriminated union:

```typescript
type Result =
  | { ok: true; data: string }
  | { ok: false; error: string };

function handle(result: Result): string {
  if (result.ok) {
    return result.data;   // TS biladi: data mavjud
  } else {
    return result.error;  // TS biladi: error mavjud
  }
}
```

Enum alternativasi:

```typescript
// Eski uslub — enum
enum Status { Active, Inactive, Pending }

// Zamonaviy — literal union
type Status = "active" | "inactive" | "pending";

function setStatus(s: Status): void {
  // s faqat "active"/"inactive"/"pending"
}

setStatus("active");  // ✅
// setStatus("deleted"); ❌
```

Number literal — HTTP status:

```typescript
type SuccessCode = 200 | 201 | 204;
type ErrorCode = 400 | 401 | 403 | 404 | 500;
type HttpCode = SuccessCode | ErrorCode;

function handleStatus(code: HttpCode): string {
  if (code === 200) return "OK";
  if (code === 404) return "Not Found";
  return "Other";
}
```

Widening:

```typescript
const a = "hello";              // "hello" (literal)
let b = "hello";                // string (widened)
let c = "hello" as const;       // "hello" (literal saqlanadi, faqat "hello" ga qayta assign mumkin)

const obj1 = { x: 10, y: "hi" };
// type: { x: number; y: string } — property'lar widened

const obj2 = { x: 10, y: "hi" } as const;
// type: { readonly x: 10; readonly y: "hi" }
```

### Edge Cases

- **Template literal type** (TS 4.1+) — `` type Greeting = `Hello, ${string}` ``
- **Numeric literal range** — TS literal'lar ranges qo'llab-quvvatlamaydi (`1..10` yo'q). Union bilan `1 | 2 | 3 | ...` yozish kerak
- **Boolean literal** kamdan-kam useful — odatda `boolean` yetadi. `true` literal flag pattern uchun foydali
- **`as const` array** — element'lar literal va `readonly`: `["a", "b"] as const` → `readonly ["a", "b"]`

### Follow-up savollar

1. **"Literal type union vs enum — qaysi biri afzal?"** — Literal union. Bundle size 0, tree-shaking yaxshi, `isolatedModules`/`erasableSyntaxOnly` muammosiz. Enum faqat legacy yoki bit flags uchun.
2. **"String literal'ni dynamic generate qilish mumkinmi?"** — Template literal types (TS 4.1+) bilan: `type EventName<T extends string> = ${T}Changed`.

</details>

---

### Savol 7: `as const` qanday ishlaydi va `as` type assertion'dan farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`as const` — expression'ning barcha qiymatlarini **literal type** va **readonly** qiladi (widening to'xtatadi). `as Type` — type assertion, compiler'ga "men bilaman, bu shu tip" deyish. `as const` xavfsiz (faqat narrowing), `as Type` xavfli (runtime tekshirilmaydi).

### To'liq tushuntirish

`as const` (TS 3.4+) ta'sirlari:

1. **String/Number/Boolean literal** — widening to'xtatiladi (`"hello"` → `"hello"`, emas `string`)
2. **Object property** — barcha property `readonly`
3. **Array** — `readonly` tuple, element'lar literal
4. **Nested struktura** — to'liq frozen (recursive readonly + literal)

`as Type` (type assertion):
- Compile-time `as` runtime da o'chiriladi (type erasure)
- Compiler tekshiradimi? — faqat aniq nomuvofiqlik (asl tip va target tip umuman mos kelmasa)
- Aslida noto'g'ri bo'lsa — runtime da crash

### Kod misol

`as const` ishlatish:

```typescript
// as const bilan literal inference
const config1 = {
  url: "/api",
  retries: 3,
} as const;
// type: { readonly url: "/api"; readonly retries: 3 }

// as const bilan array → tuple
const ROLES = ["admin", "user", "guest"] as const;
// type: readonly ["admin", "user", "guest"]

type Role = typeof ROLES[number];
// "admin" | "user" | "guest"

// as const bilan enum alternativasi
const HttpMethod = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
} as const;

type HttpMethod = typeof HttpMethod[keyof typeof HttpMethod];
// "GET" | "POST" | "PUT" | "DELETE"
```

`as Type` ishlatish:

```typescript
// ✅ DOM query — type assertion (developer biladi)
const input = document.getElementById("name") as HTMLInputElement;
console.log(input.value);
// Lekin element yo'q bo'lsa — runtime crash

// ✅ Xavfsizroq variant — instanceof check
const el = document.getElementById("name");
if (el instanceof HTMLInputElement) {
  console.log(el.value);
}

// ❌ Xavfli — aslida noto'g'ri
const obj = { x: 1 } as { x: number; y: number };
console.log(obj.y); // → undefined, runtime da crash bo'lishi mumkin
```

### Edge Cases

- `as const` faqat literal qiymatlar bilan ishlaydi — `let x = something() as const` xato beradi (function call literal emas)
- `as` ikki tip mos kelmasa xato beradi (`"hello" as number` ❌) — `as unknown as` bilan o'tkazib yuborish mumkin (double assertion)
- `as const` bilan `satisfies` birga: `{ ... } as const satisfies T` — literal saqlash + type validation
- `as` faqat compile-time — runtime da hech qanday tekshirish yo'q

### Follow-up savollar

1. **"`as const` array `readonly` bo'lganda `.map()` ishlaydimi?"** — Ha. `map`, `filter`, `slice` — read-only method'lar (yangi array qaytaradi). `push`, `pop`, `sort` (mutate) — yo'q.
2. **"`as` bilan tip yo'qotmasdan widening qilish mumkinmi?"** — Ha, `as string`: `const x = "hello" as string` → `string`. Lekin bu inverse use case — odatda `as const` (narrow) ishlatiladi.

</details>

---

### Savol 8: Type assertion va type guard farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Type assertion** (`as`, `<>`) — developer compiler'ga "men bilaman, bu shu tip" deydi. Runtime da hech narsa o'zgarmaydi. Noto'g'ri bo'lsa — runtime crash. **Type guard** — runtime tekshiruv (`typeof`, `instanceof`, `in`, user-defined predicate) orqali compiler'ga tipni isbotlaydi. Xavfsiz.

### To'liq tushuntirish

| Xususiyat | Type Assertion | Type Guard |
|-----------|----------------|------------|
| Runtime tekshiruv | ❌ Yo'q | ✅ Bor |
| Xavfsizlik | ❌ Xavfli | ✅ Xavfsiz |
| Syntax | `value as Type` yoki `<Type>value` | `typeof`/`instanceof`/`in`/`(x: unknown): x is T` |
| Qachon ishlatish | 100% ishonchli bo'lganda | Default holat |

Type guard turlari:

1. **`typeof`** — primitive type'lar uchun: `typeof x === "string"`
2. **`instanceof`** — class instance: `x instanceof Error`
3. **`in` operator** — property existence: `"name" in obj`
4. **Discriminated union** — literal tag: `if (shape.kind === "circle")`
5. **User-defined type guard** — `function isUser(x: unknown): x is User`
6. **Assertion function** — `function assert(condition): asserts condition`

### Kod misol

Type assertion vs type guard:

```typescript
// ❌ Type assertion — xavfli
const el1 = document.getElementById("name") as HTMLInputElement;
console.log(el1.value); // Element yo'q bo'lsa → crash

// ✅ Type guard — xavfsiz
const el2 = document.getElementById("name");
if (el2 instanceof HTMLInputElement) {
  console.log(el2.value); // ✅ runtime tekshirilgan
}
```

User-defined type guard:

```typescript
interface User {
  name: string;
  email: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" && value !== null &&
    "name" in value && typeof (value as Record<string, unknown>).name === "string" &&
    "email" in value && typeof (value as Record<string, unknown>).email === "string"
  );
}

const data: unknown = JSON.parse(json);
if (isUser(data)) {
  console.log(data.name); // ✅ narrowed to User
}
```

Assertion function (TS 3.7+):

```typescript
function assertIsNumber(value: unknown): asserts value is number {
  if (typeof value !== "number") {
    throw new Error("Not a number");
  }
}

function double(x: unknown): number {
  assertIsNumber(x);
  return x * 2; // ✅ x: number (after assertion)
}
```

### Edge Cases

- Type assertion ikki tip butunlay mos kelmasa xato beradi: `"hello" as number` ❌
- Double assertion (`as unknown as T`) — bu cheklovni o'tkazadi, lekin yanada xavfli
- `as` runtime da o'chiriladi — performance ta'siri nol
- Type guard narrowing closure ichida saqlanmasligi mumkin (TS 5.4 da yaxshilangan)
- TS 5.5+ — narrowing pattern li funksiya uchun avtomatik type predicate inference

### Follow-up savollar

1. **"`as` operator ni qachon ishlatish maqsadga muvofiq?"** — DOM query (faqat element 100% mavjud bo'lsa), test mock, generic constraint workaround. Default holat — type guard.
2. **"Assertion function va type guard farqi qachon ahamiyatli?"** — Assertion function exception throw qiladi (control flow uzilishi). Type guard `boolean` qaytaradi (caller ikki branch ham handle qilishi mumkin). Assertion function aniqlik beradi (xato bo'lsa to'xtaydi), type guard moslashuvchanroq.

</details>

---

### Savol 9: Non-null assertion (`!`) nima va nima uchun xavfli? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`!` postfix operator — TypeScript ga "bu qiymat `null`/`undefined` emas" deyish. Runtime da hech narsa o'zgarmaydi — type erasure dan keyin yo'qoladi. Qiymat aslida `null` bo'lsa — runtime crash. Faqat 100% ishonchli bo'lganda ishlatish kerak.

### To'liq tushuntirish

`!` operator faqat compile-time:
- TypeScript `null`/`undefined` ni union'dan olib tashlaydi
- Runtime da hech narsa o'zgarmaydi
- Aslida qiymat `null` bo'lsa — keyingi property access da `TypeError`

Use case'lar:

1. **Map / cache** — siz qo'ygan key ekanligini bilasiz
2. **Initialization order** — class field setup pattern
3. **Optional callback** — `onClick!()` agar callback aniq mavjud bo'lsa

Xavfli use case'lar:
- API javob — har doim tekshirilishi kerak
- DOM query — element yo'q bo'lishi mumkin
- Array index — `arr[i]!` agar `noUncheckedIndexedAccess: true`

### Kod misol

`!` xavfli:

```typescript
const el = document.getElementById("app")!;
// el: HTMLElement (not HTMLElement | null)
console.log(el.innerHTML);
// Element yo'q bo'lsa: TypeError: Cannot read property 'innerHTML' of null

// ✅ Xavfsiz alternativa
const elSafe = document.getElementById("app");
if (elSafe !== null) {
  console.log(elSafe.innerHTML); // ✅
}
```

`!` xavfsiz use case — Map dan olish:

```typescript
const cache = new Map<string, number>();
cache.set("count", 42);

// Biz qo'ydik, 100% mavjud
const value = cache.get("count")!;
console.log(value); // 42

// ⚠️ Tashqi key — xavfli
function getValue(key: string): number {
  return cache.get(key)!; // ❌ Key mavjud bo'lmasligi mumkin
}
```

Definite assignment assertion — class field uchun:

```typescript
class UserService {
  // ❌ strictPropertyInitialization xato
  // private user: User;

  // ✅ `!` — keyinroq initialize bo'ladi (init() chaqirilishi shart)
  private user!: User;

  async init(): Promise<void> {
    this.user = await fetchUser();
  }
}
```

### Edge Cases

- `!` `?.` bilan birga: `obj?.prop!` — `obj` `null` bo'lsa `undefined`, lekin TS keyingi access uchun `!` bilan `undefined` ni olib tashlaydi
- `!` array index uchun: `arr[i]!` — `noUncheckedIndexedAccess: true` yoqilganda kerak bo'lishi mumkin
- `!` function call uchun: `callback!()` — callback aniq mavjud bo'lsa
- `!` runtime overhead nol — type erasure dan keyin yo'qoladi

### Follow-up savollar

1. **"`!` o'rniga doim narrowing ishlatish kerakmi?"** — Ko'p hollarda ha. `!` faqat siz 100% ishonchli bo'lgan joylar uchun (Map.get o'zingiz qo'ygan key, dependency injection).
2. **"`strictNullChecks: false` da `!` ishlaydimi?"** — Sintaksis ruxsat etiladi, lekin ta'siri yo'q (`null`/`undefined` allaqachon har tipga assignable). Pratique — `strictNullChecks: true` bilan ishlaydi.

</details>

---

### Savol 10: Double assertion (`as unknown as T`) qachon ishlatiladi va nima uchun xavfli? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Double assertion — `value as unknown as TargetType` pattern. Normal `as` ikki tip butunlay mos kelmasa xato beradi (`"hello" as number` ❌). `unknown` orqali bu cheklov chetlab o'tiladi. Asosan test mock va legacy code migration uchun. Production kodda ishlatish xavfli.

### To'liq tushuntirish

Normal `as` qoidalari:
- Source va target tiplar **ba'zi mosligi bo'lishi shart** (`Subtype` yoki `Supertype` aloqasi)
- Butunlay yot tiplar uchun xato: `"hello" as number` → "Type 'string' is not assignable to type 'number'"

Double assertion qanday ishlaydi:
1. `value as unknown` — har tipni `unknown` ga aylantirish (yuqori cast)
2. `unknown as TargetType` — `unknown` dan istalgan tipga (pastki cast)

Bu compiler check'ni butunlay chetlab o'tadi — runtime da hech qanday tekshirish yo'q.

Use case'lar:

1. **Test mock** — interface'ning faqat ba'zi method'larini implement qilish
2. **Legacy migration** — eski JavaScript API ni TypeScript type'ga sig'dirish
3. **Generic constraint workaround** — TypeScript type system cheklovini chetlab o'tish

### Kod misol

Test mock:

```typescript
interface UserService {
  getUser(id: string): Promise<{ name: string; email: string }>;
  updateUser(id: string, data: Partial<User>): Promise<void>;
  deleteUser(id: string): Promise<void>;
  listUsers(page: number): Promise<User[]>;
}

// Test da faqat getUser kerak, qolganlarini implement qilmaslik:
const mockService = {
  getUser: async (id: string) => ({ name: "Test", email: "t@t.com" }),
} as unknown as UserService;

// ✅ Test ishlaydi, lekin agar updateUser chaqirilsa — runtime crash
```

Production da xavfli:

```typescript
// ❌ Xavfli — runtime crash kafolatlangan
const fake = {} as unknown as { name: string };
console.log(fake.name.toUpperCase()); // 💥 Cannot read property 'toUpperCase' of undefined

// ❌ Eski API ni TypeScript ga "siqish"
const legacyData = getRawData() as unknown as User;
// Aslida getRawData() User shape ga mos kelmasa — runtime issues
```

Yaxshiroq alternativalar:

```typescript
// ✅ Variant 1: Partial + spread (test uchun)
const baseService: UserService = {
  getUser: async () => ({ name: "Test", email: "t@t.com" }),
  updateUser: async () => {},
  deleteUser: async () => {},
  listUsers: async () => [],
};

// ✅ Variant 2: Jest/Vitest with vi.fn()
import { vi } from "vitest";
const mockService: UserService = {
  getUser: vi.fn().mockResolvedValue({ name: "Test", email: "t@t.com" }),
  updateUser: vi.fn(),
  deleteUser: vi.fn(),
  listUsers: vi.fn(),
};
```

### Edge Cases

- Double assertion bilan TypeScript hech qanday tekshirish qilmaydi — `"" as unknown as Date` ham ruxsat
- `unknown` o'rniga `any` ham ishlaydi (`as any as T`), lekin `unknown` aniqroq niyat ko'rsatadi
- Some linter (ESLint) — `@typescript-eslint/no-explicit-any` va `@typescript-eslint/consistent-type-assertions` rule'lari double assertion'ni cheklaydi
- Generic da double assertion — type parameter inference'ni buzishi mumkin

### Follow-up savollar

1. **"Type assertion'ni butunlay taqiqlash kerakmi?"** — Yo'q, faqat ehtiyot bilan. Test, library boundary, DOM query — assertion zaruriy. Lekin har joyda `as` — kod smell.
2. **"`satisfies` bilan double assertion'ni almashtirish mumkinmi?"** — Ba'zan ha. `satisfies` type check qiladi va inference saqlaydi. Lekin `satisfies` faqat aniq type'ga moslikni tekshiradi, double assertion esa har qanday tipga "cast" qiladi.

</details>

---

### Savol 11: Type widening va narrowing nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Widening** — TypeScript literal type'ni kengaytirib base type qiladi (`"hello"` → `string`). `let` declaration, function parameter passing, object property'da avtomatik. **Narrowing** — union type'ni control flow analysis orqali aniq tipga qisqartirish (`string | null` → `string`).

### To'liq tushuntirish

**Widening qoidalari:**

- `const x = "hello"` → `"hello"` (literal saqlanadi, const mutate bo'lmaydi)
- `let x = "hello"` → `string` (widened, let mutate bo'ladi)
- Object property — har property widened: `{ x: 10 }` → `{ x: number }`
- Function argument — literal `"hello"` `string` ga widened bo'lib uzatiladi (agar parameter type literal bo'lmasa)
- `as const` — widening to'xtatadi

**Narrowing texnikalari:**

1. **`typeof` guard** — `typeof x === "string"` → `x: string`
2. **`instanceof`** — `x instanceof Error` → `x: Error`
3. **Equality** — `x === null` → narrowing
4. **Truthiness** — `if (x)` → null/undefined olib tashlanadi
5. **`in` operator** — `"name" in obj` → property mavjud
6. **Discriminated union** — `if (shape.kind === "circle")` → narrow to specific variant
7. **User-defined type guard** — `(x: unknown): x is User`
8. **Assertion function** — `asserts x is T`

### Kod misol

Widening:

```typescript
// Literal vs widened
const a = "hello";        // "hello" (literal)
let b = "hello";          // string (widened)
let c = "hello" as const; // "hello" (no widening)

// Object property widening
const obj = { name: "Ali", age: 25 };
// type: { name: string; age: number }

const objConst = { name: "Ali", age: 25 } as const;
// type: { readonly name: "Ali"; readonly age: 25 }

// Function argument widening
function setStatus(s: "active" | "inactive") { /* ... */ }
const status = "active"; // const: "active" (literal)
setStatus(status); // ✅

let statusLet = "active"; // let: string (widened)
// setStatus(statusLet); // ❌ string not assignable to "active" | "inactive"
```

Narrowing:

```typescript
// typeof narrowing
function process(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // value: string
  }
  return value.toFixed(2);       // value: number
}

// instanceof narrowing
function handleError(error: Error | string): void {
  if (error instanceof Error) {
    console.log(error.message); // error: Error
  } else {
    console.log(error);          // error: string
  }
}

// Discriminated union narrowing
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number };

function area(shape: Shape): number {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2; // shape: { kind: "circle"; radius: number }
  }
  return shape.size ** 2; // shape: { kind: "square"; size: number }
}

// Truthiness narrowing
function getLength(str: string | null): number {
  if (str) {
    return str.length; // str: string
  }
  return 0;
}
```

### Edge Cases

- **Truthiness narrowing** `""`, `0`, `false`, `null`, `undefined`, `NaN` ni hisobga oladi. `string | null` da `if (str)` — `""` ham filter qilinadi (kerak bo'lsa `str !== null` aniqroq)
- **Aliased narrowing** — `const isString = typeof x === "string"` → keyingi `if (isString)` da narrowing ishlaydi (TS 4.4+)
- **Narrowing closure ichida yo'qoladi** — `() => x.length` callback ichida narrowing saqlanmaydi (TS 5.4+ ba'zi pattern'lar yaxshilangan)
- **Type predicate inference** (TS 5.5+) — funksiya tanasidan avtomatik `value is T` predicate hosil bo'lishi mumkin

### Follow-up savollar

1. **"Closure ichida narrowing nima uchun yo'qoladi?"** — Variable closure orqali ulanganda value o'zgarishi mumkin — TS conservative bo'lib type'ni keng saqlaydi. Yechim: local variable ga ko'chirish (`const local = obj.value; if (local) { setTimeout(() => local.length) }`).
2. **"Narrowing va widening bir vaqtda ishlay oladimi?"** — Ha. Funksiya parametri widened (literal `"a"` → `string`), funksiya ichida `typeof === "a"` bilan narrow qilish mumkin (lekin `"a"` literal type bo'lsa parameter literal bo'lishi kerak).

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript narrowing'ni control flow graph orqali amalga oshiradi. Graph binder bosqichida (`binder.ts`) quriladi — har statement uchun flow node hosil bo'ladi. Checker reference nuqtasida `getFlowTypeOfReference` (`checker.ts`) ni chaqiradi: bu funksiya flow graph bo'ylab orqaga (reference'dan declaration tomon) yurib, har `if`/`switch`/`&&`/`||` branch'i qo'ygan refinement'ni qiymat tipiga qo'llaydi.

Discriminated union narrowing — TS literal tag (`kind`, `type`, `_tag`) ni `===` solishtirish orqali union member'larini filter qiladi. Strukturaviy tekshiruvdan farqi: har variant'ning butun shape'ini emas, faqat bitta tag property literal'ini taqqoslash yetadi.

TS 4.4 ning "aliased conditions" yaxshilanishi — `const isUserPresent = user !== null` pattern bilan boolean variable saqlanib, keyingi `if (isUserPresent)` da narrowing qo'llanadi.

TS 5.4 closure narrowing yaxshilanishi — `const` va hech qachon assign qilinmaydigan parameter uchun narrowing closure ichida avval ham saqlanardi; 5.4 buni `let` variable va parameter'ga ham kengaytirdi, agar closure ularning oxirgi assignment'idan keyin yaratilgan bo'lsa (nested function ichida assign bo'lsa — ishlamaydi).

</details>

</details>

---

### Savol 12: TypeScript type inference algoritmi va contextual typing qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type inference ikki yo'nalishda ishlaydi: **best common type** — array literal yoki bir nechta candidate'dan umumiy tipni hisoblash (`getUnionType` orqali union qilish), **contextual typing** (`getContextualType`) — function parameter, callback, object literal uchun tipni "tashqaridan" yuborish. Context mavjud joyda inference uni hisobga oladi, yo'q joyda — element type'larini birlashtirib base type chiqaradi.

### To'liq tushuntirish

TS checker har expression uchun tipni `getTypeOfExpression` orqali hisoblaydi. Lekin ba'zi joylarda — function argument, return statement, callback parameter — checker contextual type'ni "tashqaridan" yuboradi.

**Inference algoritm bosqichlari:**

1. **Contextual type collection**: TS expression joylashgan joyni topadi (assignment target, function parameter, return type). Agar context'da type mavjud bo'lsa — bu "contextual type" sifatida saqlanadi.
2. **Inference candidate'lar**: Har expression uchun tip aniqlanadi.
3. **Best common type** (array va bir nechta candidate uchun): `[1, "x", true]` element type'lari `number | string | boolean` ga birlashtiriladi (`getUnionType`).
4. **Widening**: literal type'lar context yo'q joyda widening'ga uchraydi (`getWidenedType`): `"hello"` → `string`, `10` → `number`. `as const` yoki contextual literal type bo'lsa — widening to'xtatiladi.
5. **Final type assignment**: Variable/parameter ga aniqlangan type biriktiriladi.

**Contextual typing misol:**

```typescript
// Context yo'q — widening ishlaydi
const arr1 = [1, 2, 3];          // number[]
const callback1 = (x) => x * 2;  // ❌ noImplicitAny xato (x: any)

// Context bor — inference contextual type'dan oladi
const callback2: (x: number) => number = (x) => x * 2;
// x: number — context'dan, widening yo'q

[1, 2, 3].map(x => x * 2);
// x: number — Array<T>.map signature'idan kontekstual
```

### Kod misol

```typescript
// === Best common type ===
const arr = [1, "two", true];
// Algoritm:
// 1. Element types: number, string, boolean
// 2. Common supertype: union (number | string | boolean)
// 3. Array element wrapping: (number | string | boolean)[]

// === Contextual typing — function parameter ===
interface Handler {
  (event: MouseEvent): void;
}

const onClick: Handler = (e) => {
  console.log(e.clientX); // ✅ e: MouseEvent (contextual)
};

// === Contextual typing — object literal ===
type Config = { url: string; method: "GET" | "POST" };
const config: Config = {
  url: "/api",
  method: "GET", // literal saqlanadi context'dan, "string" emas
};

// === Inheritance — return type contextual ===
class Base {
  process(): string { return "base"; }
}
class Child extends Base {
  process() { return "child"; } // return type: string — literal "child" widening'ga uchraydi
}

// === Generic inference ===
function identity<T>(x: T): T { return x; }
const r1 = identity("hello");    // T inferred: "hello" (literal)
const r2 = identity<string>("hello"); // T explicit: string

// Inference algoritmi: TS argument type'idan T'ni "extract" qiladi
// `x: T` da T = typeof argument
```

### Edge Cases

- **Widening literal context'da to'xtaydi**: `const config: Config = { method: "GET" }` — `"GET"` literal saqlanadi (context literal union talab qiladi).
- **Generic inference cheklovlari**: `function f<T extends string>(x: T)` — argument literal bo'lsa T literal, widened bo'lsa T base type.
- **Inference from usage**: IDE quick fix (codefix) — function tanasidagi `x.toUpperCase()` ga qarab `x: string` annotation'ini taklif qiladi (bu actual type inference emas, balki tahrir taklifi).
- **Reverse inference**: `const x: number[] = [...]` — array literal type kontekstdan `number[]` ga widened (har element literal saqlanmaydi).
- **`satisfies` (TS 4.9+)**: type-check qiladi, lekin inference saqlanadi — `{ a: 1 } satisfies Record<string, number>` → type `{ a: 1 }`, `Record<string, number>` emas.

### Follow-up savollar

1. **"Generic inference'da `T` aniqlanmasa nima bo'ladi?"** — TS `unknown` yoki default constraint'ni oladi. `function f<T>(x?: T)` da argument yo'q bo'lsa, T `unknown`.
2. **"Contextual typing nima uchun har joyda ishlamaydi?"** — Context aniq bo'lishi kerak. `let x; x = (a) => a * 2` — `let x` da context yo'q, callback `any` parameter oladi. Yechim: `let x: (a: number) => number = ...`.

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript checker'da generic inference `inferTypes` (`checker.ts`) funksiyasi orqali boshlanadi: argument va parameter tiplarini taqqoslab, har type parameter uchun candidate'lar to'plami yig'iladi (covariant/contravariant position'larga qarab). `inferTypeArguments` bu jarayonni boshqaradi.

`getInferredTypes` bosqichi: har type parameter uchun yig'ilgan candidate massivini bitta tipga sig'diradi — covariant position'da candidate'lar `getUnionType` orqali birlashtiriladi.

Widening `getWidenedType` orqali: literal type'lar (`StringLiteral`, `NumberLiteral`, `BooleanLiteral`) base type'iga aylantiriladi. Istisno: `as const`, contextual literal type, va "fresh literal type" hali widening'ga uchramagan holatda.

"Fresh literal type" — TS internal concept. Object literal'da har property dastlab "fresh literal": context yo'q bo'lsa widening'ga uchraydi, context bor bo'lsa literal saqlanadi.

Contextual type propagation recursive ishlaydi: `register({ port: getPort() })` da `register` parameter type'i `{ port: T }` bo'lsa, `getPort()` ham `T` contextual type'ni oladi (nested propagation).

</details>

</details>

---

## Output prediction savollari

### Savol 13: Quyidagi kodda har bir o'zgaruvchining TypeScript type'ini ayting [Junior+]

```typescript
const a = "hello";
let b = "hello";
let c = "hello" as const;

const obj = { x: 10, y: "hi" };
const obj2 = { x: 10, y: "hi" } as const;

obj.x = 20;
// obj2.x = 20; // Uncomment qilinsa?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`a: "hello"` (literal, const), `b: string` (widened, let), `c: "hello"` (as const), `obj: { x: number; y: string }`, `obj2: { readonly x: 10; readonly y: "hi" }`. `obj.x = 20` ✅, `obj2.x = 20` ❌ Cannot assign to read-only.

### To'liq tushuntirish

```typescript
const a = "hello";
// TS type: "hello" — literal saqlanadi (const qayta assign mumkin emas)

let b = "hello";
// TS type: string — widened (let qayta assign mumkin)

let c = "hello" as const;
// TS type: "hello" — as const widening'ni to'xtatdi

const obj = { x: 10, y: "hi" };
// TS type: { x: number; y: string } — property'lar widened

const obj2 = { x: 10, y: "hi" } as const;
// TS type: { readonly x: 10; readonly y: "hi" } — literal + readonly

obj.x = 20;    // ✅ Ishlaydi — property mutable
// obj2.x = 20; // ❌ Cannot assign to 'x' because it is a read-only property
```

`as const` va `readonly` faqat compile-time. Runtime da `typeof` hammasi uchun bir xil. `obj2.x = 20` ni JavaScript sifatida ishlatsa ishlaydi (Object.freeze yoqilmagan).

### Edge Cases

- `const obj = { ... }` — object reference const, lekin property'lar mutable
- `Object.freeze(obj)` — runtime da property mutate qilinmaydi (silent fail yoki throw in strict mode)
- `as const` ichidagi array — `readonly` tuple bo'ladi
- `as const` nested object — recursive readonly + literal

### Follow-up savollar

1. **"`obj2.x = 20` ni runtime da to'xtatish uchun nima kerak?"** — `Object.freeze(obj2)`. TypeScript `readonly` faqat compile-time himoya.

</details>

---

### Savol 14: Type inference quiz — har satrda type nima? [Middle]

`strictNullChecks: true` va `noImplicitAny: true` rejimida:

```typescript
let a = [];
const b = [1, 2, 3];
const c = [1, "two", true];
const d = [1, 2, 3] as const;
const f = null;
let g = null;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`a: any[]` (evolving array), `b: number[]`, `c: (string | number | boolean)[]` (union, tuple emas), `d: readonly [1, 2, 3]` (readonly tuple), `f: null` (literal), `g: any` (evolving any).

### To'liq tushuntirish

```typescript
let a = [];
// type: any[] — "evolving array". TS element tipini bilmaydi.
// noImplicitAny ham buni to'xtatmaydi — maxsus holat.
// Yaxshiroq: let a: number[] = []

const b = [1, 2, 3];
// type: number[]

const c = [1, "two", true];
// type: (string | number | boolean)[] — union array, tuple emas!
// Tuple kerak: const c = [1, "two", true] as const → readonly [1, "two", true]

const d = [1, 2, 3] as const;
// type: readonly [1, 2, 3] — readonly tuple, har element literal

const f = null;
// type: null

let g = null;
// type: any — "evolving any". null widening bo'lgani uchun
// TS dastlab any qo'yadi, keyingi assign larga qarab evolve
// g = "hello" → g: string, g = 42 → g: number
```

**Evolving pattern:** `let a = []` va `let g = null` — TS dastlab `any[]`/`any`, lekin keyingi assignment ga qarab type evolve qiladi. Bu faqat `let` + initialization bilan ishlaydi.

### Edge Cases

- `[].push(1)` — bu yerda `[]` inline `never[]` literal: u biror variable'ga bog'lanmagani uchun evolve qilmaydi. `push` parametri `never` bo'lgani uchun `1` argumenti xato beradi: `Argument of type '1' is not assignable to parameter of type 'never'`. Evolving array faqat `[]` biror variable'ga (`let a = []` yoki `const empty = []`) assign qilinganda hosil bo'ladi
- `const empty = []` — `noImplicitAny: true` da evolving `any[]`: keyingi `push`/element assign'lar tipni aniqlamasa, o'qishda `TS7034 'empty' implicitly has type 'any[]'` xatosi chiqadi. `noImplicitAny: false` da esa evolving mexanizmi yo'q — `never[]`. `const` reassign qila olmaydi, lekin push orqali mutate qilib evolve qilishi mumkin
- `let x: null = null` — type aniq `null`, evolve qilmaydi (annotation evolving any'ni to'xtatadi)
- `noImplicitAny: true` evolving any/array'ni to'xtatmaydi (`let a = []` baribir `any[]`), faqat parameter implicit any'ni ushlaydi

### Follow-up savollar

1. **"Evolving array'dan qutulish uchun nima qilish kerak?"** — Annotation yozish: `let a: number[] = []`. Bu best practice — type'ni boshidan aniq belgilash.

</details>

---

## Coding challenges

### Savol 15: `as const` bilan type-safe configuration [Middle+]

**Savol:** `as const` va `typeof`/`keyof` ishlatib, type-safe configuration yarating. `getConfig` funksiyasi faqat mavjud kalitlarni qabul qilsin va aniq literal type qaytarsin:

```typescript
// getConfig("apiUrl")  → "https://api.example.com" (literal!)
// getConfig("port")    → 3000 (literal!)
// getConfig("debug")   → false (literal!)
// getConfig("invalid") → ❌ compile error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`as const` bilan config object yaratish. Generic constraint `K extends keyof typeof CONFIG`. Return type `typeof CONFIG[K]` — kalitga qarab aniq literal qaytaradi.

### Kod misol

```typescript
const CONFIG = {
  apiUrl: "https://api.example.com",
  port: 3000,
  debug: false,
  maxRetries: 5,
} as const;

type ConfigKey = keyof typeof CONFIG;

function getConfig<K extends ConfigKey>(key: K): typeof CONFIG[K] {
  return CONFIG[key];
}

const url = getConfig("apiUrl");
// type: "https://api.example.com"

const port = getConfig("port");
// type: 3000

const debug = getConfig("debug");
// type: false

// getConfig("invalid");
// ❌ Argument of type '"invalid"' is not assignable
```

### To'liq tushuntirish

- `as const` — barcha qiymatlar literal type va readonly
- `keyof typeof CONFIG` — `"apiUrl" | "port" | "debug" | "maxRetries"`
- Generic `K extends ConfigKey` — har chaqiruvda aniq kalitni biladi
- `typeof CONFIG[K]` (indexed access type) — return type kalitga qarab aniq literal (string, number, boolean emas!)

Boshqa variant — `Object.keys(CONFIG)` type-safe iterate qilish:

```typescript
function logAllConfig(): void {
  (Object.keys(CONFIG) as ConfigKey[]).forEach(key => {
    console.log(`${key}: ${CONFIG[key]}`);
  });
}
```

### Edge Cases

- `as const` nested object'da ham ishlaydi — to'liq frozen
- `Object.keys` return type `string[]` (TypeScript design choice) — manual cast kerak
- Generic constraint `K extends ConfigKey` instead of `K extends keyof typeof CONFIG` — ikkalasi bir xil, lekin alias aniqroq

### Follow-up savollar

1. **"Runtime da config'ni o'zgartirish mumkinmi?"** — `as const` faqat compile-time. Runtime da `Object.freeze(CONFIG)` ishlatish kerak haqiqiy immutability uchun.

</details>

---

### Savol 16: User-defined type guard + `unknown` validation [Middle+]

**Savol:** API javobini `unknown` dan `User` ga xavfsiz convert qiluvchi `parseUser` funksiyasini yozing. Format noto'g'ri bo'lsa — `null` qaytarsin.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  isActive: boolean;
}

function parseUser(data: unknown): User | null {
  // Implement qiling
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type guard helper funksiya bilan har field tekshirish (`typeof`, `in`). To'g'ri bo'lsa `User` shape ga cast va qaytarish, noto'g'ri bo'lsa `null`.

### Kod misol

```typescript
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.id === "number" &&
    typeof v.name === "string" &&
    typeof v.email === "string" &&
    typeof v.isActive === "boolean"
  );
}

function parseUser(data: unknown): User | null {
  return isUser(data) ? data : null;
}

// Test
const valid = { id: 1, name: "Ali", email: "a@t.com", isActive: true };
const invalid = { id: "1", name: "Ali", email: "a@t.com", isActive: true };

console.log(parseUser(valid));    // → { id: 1, name: "Ali", ... }
console.log(parseUser(invalid));  // → null (id string)
console.log(parseUser(null));     // → null
console.log(parseUser("string")); // → null

// Production usage
const data: unknown = await fetch("/api/user").then(r => r.json());
const user = parseUser(data);
if (user) {
  console.log(user.name); // ✅ type: User
}
```

### To'liq tushuntirish

- `isUser(value): value is User` — type predicate
- `typeof value === "object" && value !== null` — `typeof null === "object"` legacy bug uchun
- `Record<string, unknown>` intermediate cast — `as any` dan xavfsizroq
- Har field tekshirish: `typeof v.id === "number"` (ham property mavjudligini, ham tipini)
- Production da `zod`/`valibot` afzal — deklarativ schema va aniq xato xabarlar

### Edge Cases

- `typeof null === "object"` — JS legacy bug, manual check kerak
- Optional field uchun: `(typeof v.role === "string" || v.role === undefined)`
- Nested object — recursive validation kerak yoki schema library
- TS 5.5+ — funksiya tanasidan avtomatik type predicate inference, lekin aniq yozish IDE uchun aniqroq

### Follow-up savollar

1. **"`zod` bilan bu qanday yoziladi?"** —
```typescript
import { z } from "zod";
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string(),
  isActive: z.boolean(),
});
type User = z.infer<typeof UserSchema>;

function parseUser(data: unknown): User | null {
  const result = UserSchema.safeParse(data);
  return result.success ? result.data : null;
}
```

</details>

---

### Savol 17: Discriminated union va exhaustive check [Middle+]

**Savol:** `Result<T>` discriminated union yozing va `handleResult` funksiyasi exhaustive bo'lsin — barcha variant'lar qamrab olinishi shart:

```typescript
type Result<T> = /* implement */;

function handleResult<T>(result: Result<T>): string {
  // Har holat handle qilinsin, never check bilan
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Tag-based union — `{ kind: "success", data: T }` va `{ kind: "error", message: string }` va `{ kind: "loading" }`. Switch da har case + default `never` exhaustive check.

### Kod misol

```typescript
type Result<T> =
  | { kind: "success"; data: T }
  | { kind: "error"; message: string }
  | { kind: "loading" };

function handleResult<T>(result: Result<T>): string {
  switch (result.kind) {
    case "success":
      return `Data: ${JSON.stringify(result.data)}`;
    case "error":
      return `Error: ${result.message}`;
    case "loading":
      return "Loading...";
    default:
      // Exhaustive check — agar yangi variant qo'shilsa, compile error
      const _exhaustive: never = result;
      return _exhaustive;
  }
}

// Test
const r1: Result<number> = { kind: "success", data: 42 };
const r2: Result<number> = { kind: "error", message: "Not found" };
const r3: Result<number> = { kind: "loading" };

console.log(handleResult(r1)); // "Data: 42"
console.log(handleResult(r2)); // "Error: Not found"
console.log(handleResult(r3)); // "Loading..."
```

### To'liq tushuntirish

- Discriminated union `kind` (yoki `type`, `tag`) literal field bilan — narrowing tag asosida
- `switch (result.kind)` har case'da TS aniq narrowing qiladi
- `default` block da `_exhaustive: never = result` — agar barcha variant qamrab olinmagan bo'lsa, `result` `never` emas qoldiqqa teng bo'ladi va compile error chiqadi
- Yangi variant qo'shilganda (`{ kind: "cancelled" }`) avtomatik xato chiqadi — refactoring xavfsiz

### Edge Cases

- `never` exhaustive pattern faqat compile-time — runtime da `_exhaustive: never = result` har doim "qiymat" assign bo'ladi (lekin TS compile da to'xtatadi)
- Tag field bir xil bo'lishi shart har variant'da (`kind`, emas `kind` va `type` aralash)
- TS 4.x — narrowing closure ichida saqlanmaydi, lekin switch case ichida ishlaydi
- Tag literal — string yoki numeric literal bo'lishi mumkin

### Follow-up savollar

1. **"`assertNever` helper funksiya bilan yaxshiroq qilish mumkinmi?"** —
```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

// Switch da:
default: return assertNever(result);
```
Compile-time exhaustive check + runtime safety.

</details>

---

## Bug fix savollari

### Savol 18: `fetch` kodida xatolarni toping [Middle]

**Savol:** Bu kodda bir nechta xato bor. Hammasini toping va tuzating:

```typescript
function fetchUser(id: number) {
  const response = fetch(`/api/users/${id}`);
  const user = response.json();
  return user.name.toUpperCase();
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Xatolar: (1) `fetch` `Promise` qaytaradi, `await` yo'q; (2) `.json()` ham `Promise`; (3) `user: any` — type safety yo'q; (4) `response.ok` tekshirilmagan; (5) return type aniq emas.

### Kod misol

To'g'ri variant:

```typescript
interface User {
  name: string;
}

async function fetchUser(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  const data: unknown = await response.json();

  // Runtime validation
  if (
    typeof data !== "object" || data === null ||
    !("name" in data) ||
    typeof (data as Record<string, unknown>).name !== "string"
  ) {
    throw new Error("Invalid user format");
  }

  return (data as User).name.toUpperCase();
}
```

### To'liq tushuntirish

Xato 1: `fetch` `Promise<Response>` qaytaradi — `await` kerak yoki `.then()`. Source da `response: Promise<Response>` edi.

Xato 2: `response.json()` ham `Promise<any>` qaytaradi — `await` kerak.

Xato 3: `.json()` return type `any` — type safety yo'q. `unknown` cast va validation kerak.

Xato 4: HTTP error (404, 500) `.json()` da exception otmaydi — `response.ok` qo'lda tekshirilishi kerak.

Xato 5: Function `Promise` qaytaradi, lekin return type yozilmagan — public API uchun `Promise<string>` aniq yozish kerak.

### Edge Cases

- `response.ok` `200-299` status uchun `true`. `redirects` (`300-399`) — `false`
- `.json()` invalid JSON bo'lsa `SyntaxError` otadi — try/catch kerak
- `AbortController` bilan timeout — production da muhim
- `fetch` Node.js da 18+ versiyada native (avval `node-fetch` kerak edi)

### Follow-up savollar

1. **"Production'da fetch dan boshqa qanday tool ishlatish mumkin?"** — `axios` (interceptors, baseURL), `ky` (modern fetch wrapper), `@tanstack/react-query` (React uchun caching).
2. **"`zod` bilan validation qanday qilinadi?"** —
```typescript
const UserSchema = z.object({ name: z.string() });
const data = UserSchema.parse(await response.json());
return data.name.toUpperCase();
```

</details>

---

### Savol 19: Type assertion abuse — kodda xato qaerda? [Middle]

**Savol:** Bu kodda type assertion noto'g'ri ishlatilgan. Sabab nima va qanday tuzatish kerak?

```typescript
interface User {
  name: string;
  email: string;
  isActive: boolean;
}

function loadUser(): User {
  const raw = localStorage.getItem("user"); // string | null
  return JSON.parse(raw as string) as User;
}

const user = loadUser();
console.log(user.name.toUpperCase()); // Crashes sometimes
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

(1) `raw as string` — `null` bo'lsa `JSON.parse(null)` `null` qaytaradi, keyingi as cast type safety bermaydi. (2) `as User` — `JSON.parse` `any`, runtime tekshirilmaydi. Yechim: null check + validation.

### Kod misol

To'g'ri variant:

```typescript
interface User {
  name: string;
  email: string;
  isActive: boolean;
}

function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.name === "string" &&
    typeof v.email === "string" &&
    typeof v.isActive === "boolean"
  );
}

function loadUser(): User | null {
  const raw = localStorage.getItem("user");
  if (raw === null) return null;

  try {
    const parsed: unknown = JSON.parse(raw);
    return isUser(parsed) ? parsed : null;
  } catch {
    return null; // Invalid JSON
  }
}

const user = loadUser();
if (user !== null) {
  console.log(user.name.toUpperCase()); // ✅
}
```

### To'liq tushuntirish

Asl kod muammolari:

1. **`raw as string`** — `localStorage.getItem` `string | null` qaytaradi. `null` bo'lganda `JSON.parse(null)` `"null"` ga aylantirib, `null` ni parse qiladi → `null` qaytaradi
2. **`as User`** — type assertion runtime da hech narsa qilmaydi. Aslida `null` yoki noto'g'ri shape bo'lsa, `user.name.toUpperCase()` da crash
3. **Try/catch yo'q** — invalid JSON bo'lsa `SyntaxError`
4. **Return type `User`** — aslida `null` qaytarish ham mumkin, type misleading

To'g'ri yondashuv:
- Null check explicit
- Try/catch JSON parse uchun
- Validation type guard
- Return type `User | null` haqiqatga mos

### Edge Cases

- `localStorage` server-side rendering da yo'q (Next.js) — `typeof window === "undefined"` check
- `localStorage` SecurityError otishi mumkin (private mode, third-party context)
- Large data — `localStorage` origin'iga odatda ~5 MiB cheklov (browser'ga qarab farq qiladi, aniq spec'da belgilanmagan), limitdan oshsa `QuotaExceededError`
- Stale data — schema o'zgarsa eski data invalid bo'lishi mumkin (versioning kerak)

### Follow-up savollar

1. **"Storage abstraction qanday qilish kerak?"** — Generic `Storage<T>` wrapper: `class TypedStorage<T> { get(key: string, schema: Schema<T>): T | null }`. `zod` bilan integratsiya.

</details>

---

## Xulosa

Bu bo'limdagi savollar TypeScript ning type system asoslarini qamrab oldi:

- **Type annotations va inference** — qachon yozish kerak, qachon inference ishonchli
- **Primitive types** — string/number/boolean/bigint/symbol/null/undefined, `string` vs `String`
- **null va undefined** — semantic farqi, `strictNullChecks` ta'siri
- **any vs unknown** — type safety, viral vs non-viral, use case'lar
- **never va void** — exhaustive check, throw funksiya, side effect
- **Literal types** — discriminated union, enum alternativasi, widening qoidalari
- **as const** — literal + readonly, type assertion'dan farqi
- **Type assertion va guards** — `as` vs `typeof`/`instanceof`/`in`/predicate
- **Non-null assertion (`!`)** — qachon xavfsiz, qachon xavfli
- **Double assertion** — test mock, legacy migration
- **Widening va narrowing** — control flow analysis, type predicate
- **Type inference algoritmi** — best common type, contextual typing, generic inference
- **Bug fix patterns** — `fetch` async handling, type assertion abuse, JSON validation
