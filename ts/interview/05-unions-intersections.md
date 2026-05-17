# Interview: Union va Intersection Types

> Type aliases, union types, intersection types, type guards (`typeof`, `instanceof`, `in`), discriminated unions, exhaustive checking, custom type guards, `assertNever` pattern, function overloads, `satisfies` validation bo'yicha interview savollari.

---

## Mundarija

**Nazariy savollar**
- Savol 1: Union vs Intersection farqi `[Junior+]`
- Savol 2: Discriminated union — uchta shart `[Middle]`
- Savol 3: Exhaustive checking va `assertNever` `[Middle+]`
- Savol 4: `typeof`/`instanceof`/`in` type guards `[Middle]`
- Savol 5: Custom type guard va `is` predicate `[Middle]`
- Savol 6: Function overload union bilan `[Middle+]`
- Savol 7: `satisfies` operator union/intersection bilan `[Senior]`

**Output savollar**
- Savol 8: Intersection property conflict `[Junior+]`
- Savol 9: `in` operator narrowing `[Middle]`

**Coding savollar**
- Savol 10: E-commerce order state — discriminated union `[Middle]`
- Savol 11: Traffic light state machine `[Middle+]`
- Savol 12: API response handler `ApiResult<T>` `[Middle+]`

**Bug fix savollar**
- Savol 13: Union narrowing — `in` operator xato `[Middle]`
- Savol 14: Intersection conflict — yashirin `never` `[Middle+]`

---

## Nazariy savollar

### Savol 1: Union va Intersection type farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Union (`|`) — qiymat bir nechta tip'lardan birortasi (OR). Intersection (`&`) — qiymat barcha tip'larning property'larini o'z ichiga oladi (AND). Set theory'da union qiymat to'plamini kengaytiradi, intersection tip talablarini kengaytiradi.

### To'liq tushuntirish

TypeScript tip tizimi set theory asosida ishlaydi. `string | number` — string'lar to'plami va number'lar to'plamining birlashmasi (qiymat ikkalasidan birortasi bo'lishi mumkin). `A & B` — A va B tip'larining barcha property'larini majburiy qilib qo'shadi (qiymat ikkala tipni qondirishi shart).

Object tip'larida ikki operatsiya teskari ishlaydi:
- Union: qiymat ko'pi bilan **umumiy** property'larga ega bo'ladi.
- Intersection: qiymat barcha tip'larning **birlashgan** property'lariga ega bo'lishi shart.

| | Union (`\|`) | Intersection (`&`) |
|---|---|---|
| Mantiq | OR — birortasi | AND — barchasi |
| Object property'lar | umumiy (kesishish) | birlashgan |
| Primitive | birortasi | odatda `never` |
| Set theory | qiymatlar to'plami kengayadi | tip talablari kengayadi |
| Use case | turli variant'lar | tip composition |

### Kod misol

```typescript
// Union — string YOKI number
type ID = string | number;
let userId: ID = "user-123";  // ✅
userId = 42;                  // ✅
// userId = true;             // ❌ boolean ID emas

// Intersection — ikkala tip'ning birlashgan property'lari
interface HasName { name: string }
interface HasAge { age: number }
type Person = HasName & HasAge;

const person: Person = {
  name: "Aziz",  // HasName'dan — MAJBURIY
  age: 25,       // HasAge'dan — MAJBURIY
};

// Object union — faqat umumiy property'lar to'g'ridan-to'g'ri o'qiladi
type Dog = { name: string; bark(): void };
type Cat = { name: string; meow(): void };
type Pet = Dog | Cat;

function greet(pet: Pet): void {
  console.log(pet.name);  // ✅ umumiy property
  // pet.bark();          // ❌ Cat'da yo'q
  // pet.meow();          // ❌ Dog'da yo'q
}

// Primitive intersection — odatda never
type Impossible = string & number;
// Impossible: never — bir qiymat ham string ham number bo'lolmaydi
```

### Edge Cases

- **Distributive union**: conditional type'da union har member uchun alohida hisoblanadi.
- **Object intersection conflict**: bir xil property nomi turli tip'larda — `never`.
- **`never` in union**: `T | never = T` (absorbed). `T & never = never` (absorbs).
- **`unknown` in union**: `T | unknown = unknown` (top type). `T & unknown = T`.
- **`any` in union**: `T | any = any` (yutib yuboradi).

### Follow-up savollar

1. **"Function intersection nima beradi?"** — Call signature overload: `((x: string) => void) & ((x: number) => void)` har ikkala chaqiruv tipini qabul qiladi.
2. **"Set theory'da union va intersection o'rni qanday?"** — Union: $A \cup B$ (kengaytirish). Intersection: $A \cap B$ (kesishish — qiymat ikkalasiga tegishli).
3. **"Excess property check union'da qanday ishlaydi?"** — Fresh literal hech bir union member'iga aniq mos kelmasa — error.

</details>

---

### Savol 2: Discriminated union nima? Uchta sharti? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Discriminated union — har bir union member'ida umumiy nomli **discriminant property** bor, bu property **literal type**'ga ega va har member'da farqli qiymat oladi. TypeScript shu property bo'yicha narrowing'ni avtomatik bajaradi.

### To'liq tushuntirish

Discriminated union pattern uchta majburiy shartdan iborat:

1. **Umumiy property**: har member'da bir xil nomli property mavjud.
2. **Literal type**: bu property tipi literal (string/number/boolean literal yoki literal union).
3. **Farqli qiymat**: har member shu property uchun yagona qiymat oladi.

Bu shartlar bajarilganda `switch (value.discriminant)` yoki `if (value.discriminant === "x")` ishlatib narrowing qilish mumkin — TypeScript shu property bo'yicha aniq member tipini biladi. Bu pattern Redux actions, API response, state machine, AST node, FSA model'larida fundamental.

### Kod misol

```typescript
// Klassik shape example
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;        // TS biladi: Circle
    case "square":
      return shape.side ** 2;                    // TS biladi: Square
    case "rectangle":
      return shape.width * shape.height;         // TS biladi: Rectangle
  }
}

// API response — status discriminant
type ApiResult<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: string; code: number }
  | { status: "loading" };

function render<T>(result: ApiResult<T>): string {
  switch (result.status) {
    case "success": return `Data: ${JSON.stringify(result.data)}`;
    case "error":   return `Error ${result.code}: ${result.error}`;
    case "loading": return "Loading...";
  }
}

// Redux actions
type Action =
  | { type: "USER_LOGIN"; payload: { userId: string } }
  | { type: "USER_LOGOUT" }
  | { type: "UPDATE_PROFILE"; payload: { name: string; email: string } };

function reducer(state: AppState, action: Action): AppState {
  switch (action.type) {
    case "USER_LOGIN":
      return { ...state, userId: action.payload.userId };
    case "USER_LOGOUT":
      return { ...state, userId: null };
    case "UPDATE_PROFILE":
      return { ...state, ...action.payload };
  }
}
```

### Edge Cases

- **Boolean discriminator**: `kind: true | false` — ishlaydi, lekin string literal aniqroq.
- **Numeric discriminator**: `kind: 1 | 2 | 3` — ishlaydi, but enum yoki string ko'pincha o'qish uchun yaxshi.
- **Discriminant bo'lmagan property**: agar property barcha member'larda bor, lekin literal emas — narrowing ishlamaydi.
- **Optional discriminator**: `kind?: "x"` — narrowing ishlamaydi (undefined ehtimoli).
- **Multiple discriminants**: bir nechta literal property bo'lsa ham ishlaydi, lekin bittasi yetarli.

### Follow-up savollar

1. **"`type` vs `kind` vs `tag` — qaysi nom yaxshi?"** — Konvensiya: Redux `type`, FSA-style `type`, akademik `tag`, React `kind`. Asosiysi loyihada birxil.
2. **"Yangi member qo'shilsa nima sodir bo'ladi?"** — Exhaustive check (`assertNever`) bo'lsa — compile error, bo'lmasa silent fallthrough.
3. **"Class hierarchy bilan farq nima?"** — Discriminated union — flat, structural, JSON-serializable. Class — nominal, method'lar bilan, `instanceof` ishlaydi.

</details>

---

### Savol 3: Exhaustive checking va `assertNever` pattern qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Exhaustive checking — union'ning barcha member'lari handle qilinganligini compile-time'da kafolatlash. `assertNever(value: never): never` funksiyasi `default` branch'ida chaqiriladi — barcha case'lar qoplangan bo'lsa `value` tipi `never`, qoldirilgan bo'lsa compile error.

### To'liq tushuntirish

TypeScript control flow analysis'i `switch`/`if-else` da har case bo'yicha tipni eliminate qiladi. Barcha union member'lari handle qilingandan keyin qolgan tip `never` bo'lishi shart. `assertNever` funksiyasi `never` parametr qabul qiladi va o'zi `never` qaytaradi — agar argument `never` emas bo'lsa, compile error.

Bu pattern yangi member qo'shilganda muhim:
1. Yangi member qo'shildi (`type Status` ga yangi qiymat).
2. `switch` da yangi case yozilmadi.
3. `default` branch'da `assertNever(status)` — endi `status` `never` emas, yangi member tipini saqlaydi.
4. Compile error — barcha unhandled joylar topiladi.

### Kod misol

```typescript
type Status = "active" | "inactive" | "banned";

function assertNever(value: never): never {
  throw new Error(`Unhandled value: ${JSON.stringify(value)}`);
}

function getLabel(status: Status): string {
  switch (status) {
    case "active":   return "Faol";
    case "inactive": return "Nofaol";
    case "banned":   return "Bloklangan";
    default:
      return assertNever(status);
      // Barcha case qoplangan — status: never
  }
}

// Union'ga yangi member qo'shilganda
type StatusV2 = "active" | "inactive" | "banned" | "suspended";

function getLabelV2(status: StatusV2): string {
  switch (status) {
    case "active":   return "Faol";
    case "inactive": return "Nofaol";
    case "banned":   return "Bloklangan";
    default:
      return assertNever(status);
      // ❌ Argument of type '"suspended"' is not assignable to type 'never'
      // Compiler "suspended" handle qilinmaganini ushladi
  }
}

// `if-else` da ham ishlaydi
function getColor(status: Status): string {
  if (status === "active")   return "green";
  if (status === "inactive") return "gray";
  if (status === "banned")   return "red";
  return assertNever(status);
}

// Discriminated union'da
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "keypress"; key: string };

function logEvent(event: Event): string {
  switch (event.type) {
    case "click":    return `Click at (${event.x}, ${event.y})`;
    case "keypress": return `Key: ${event.key}`;
    default:         return assertNever(event);
  }
}
```

### Edge Cases

- **Return type infer**: `assertNever` `never` qaytarganda, function return type'ga ta'sir qilmaydi (`never` har tipga assignable).
- **`throw` vs `return`**: `default: throw assertNever(...)` ham ishlaydi, lekin `return` kompozitsiyasi yaxshi.
- **Generic exhaustive**: generic `<T extends Status>` da ham `assertNever` ishlaydi.
- **`if-else` chain**: oxirgi `else` bo'lmasa — narrowing'ga e'tibor; explicit `else` afzal.
- **Production behavior**: `throw` qiladi — agar runtime'da yangi qiymat kelsa, fail fast.

### Follow-up savollar

1. **"Production'da `throw` o'rniga nima qilish kerak?"** — Sentry/logger orqali xabar berish, default fallback qaytarish. Lekin `throw` invariant'ni saqlash uchun yaxshi.
2. **"`never` type qachon paydo bo'ladi?"** — Exhaustive narrowing, `throw` function return, infinite loop, impossible intersection (`string & number`).
3. **"ESLint qoidasi bormi?"** — `@typescript-eslint/switch-exhaustiveness-check` — `assertNever` ishlatmasdan ham exhaustive check qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler tahlili**: TypeScript control flow `getFlowTypeOfReference` ichida har branch uchun type narrowing track qiladi. `switch` statement'i `analyzeSwitch` orqali iterate qilinadi — har case'da `narrowTypeBySwitchOnDiscriminant` `discriminantPropertyAccess` orqali narrow qiladi.

**`never` bottom type**: type lattice'da `never` eng tubdagi tip. `never <: T` (har tipga subtype). Bu `assertNever(value)` da `value: never` argumentni har funksiyada return qila olishini ta'minlaydi.

**Exhaustiveness vs `noFallthroughCasesInSwitch`**: ikkinchi flag `case` orasida `break`/`return` yo'qligini ushlaydi, exhaustiveness esa `case` yetishmasligini. Ikkalasi birga ishlaydi.

</details>

</details>

---

### Savol 4: `typeof`, `instanceof`, `in` type guard'lar farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`typeof` — primitive tip'lar uchun (`string`, `number`, `boolean`, `bigint`, `symbol`, `undefined`, `object`, `function`). `instanceof` — class/constructor uchun (prototype chain). `in` — property mavjudligi orqali object union'larni narrow qilish.

### To'liq tushuntirish

Har uchchovi runtime JavaScript operator — TypeScript ularga maxsus narrowing semantikasi qo'shgan:

**`typeof`** — JavaScript value'ning eng asosiy tipini olish. 8 ta natija bor, har biri TypeScript tipiga mos keladi. `typeof null === "object"` — JavaScript bug, TypeScript buni biladi (null'ni alohida tekshirish kerak).

**`instanceof`** — value'ning prototype chain'ida class constructor borligini tekshiradi. Class instance, custom error subclass, built-in tip'lar (`Date`, `RegExp`, `Map`) uchun. Cross-realm muammosi: iframe'lar orasida `Array` instance'lari `instanceof Array` ga `false` qaytarishi mumkin.

**`in` operator** — object'da property mavjudligini tekshiradi (own + prototype chain). Discriminated union'siz union'larni narrow qilish uchun ishlatiladi.

### Kod misol

```typescript
// 1. typeof — primitive narrowing
function format(value: string | number | boolean): string {
  if (typeof value === "string") {
    return value.toUpperCase();   // value: string
  }
  if (typeof value === "number") {
    return value.toFixed(2);      // value: number
  }
  return value ? "yes" : "no";    // value: boolean
}

// typeof null muammosi
function process(value: object | null): string {
  // ❌ Xavfli — null ham "object" qaytaradi
  // if (typeof value === "object") return value.toString();

  // ✅ Xavfsiz
  if (value !== null && typeof value === "object") {
    return value.toString();
  }
  return "null";
}

// 2. instanceof — class narrowing
class AppError extends Error {
  constructor(public code: number, message: string) {
    super(message);
  }
}

class NetworkError extends AppError {
  constructor(public url: string, message: string) {
    super(500, message);
  }
}

function handleError(error: Error): string {
  if (error instanceof NetworkError) {
    return `Network: ${error.url} — ${error.message}`;
  }
  if (error instanceof AppError) {
    return `App ${error.code}: ${error.message}`;
  }
  return `Error: ${error.message}`;
}

// Built-in instanceof
function describe(value: unknown): string {
  if (value instanceof Date) return value.toISOString();
  if (value instanceof RegExp) return value.source;
  if (value instanceof Map) return `Map(${value.size})`;
  if (value instanceof Array) return `Array(${value.length})`;
  return String(value);
}

// 3. in operator — property narrowing
type Bird = { fly(): void; eggs: number };
type Fish = { swim(): void; gills: boolean };
type Animal = Bird | Fish;

function move(animal: Animal): void {
  if ("fly" in animal) {
    animal.fly();   // animal: Bird
  } else {
    animal.swim();  // animal: Fish
  }
}

// `in` discriminated union bilan ham ishlaydi
type Result =
  | { ok: true; data: string }
  | { ok: false; error: Error };

function handle(result: Result): void {
  if ("data" in result) {
    console.log(result.data);   // result: { ok: true; data: string }
  } else {
    console.error(result.error);
  }
}
```

### Edge Cases

- **`typeof function`**: TypeScript class constructor ham `function` qaytaradi.
- **`instanceof` cross-realm**: iframe'lar orasida — `Symbol.hasInstance` bilan custom qilish mumkin.
- **`in` optional property bilan**: optional property `?` mavjud bo'lmasligi mumkin — `in` false qaytaradi.
- **`in` prototype chain**: `"toString" in obj` har object'da true (Object.prototype'dan keladi).
- **`hasOwnProperty` farqi**: faqat own property'larni tekshiradi (prototype emas).

### Follow-up savollar

1. **"Custom class'da `instanceof` qanday override qilinadi?"** — `static [Symbol.hasInstance](instance)` method.
2. **"`in` operator discriminated union bilan ortiqchami?"** — Yo'q. Discriminator literal property — `switch` aniqroq. `in` — property mavjudligi (struktura farqi) uchun.
3. **"`typeof` bilan custom class'larni narrow qilish mumkinmi?"** — Yo'q. `typeof instance === "object"` umumiy. `instanceof` ishlatish kerak.

</details>

---

### Savol 5: Custom type guard funksiya nima? Type predicate (`is`) qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Custom type guard — return tipi `value is T` shaklida bo'lgan funksiya. `true` qaytarsa — TypeScript value'ni `T` tipida deb hisoblaydi. `filter`, `find`, `every` kabi predikat-asosli method'lar bilan ishlaydi.

### To'liq tushuntirish

Type predicate (`x is T`) — funksiya signature'iga qo'shiladigan maxsus return type. TypeScript bu funksiyani chaqirish natijasini truthy bo'lsa, argument'ni `T` tipida deb narrow qiladi. Oddiy `boolean` return tipi narrowing'ga ta'sir qilmaydi — predicate alohida qo'shilishi shart.

TS 5.5+ da **inferred type predicates** — TypeScript ba'zi predicate'larni avtomatik infer qiladi (`filter(x => x !== null)` kabi). Lekin murakkab logic, generic, yoki external library funksiyalarida explicit predicate kerak.

### Kod misol

```typescript
// Asosiy type guard
function isString(value: unknown): value is string {
  return typeof value === "string";
}

const input: unknown = getUserInput();
if (isString(input)) {
  console.log(input.toUpperCase()); // ✅ input: string
}

// `isNonNullable` — keng tarqalgan utility
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const items: (string | null | undefined)[] = ["Aziz", null, "Vali", undefined];

// ❌ Oddiy predicate — narrowing ishlamaydi (TS 5.5'gacha)
const filtered1 = items.filter(item => item !== null && item !== undefined);
// type: (string | null | undefined)[]

// ✅ Type guard bilan
const filtered2 = items.filter(isNonNullable);
// type: string[]

// Discriminated union bilan
type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };

function isSuccess<T>(result: Result<T>): result is { ok: true; data: T } {
  return result.ok === true;
}

const results: Result<number>[] = [
  { ok: true, data: 1 },
  { ok: false, error: "404" },
  { ok: true, data: 2 },
];

const successful = results.filter(isSuccess);
// type: { ok: true; data: number }[]
const values = successful.map(r => r.data); // number[]

// Class narrowing
class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}

// Object struktura tekshiruvi
interface User {
  id: number;
  name: string;
  email: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as User).id === "number" &&
    typeof (value as User).name === "string" &&
    typeof (value as User).email === "string"
  );
}
```

### Edge Cases

- **Predicate logic xatosi**: `value is T` qaytsa ham boshqa narsa qaytarib bo'lmaydi — body to'g'ri narrowing qilishini ta'minlash.
- **Generic type guard**: `<T>(value: unknown): value is T` — xavfli, runtime tekshiruv yo'q, faqat unsafe escape hatch.
- **`asserts` farqi**: `is` boolean qaytaradi, `asserts` void va throw qiladi.
- **Negation narrowing**: `!isString(x)` — `x` `string`'dan tashqari tipga narrow bo'ladi.
- **Inferred predicates (TS 5.5+)**: `filter(x => x.length > 0)` — TS o'zi predicate'ni infer qiladi (oddiy holatlarda).

### Follow-up savollar

1. **"`zod` kabi library qanday qilib runtime validation + type guard beradi?"** — Schema'dan `parse` method generate qilinadi, schema tipi guard predicate sifatida ishlatiladi.
2. **"Type guard noto'g'ri logic bilan yozilsa nima bo'ladi?"** — TypeScript runtime'da tekshirmaydi — silent fail, runtime crash. Test bilan validate qilish kerak.
3. **"`is` predicate bilan negation qanday ishlaydi?"** — `if (!isString(x)) {}` — `x` `string` tipi olib tashlanadi.

</details>

---

### Savol 6: Function overload union bilan qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Function overload — bir nechta signature bilan call-site'da aniq tip beradi. Union parametr bilan ham ishlatish mumkin, lekin overload aniq input → aniq output bog'lanishini ta'minlaydi. Implementation signature mos kelishi shart.

### To'liq tushuntirish

Funksiya signature'da union parametr ishlatilganda, return tipi har holat uchun bir xil bo'ladi. Bu cheklov agar input tipiga qarab output tipi farqlanishi kerak bo'lsa muammo. Function overload bu vaziyatni hal qiladi: bir nechta signature deklaratsiya + bitta implementation. TypeScript call-site'da mos signature'ni topadi.

Overload buyrug'i:
1. Yuqoridan pastga signature'larni iterate qiladi.
2. Birinchi mos kelganini tanlaydi (eng tor avval).
3. Implementation signature'ni tashqaridan ko'rinmaydi — faqat overload signature'lari ko'rinadi.

### Kod misol

```typescript
// 1. Union parametr — return tipi cheklangan
function double1(value: string | number): string | number {
  if (typeof value === "string") return value + value;
  return value * 2;
}
const a = double1("ab");  // string | number — TS aniq tipni bilmaydi
const b = double1(5);     // string | number

// 2. Overload bilan — aniq mapping
function double(value: string): string;
function double(value: number): number;
function double(value: string | number): string | number {
  if (typeof value === "string") return value + value;
  return value * 2;
}
const c = double("ab"); // string — aniq
const d = double(5);    // number — aniq

// Real-world — createElement
function createElement(tag: "img"): HTMLImageElement;
function createElement(tag: "a"): HTMLAnchorElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const img = createElement("img");    // HTMLImageElement
const link = createElement("a");      // HTMLAnchorElement
const div = createElement("div");     // HTMLElement (umumiy)

// Method overload — class ichida
class EventBus {
  on(event: "click", handler: (e: MouseEvent) => void): void;
  on(event: "keypress", handler: (e: KeyboardEvent) => void): void;
  on(event: "load", handler: () => void): void;
  on(event: string, handler: Function): void {
    // implementation
  }
}

const bus = new EventBus();
bus.on("click", (e) => console.log(e.clientX));   // MouseEvent
bus.on("keypress", (e) => console.log(e.key));    // KeyboardEvent
bus.on("load", () => console.log("loaded"));      // no param

// Generic overload
function map<T, U>(arr: T[], fn: (item: T) => U): U[];
function map<T, K extends string, U>(
  arr: T[],
  key: K,
  fn: (value: T[K & keyof T]) => U
): U[];
function map(arr: any[], second: any, third?: any): any[] {
  if (typeof second === "function") {
    return arr.map(second);
  }
  return arr.map(item => third(item[second]));
}
```

### Edge Cases

- **Order matters**: eng tor signature avval yozilishi shart, aks holda noto'g'ri tanlash.
- **Implementation signature**: external'dan ko'rinmaydi — overload signature'lari ishlatilishi shart.
- **`this` parameter**: overload'da `this` tipini belgilash mumkin.
- **Discriminated union alternative**: ko'pincha overload o'rniga discriminated union aniqroq.
- **Conditional types alternative**: TS 4.x+ da generic + conditional type bilan overload o'rnini bosish mumkin.

### Follow-up savollar

1. **"Overload vs union return — qachon qaysi?"** — Input-output bog'lanishi kerak bo'lsa overload, oddiy union return bo'lsa union signature.
2. **"Conditional type bilan overload ni qanday almashtirish mumkin?"** — `T extends "img" ? HTMLImageElement : T extends "a" ? HTMLAnchorElement : HTMLElement`.
3. **"Implementation signature qanday bo'lishi kerak?"** — Barcha overload'larga umumiy parent tip — odatda union va `any` return.

<details>
<summary><strong>Deep Dive</strong></summary>

**Overload resolution algoritmi**: `chooseOverload` function compiler'da — har call-site uchun candidate signature'larni iterate qiladi. Selectiya order:
1. Generic'siz arity-mos signature'lar.
2. Default parametr bilan signature'lar.
3. Rest parametr bilan signature'lar.
4. Generic signature'lar.

Har qaysida signature mos kelishi `compareSignaturesIdentical` orqali tekshiriladi.

**Implementation hidden**: TypeScript design intent — implementation signature `any`/`unknown` parametr olishi mumkin, lekin consumer'lar uchun ko'rinmasligi kerak (capsulation).

**Discriminated alternative**: conditional type bilan `function fn<T extends Input>(x: T): Output<T>` — single signature, TypeScript map'ni hisoblaydi. Overload'dan farqi — readability va inferring power.

</details>

</details>

---

### Savol 7: `satisfies` operator union/intersection bilan qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies` (TS 4.9+) — value tipini tekshiradi (target tipga mosligini), lekin inferred tipni saqlab qoladi. Union/intersection target bilan — har property aniq variant'ga narrow bo'ladi, lekin `:` annotation farqli ravishda widen qilinmaydi.

### To'liq tushuntirish

`:` annotation tipni "widen" qiladi — inferred literal tip target tipga kengaytirildi. `satisfies` esa "narrow" qiladi — value tipi mos kelishi tekshiriladi, lekin TypeScript original inferred tipni saqlaydi. Bu union/intersection'da farq juda muhim:

- `: Config` bilan — `config.port` `string | number` (Config tipi).
- `satisfies Config` bilan — `config.port` `number` (inferred aniq tip).

Bu hatto `as const` bilan birga ishlatilganda — `satisfies` validation beradi, `as const` literal'larni saqlaydi.

### Kod misol

```typescript
type Config = Record<string, string | number>;

// 1. Annotation — aniq tip yo'qoladi
const config1: Config = { port: 3000, host: "localhost" };
config1.port;  // string | number — TS aniq bilmaydi
config1.invalid; // string | number — har string kalit ruxsat

// 2. satisfies — aniq tip saqlanadi + validation
const config2 = { port: 3000, host: "localhost" } satisfies Config;
config2.port;  // number — aniq
config2.host;  // string — aniq
// config2.invalid; // ❌ Property 'invalid' does not exist
// Type tekshirilgan, lekin shape closed qoldi

// Union literal bilan validation
type Status = "active" | "inactive" | "banned";
type StatusMap = Record<Status, string>;

const labels = {
  active: "Faol",
  inactive: "Nofaol",
  banned: "Bloklangan",
  // archived: "x", // ❌ 'archived' Status'da yo'q (satisfies tekshirdi)
} satisfies StatusMap;

labels.active;  // string — aniq

// Discriminated union'da
type Theme =
  | { mode: "light"; bg: "white"; fg: "black" }
  | { mode: "dark"; bg: "black"; fg: "white" };

const light = {
  mode: "light",
  bg: "white",
  fg: "black",
} satisfies Theme;
// light tipi: aniq — light variant
light.bg;  // "white" — literal saqlanadi

const dark: Theme = {
  mode: "dark",
  bg: "black",
  fg: "white",
};
// dark tipi: Theme (union)
dark.bg;  // "white" | "black" — widen qilingan

// Function return type — satisfies bilan refinement
function createUser() {
  return {
    id: 1,
    role: "admin",
    permissions: ["read", "write"],
  } satisfies { id: number; role: string; permissions: string[] };
}
const user = createUser();
user.role;          // string — aniq
user.permissions[0]; // string

// as const + satisfies — eng aniq
const routes = {
  home: { path: "/", method: "GET" },
  api: { path: "/api/v1", method: "POST" },
} as const satisfies Record<string, { path: string; method: string }>;

routes.home.path;   // "/" — literal
routes.api.method;  // "POST" — literal
```

### Edge Cases

- **Annotation + satisfies birga**: ikkalasini birga ishlatish — annotation ustun, lekin satisfies'ning narrowing'i yo'qoladi.
- **Function overload va satisfies**: function return tipini constraint qilish, lekin specific signature saqlash.
- **`as const` bilan**: `{} as const satisfies T` — literal saqlash + validation.
- **Excess property check**: `satisfies` excess property'larni ushlaydi (annotation kabi).
- **Generic'da satisfies**: generic constraint sifatida emas, lekin generic value bilan ishlatish mumkin.

### Follow-up savollar

1. **"`satisfies` vs `as` farqi nima?"** — `as` — unsafe assertion (tekshirmaydi). `satisfies` — type-safe validation + inferred type saqlash.
2. **"Qachon `:` annotation aniq kerak?"** — Variable boshqa joyda widen tip kerak bo'lganda (function parametr sifatida).
3. **"`satisfies` performance impact bormi?"** — Compile-time'da bir marta tekshiriladi, runtime'da yo'q.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler implementation**: `checkSatisfiesExpression` ichida ikki nuqta tekshiruvi:
1. `isTypeAssignableTo(sourceType, targetType)` — value target'ga mos kelishini tekshirish.
2. Original `getContextualType` ni saqlash — inferred type'ni return qilish.

**`as const` interaction**: `as const` literal'ni "frozen" qiladi — widen qilmaydi. `satisfies` ham widen qilmaydi. Birga ishlatilganda — eng tor (literal + readonly) tip saqlanadi.

**Use case'lar**:
- Theme/config object'lar (literal type'lar).
- Route definitions (path va method literal).
- Action creators (action type literal).
- Tagged template literal'lar (constant pattern).

</details>

</details>

---

## Output savollari

### Savol 8: Intersection property conflict — output [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bir xil nomli property turli tip'lar bilan intersection'da `never` bo'ladi. Bunday object yaratish mumkin emas.

### To'liq tushuntirish

Intersection (`A & B`) ikkala tip property'larining birlashmasini olganda, bir xil nomli property turli tip'larda bo'lsa, TypeScript ularning intersection'ini hisoblaydi: `string & number = never`. Compiler bu paytda error bermaydi (lazy resolution), faqat consumer kod object yaratishga uringanda — hech qanday qiymat `never` tipiga assignable emas — error chiqadi. Primitive intersection da literal subset qoidasi qo'llanadi: `string & "x" = "x"` (literal — string'ning torroq subset'i).

### Kod misol

```typescript
type A = { x: number; y: string };
type B = { y: number; z: boolean };
type C = A & B;

// C tipining tarkibi:
type CResolved = {
  x: number;     // A dan
  y: never;      // string & number = never
  z: boolean;    // B dan
};

// const obj: C = { x: 1, y: ???, z: true };
// y uchun hech qanday qiymat mos kelmaydi — never

// Edge case — primitive'larda
type StringNumber = string & number;     // never
type StringString = string & "literal";  // "literal" (literal string subset)
type TrueBoolean = true & boolean;       // true
type DateNumber = Date & number;         // never (mos kelmaydigan tip'lar)
```

### Edge Cases

- **Literal subset**: `string & "x"` = `"x"` (literal string'ning kichik to'plami).
- **Object birga**: bir xil property bir xil tip — birlashadi (`{x:1} & {x:1}` = `{x:1}`).
- **Function intersection**: parametr — union, return — intersection.

### Follow-up savollar

1. **"Intersection vs extends conflict farqi?"** — `extends` darhol error, `&` silent `never`.
2. **"Qanday qilib conflict'ni avoid qilish kerak?"** — `Omit<A, "y"> & B` bilan property'ni olib tashlash.

</details>

---

### Savol 9: `in` operator narrowing — output [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`in` operator faqat farqlovchi property bilan narrowing qiladi. Discriminated union'da `switch` aniqroq.

### To'liq tushuntirish

`"prop" in obj` JavaScript runtime operator — property mavjudligini (own + prototype chain) tekshiradi. TypeScript bu operator'ga narrowing semantikasi qo'shgan: union member'lar orasidan shu property'ga ega bo'lganlarini saqlaydi, qolganlarini olib tashlaydi. Discriminated union'da har case uchun unique property bo'lsa, `in` aniq narrowing beradi. Lekin barcha member'larda mavjud property — narrowing'ga ta'sir qilmaydi.

### Kod misol

```typescript
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "keypress"; key: string }
  | { type: "scroll"; position: number };

function logEvent(event: Event): string {
  if ("x" in event) {
    return `Click at (${event.x}, ${event.y})`;
  }
  if ("key" in event) {
    return `Key: ${event.key}`;
  }
  return `Scroll: ${event.position}`;
}

console.log(logEvent({ type: "click", x: 10, y: 20 }));
// Output: Click at (10, 20)

console.log(logEvent({ type: "keypress", key: "Enter" }));
// Output: Key: Enter

console.log(logEvent({ type: "scroll", position: 100 }));
// Output: Scroll: 100

// Tushuntirish:
// "x" in event       → faqat click'da bor → narrow Click
// "key" in event     → faqat keypress'da → narrow Keypress
// Qolgan branch      → scroll qoladi (exhaustive narrowing)

// Discriminated alternative — aniqroq
function logEventV2(event: Event): string {
  switch (event.type) {
    case "click":    return `Click at (${event.x}, ${event.y})`;
    case "keypress": return `Key: ${event.key}`;
    case "scroll":   return `Scroll: ${event.position}`;
  }
}
```

### Edge Cases

- **Optional property bilan `in`**: optional property mavjud bo'lmasligi mumkin — `in` har doim true emas.
- **Inherited property**: `"toString" in {}` — true (Object.prototype).
- **Numeric kalit**: `0 in arr` — array'da index tekshiruvi.

### Follow-up savollar

1. **"`in` `hasOwnProperty` bilan farq nima?"** — `in` prototype chain'ni ham tekshiradi, `hasOwnProperty` faqat own property.
2. **"Discriminated union'siz `in` qachon kerak?"** — Eski API'larda discriminator yo'q bo'lsa, structural farq bo'yicha narrow qilish.

</details>

---

## Coding savollar

### Savol 10: E-commerce order state — discriminated union va exhaustive check [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har status uchun alohida shape (`pending` items, `processing` ETA, `shipped` tracking, `delivered` Date, `cancelled` reason) — `assertNever` bilan exhaustive guarantee.

### To'liq tushuntirish

E-commerce order state machine — klassik discriminated union pattern. Har status uchun alohida payload shape kerak (pending — itemlar, shipped — tracking number, cancelled — sabab). `status` literal property discriminator sifatida — switch narrowing avtomatik ishlaydi. `assertNever` default branch'ida — yangi status qo'shilsa compile-time error beradi (har joyda handle qilish majburiy). State transition matrix `Record<Status, Status[]>` orqali — qaysi status'dan qaysiga o'tish ruxsat etilgan, type-safe.

### Kod misol

```typescript
type Order =
  | { status: "pending";    orderId: string; items: string[] }
  | { status: "processing"; orderId: string; estimatedTime: number }
  | { status: "shipped";    orderId: string; trackingNumber: string }
  | { status: "delivered";  orderId: string; deliveredAt: Date }
  | { status: "cancelled";  orderId: string; reason: string };

function assertNever(value: never): never {
  throw new Error(`Unhandled order: ${JSON.stringify(value)}`);
}

function getOrderMessage(order: Order): string {
  switch (order.status) {
    case "pending":
      return `Order ${order.orderId}: ${order.items.length} items waiting`;
    case "processing":
      return `Order ${order.orderId}: Ready in ~${order.estimatedTime} min`;
    case "shipped":
      return `Order ${order.orderId}: Track at ${order.trackingNumber}`;
    case "delivered":
      return `Order ${order.orderId}: Delivered ${order.deliveredAt.toLocaleDateString()}`;
    case "cancelled":
      return `Order ${order.orderId}: Cancelled — ${order.reason}`;
    default:
      return assertNever(order);
  }
}

// Type-safe state transitions
function canTransition(from: Order["status"], to: Order["status"]): boolean {
  const transitions: Record<Order["status"], Order["status"][]> = {
    pending:    ["processing", "cancelled"],
    processing: ["shipped", "cancelled"],
    shipped:    ["delivered"],
    delivered:  [],
    cancelled:  [],
  };
  return transitions[from].includes(to);
}

const order: Order = {
  status: "shipped",
  orderId: "ORD-001",
  trackingNumber: "TR-12345",
};
console.log(getOrderMessage(order));
// Order ORD-001: Track at TR-12345
```

### Edge Cases

- **`orderId` umumiy property**: barcha member'larda bor, lekin discriminator emas (literal emas).
- **Yangi status qo'shilsa**: `assertNever` darhol compile error beradi.
- **Generic `Order<T>`**: qo'shimcha metadata tipini parametrize qilish mumkin.

### Follow-up savollar

1. **"Order history qanday model qilinadi?"** — `events: OrderEvent[]` — har transition uchun event.
2. **"`zod` schema'ga qanday o'tkaziladi?"** — `z.discriminatedUnion("status", [...])`.

</details>

---

### Savol 11: Traffic light state machine type-safe transition [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`NextState` mapped type bilan har state uchun keyingi state'ni hardcode qilish. Generic constraint `S extends TrafficLight` aniq literal saqlaydi.

### To'liq tushuntirish

Type-safe state machine — har state uchun keyingi state'ni compile-time'da kafolatlash. `NextState` — mapped object type, har key (state) o'zining keyingi state literal'iga map qiladi. Generic `<S extends TrafficLight>` constraint bilan input literal saqlanadi, return type indexed access `NextState[S]` orqali — har specific input uchun aniq output tipi inferred. `as NextState[S]` assertion zarur, chunki `case "red"` ichida TypeScript `S` ni `"red"` deb narrow qila olmaydi (literal generic constraint subtle). Terminal state'lar (transition yo'q) `never` qaytaradigan tip orqali modellashtiriladi.

### Kod misol

```typescript
type TrafficLight = "red" | "green" | "yellow";

type NextState = {
  red: "green";
  green: "yellow";
  yellow: "red";
};

function assertNever(value: never): never {
  throw new Error(`Unexpected state: ${value}`);
}

function transition<S extends TrafficLight>(state: S): NextState[S] {
  switch (state) {
    case "red":    return "green"  as NextState[S];
    case "green":  return "yellow" as NextState[S];
    case "yellow": return "red"    as NextState[S];
    default:       return assertNever(state as never);
  }
}

// Type-safe inference
const next1 = transition("red");    // type: "green"
const next2 = transition("green");  // type: "yellow"
const next3 = transition("yellow"); // type: "red"

// Compile-time error if invalid input
// transition("blue"); // ❌

// Cycle runner
function runCycle(start: TrafficLight, steps: number): TrafficLight[] {
  const result: TrafficLight[] = [start];
  let current: TrafficLight = start;
  for (let i = 0; i < steps; i++) {
    current = transition(current);
    result.push(current);
  }
  return result;
}

console.log(runCycle("red", 5));
// ["red", "green", "yellow", "red", "green", "yellow"]

// Multi-step transition with chaining
type StepsFrom<S extends TrafficLight, N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc[number] : StepsFrom<NextState[S], N, [...Acc, S]>;

// Real-world: state machine library pattern
type StateMachine<States extends string, Transitions extends Record<States, States>> = {
  initial: States;
  transitions: Transitions;
};

const trafficSM: StateMachine<TrafficLight, NextState> = {
  initial: "red",
  transitions: { red: "green", green: "yellow", yellow: "red" },
};
```

### Edge Cases

- **Generic with literal**: `S extends "red"` constraint — only "red" input, "green" output.
- **`as NextState[S]`**: assertion kerak — `case "red"` da TypeScript `S` ni "red" deb narrow qila olmaydi (literal constraint subtle).
- **Empty transition**: terminal state ("delivered", "cancelled") — `never` qaytaradigan tip.

### Follow-up savollar

1. **"XState kabi library qanday tipni hosil qiladi?"** — Generic + conditional type'lar, `infer` bilan transition map'ini parse qilish.
2. **"`as never` xavfsizmi?"** — Compiler'ga ishonadi — runtime tekshiruv yo'q. Faqat exhaustive guarantee bilan.

</details>

---

### Savol 12: API response handler discriminated union [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ApiResult<T>` discriminated union — `success`, `error`, `loading` variant'lari. Type guard funksiyalar va exhaustive switch bilan handle qilish.

### To'liq tushuntirish

API natijasi tabiatan uchta state'dan birortasi: loading (data hali yo'q), success (data bor + metadata), error (error message + code). Discriminated union `status` discriminant bilan har state uchun specific payload tipi. Generic `<T, E = string>` — har domain uchun reusable (User, Product, Order), error tipi default `string`. Custom type guard'lar (`isSuccess<T>(r): r is {...}`) `filter`/`find` bilan ishlatish uchun — array of results'ni success'lar bilan filterlash, type-safe. Pattern matching helper (`match`) — har case uchun handler qabul qiladi, exhaustive guarantee bilan return value type-safe (React Query, SWR API'lari shu pattern asosida).

### Kod misol

```typescript
type ApiResult<T, E = string> =
  | { status: "loading" }
  | { status: "success"; data: T; timestamp: number }
  | { status: "error"; error: E; code: number };

function assertNever(value: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(value)}`);
}

// Type guards
function isLoading<T, E>(r: ApiResult<T, E>): r is { status: "loading" } {
  return r.status === "loading";
}
function isSuccess<T, E>(r: ApiResult<T, E>): r is { status: "success"; data: T; timestamp: number } {
  return r.status === "success";
}
function isError<T, E>(r: ApiResult<T, E>): r is { status: "error"; error: E; code: number } {
  return r.status === "error";
}

// Pattern matching helper
function match<T, E, R>(
  result: ApiResult<T, E>,
  handlers: {
    loading: () => R;
    success: (data: T, timestamp: number) => R;
    error: (error: E, code: number) => R;
  }
): R {
  switch (result.status) {
    case "loading": return handlers.loading();
    case "success": return handlers.success(result.data, result.timestamp);
    case "error":   return handlers.error(result.error, result.code);
    default:        return assertNever(result);
  }
}

// Usage — User fetch
interface User { id: number; name: string }

const userResult: ApiResult<User> = {
  status: "success",
  data: { id: 1, name: "Aziz" },
  timestamp: Date.now(),
};

const message = match(userResult, {
  loading: () => "Loading user...",
  success: (user) => `User: ${user.name} (#${user.id})`,
  error:   (err, code) => `Error ${code}: ${err}`,
});

// Filter array of results
const results: ApiResult<User>[] = [
  { status: "success", data: { id: 1, name: "Aziz" }, timestamp: 1 },
  { status: "loading" },
  { status: "error", error: "Not found", code: 404 },
  { status: "success", data: { id: 2, name: "Vali" }, timestamp: 2 },
];

const successfulUsers = results
  .filter(isSuccess)
  .map(r => r.data); // User[]
```

### Edge Cases

- **`E = string` default**: error tipi optional, string'ga default.
- **Filter type narrowing (TS 5.5+)**: TypeScript inferred predicate'ni avtomatik tushunadi.
- **Optional discriminator**: agar `status` optional bo'lsa — narrowing ishlamaydi.

### Follow-up savollar

1. **"`Promise` bilan qanday integratsiya?"** — `Promise<ApiResult<T>>` — resolved value har doim ApiResult tipida.
2. **"React Query/SWR'da shu pattern qanday?"** — `useQuery` `{ data, error, isLoading, status }` discriminated union shaklida result qaytaradi.

</details>

---

## Bug fix savollar

### Savol 13: Union narrowing — `in` operator xato [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`breed` ikkalasi (Dog, Cat) da bor — `in` narrowing ishlamaydi. Farqlovchi property kerak.

### To'liq tushuntirish

`in` operator narrowing — property union member'larining qaysi birida mavjudligini aniqlash orqali tipni torytiradi. Agar property barcha member'larda bor bo'lsa, narrowing hech qanday tipni eliminate qila olmaydi — natijada original union saqlanadi va specific method ga kirish error beradi. To'g'ri pattern: (1) `in` faqat farqlovchi property bilan — har member'da unique property bo'lsa, (2) discriminated union (`kind` literal property) — eng aniq va exhaustive check'ga moslashgan, (3) custom type guard (`pet is Dog`) — runtime logic'ni alohida funksiyaga ajratadi.

### Kod misol

```typescript
// ❌ ORIGINAL — narrowing ishlamaydi
type Dog = { breed: string; bark(): void };
type Cat = { breed: string; meow(): void };
type Pet = Dog | Cat;

function makeSound(pet: Pet): void {
  if (pet.breed) {
    pet.bark(); // ❌ Property 'bark' does not exist on type 'Pet'
  }
}

// ✅ Yechim 1 — farqlovchi property bilan `in`
function makeSoundV1(pet: Pet): void {
  if ("bark" in pet) {
    pet.bark();   // pet: Dog
  } else {
    pet.meow();   // pet: Cat
  }
}

// ✅ Yechim 2 — discriminated union (preferred)
type DogV2 = { kind: "dog"; breed: string; bark(): void };
type CatV2 = { kind: "cat"; breed: string; meow(): void };
type PetV2 = DogV2 | CatV2;

function makeSoundV2(pet: PetV2): void {
  switch (pet.kind) {
    case "dog": pet.bark(); break;
    case "cat": pet.meow(); break;
  }
}

// ✅ Yechim 3 — custom type guard
function isDog(pet: Pet): pet is Dog {
  return "bark" in pet;
}

function makeSoundV3(pet: Pet): void {
  if (isDog(pet)) {
    pet.bark();
  } else {
    pet.meow();
  }
}
```

### Edge Cases

- **Both methods optional**: `bark?:` va `meow?:` bo'lsa — `in` `false` qaytarishi mumkin.
- **Inherited property**: prototype chain'da bo'lsa — `in` `true` qaytaradi.
- **Method vs property function**: `bark()` method'mi yoki `bark: () => void` property'mi — `in` ikkalasini bir xil ko'radi.

### Follow-up savollar

1. **"Discriminated union vs `in` qaysi yaxshi?"** — Discriminated — yangi member'lar uchun yaxshi, exhaustive check oson. `in` — eski API'lar uchun.
2. **"Class hierarchy'da qanday qilinadi?"** — `instanceof` bilan, lekin nominal — JSON serialize'ga mos kelmaydi.

</details>

---

### Savol 14: Intersection conflict — yashirin `never` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`A & B` bir xil property turli tip bo'lsa — silent `never`. Property aniq aniqlanmasdan object yaratib bo'lmaydi.

### To'liq tushuntirish

Intersection `A & B` har property uchun ikkala tipning intersection'ini hisoblaydi (`number & string = never`). Compiler bu paytda error bermaydi — type lazy resolution. Faqat object yaratish vaqti — `never` ga assignable hech qanday qiymat yo'q — yangi error yuzaga chiqadi. Bu silent fail'ni avtomatik aniqlash uchun utility tip yozish mumkin: `FindNever<T>` mapped type — har property `never` ekanligini tekshiradi va kalit nomini ekstrakt qiladi (`[K in keyof T]: T[K] extends never ? K : never`). Yechimlar: `Omit` bilan conflict property'ni olib tashlash, yoki overlap property'ni union'ga kengaytirish (`id: string | number`).

### Kod misol

```typescript
// ❌ ORIGINAL — yashirin xato
type A = { id: number; name: string };
type B = { id: string; email: string };
type AB = A & B;

// AB tipining tarkibi:
// {
//   id: number & string = never,
//   name: string,
//   email: string,
// }

// const merged: AB = { id: ???, name: "x", email: "y@x" };
// id uchun hech qanday qiymat mos kelmaydi

// ✅ Yechim 1 — bir xil property'ni Omit qilish
type FixedV1 = Omit<A, "id"> & B;
const m1: FixedV1 = { id: "1", name: "Aziz", email: "a@x.com" }; // ✅

// ✅ Yechim 2 — explicit cast yoki convert
type FixedV2 = A & Omit<B, "id">;
const m2: FixedV2 = { id: 1, name: "Aziz", email: "a@x.com" }; // ✅

// ✅ Yechim 3 — overlap property union qilish
type FixedV3 = Omit<A & B, "id"> & { id: string | number };
const m3: FixedV3 = { id: 1, name: "Aziz", email: "a@x.com" }; // ✅

// Detection utility — never property'larni topish
type FindNever<T> = {
  [K in keyof T]: T[K] extends never ? K : never;
}[keyof T];

type NeverKeys = FindNever<AB>; // "id"
// Compile-time'da conflict'ni topish mumkin
```

### Edge Cases

- **Nested intersection**: `{ a: { b: string } } & { a: { b: number } }` — `a.b` `never`.
- **Generic intersection**: generic parametr'larda runtime'da paydo bo'ladi.
- **Distributive**: `(A | B) & C` = `(A & C) | (B & C)`.

### Follow-up savollar

1. **"Bu xatoni qanday avtomatik topish mumkin?"** — `FindNever` utility yoki ESLint rule.
2. **"`extends` o'rniga `&` ishlatish qachon yomon?"** — Aniq shape kerak bo'lganda — conflict darhol ko'rinishi yaxshi.

</details>

---

## Xulosa

- Union (`|`) — OR, qiymat birortasi. Intersection (`&`) — AND, barcha tip'lar.
- Discriminated union — uchta sharti: umumiy property, literal tip, farqli qiymat.
- Exhaustive checking — `assertNever` bilan barcha member'lar handle qilinganligini guarantee qilish.
- `typeof` primitive uchun, `instanceof` class uchun, `in` property mavjudligi uchun.
- Custom type guard `value is T` — `filter`, `find` da type-safe narrowing.
- Function overload — input → output aniq mapping. Conditional type alternative.
- `satisfies` — type validation + inferred type saqlash. `:` annotation widen qiladi.
- Intersection conflict silent `never`, intersection'da bir xil property turli tip'lar bilan — runtime'da `never`.
