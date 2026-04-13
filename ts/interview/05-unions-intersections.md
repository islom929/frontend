# Interview: Union/Intersection Types

> Union types, intersection types, discriminated unions, type guards, exhaustive checking bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Union type va Intersection type farqi nima?

<details>
<summary>Javob</summary>

Union (`|`) — "bu **yoki** u", qiymat bir nechta type lardan **birortasi**. Intersection (`&`) — "ham bu, **ham** u", qiymat **barcha** type larning property larini o'z ichiga oladi.

```typescript
// Union — string YOKI number
type ID = string | number;
let id: ID = "abc"; // ✅
id = 42;            // ✅

// Intersection — ikkala type ning BARCHA property lari
type HasName = { name: string };
type HasAge = { age: number };
type Person = HasName & HasAge;

const person: Person = {
  name: "Ali",  // HasName dan
  age: 25,      // HasAge dan — ikkalasi MAJBURIY
};
```

| | Union (`\|`) | Intersection (`&`) |
|---|---|---|
| Mantiq | OR — birortasi | AND — barchasi |
| Object da | member lardan biri | barcha member lar birlashadi |
| Primitive da | qiymat birortasi | odatda `never` |
| Ishlatilish | turli variant lar | type composition |

Set theory: Union qiymatlar to'plamini **kengaytiradi** (ko'proq variant), Intersection type **talablarini** kengaytiradi (ko'proq property kerak).

</details>

### 2. Discriminated union nima? Qanday ishlaydi?

<details>
<summary>Javob</summary>

Discriminated union — union type ning har bir member ida umumiy **discriminant** (tag) property bo'ladi. Bu property **literal type** ga ega va har bir member da farqli qiymat oladi.

Uchta sharti:
1. Har bir member da **umumiy property** bor
2. Bu property **literal type** ga ega
3. TS shu property bo'yicha **narrowing** qila oladi

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // TS biladi: Circle
    case "square":
      return shape.side ** 2;             // TS biladi: Square
  }
}
```

Real-world: Redux actions (`type` field), API responses (`status` field), state machines (`state` field), AST nodes (`nodeType` field).

</details>

### 3. Exhaustive checking nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

Exhaustive checking — union type ning **barcha** member lari handle qilinganligini compile-time da tekshirish. `switch`/`if-else` da barcha holatlar ko'rib chiqilgandan keyin, qolgan type `never` bo'lishi kerak.

```typescript
type Status = "active" | "inactive" | "banned";

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function getLabel(status: Status): string {
  switch (status) {
    case "active": return "Faol";
    case "inactive": return "Nofaol";
    case "banned": return "Bloklangan";
    default: return assertNever(status);
    // Barcha case lar handle qilingan — status: never
  }
}
```

Agar yangi variant qo'shilsa:

```typescript
type Status = "active" | "inactive" | "banned" | "suspended"; // YANGI

// TS xato beradi:
// Argument of type '"suspended"' is not assignable to type 'never'
// Compiler barcha handle joylarni topadi — runtime bug oldini olinadi
```

</details>

### 4. `in` operator bilan type narrowing qanday ishlaydi?

<details>
<summary>Javob</summary>

`in` operator — property mavjudligini tekshirib union type ni toraytiradi. Property faqat ba'zi member larda bo'lsa — TS o'sha member larga narrow qiladi.

```typescript
type Dog = { breed: string; bark(): void };
type Cat = { breed: string; meow(): void };
type Pet = Dog | Cat;

function makeSound(pet: Pet): void {
  if ("bark" in pet) {
    pet.bark();  // TS biladi: Dog
  } else {
    pet.meow();  // TS biladi: Cat
  }
}
```

**Muhim:** Agar property **barcha** member larda bor bo'lsa — narrowing **ishlamaydi**:

```typescript
if (pet.breed) { // ❌ breed ikkalasida bor — narrowing effekti yo'q
  pet.bark();    // ❌ TS hali ham Pet deb biladi
}
```

Shuning uchun **farqlovchi** (distinguishing) property kerak — yoki discriminated union ishlatish.

</details>

### 5. Custom type guard funksiya nima? `isNonNullable` misoli bering.

<details>
<summary>Javob</summary>

Type guard — `value is Type` return type li funksiya. `true` qaytarsa — TS qiymatni shu tipda deb hisoblaydi.

```typescript
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const items: (string | null | undefined)[] = ["hello", null, "world", undefined];

// ❌ Oddiy filter — TS type ni toraytirmaydi
const filtered1 = items.filter(item => item !== null && item !== undefined);
// type: (string | null | undefined)[]

// ✅ Type guard bilan
const filtered2 = items.filter(isNonNullable);
// type: string[]
```

TS 5.5+ da **inferred type predicates** bor — `filter(x => x !== null)` avtomatik to'g'ri type beradi. Lekin eski versiyalarda va murakkab holatlarda custom type guard kerak.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Intersection property conflict — output savoli (Daraja: Junior+)

**Savol:** `C` tipining har bir property sining tipi nima? `C` ga mos object yaratish mumkinmi?

```typescript
type A = { x: number; y: string };
type B = { y: number; z: boolean };
type C = A & B;

const obj: C = ???; // Nima yozish kerak?
```

<details>
<summary>Yechim</summary>

```typescript
type C = {
  x: number;     // A dan
  y: never;      // string & number = never
  z: boolean;    // B dan
};
```

`y` property ikki type da bor, lekin tiplari farq qiladi: `string & number = never`. **Hech qanday qiymat** `never` ga mos kelmaydi — shuning uchun `C` ga mos object yaratib **BO'LMAYDI**.

Bu keng tarqalgan xato — intersection da property nomlari to'g'ri kelsa va tiplari farq qilsa, natija `never` property bo'ladi. Interface `extends` da bu darhol compile error berardi.

</details>

### 2. Union narrowing xato — toping va tuzating (Daraja: Middle)

**Savol:** Bu kodda mantiqiy xato bor. Toping va to'g'ri variantini yozing:

```typescript
type Dog = { breed: string; bark(): void };
type Cat = { breed: string; meow(): void };
type Pet = Dog | Cat;

function makeSound(pet: Pet): void {
  if (pet.breed) {
    pet.bark(); // Bu ishlaydi mi?
  }
}
```

<details>
<summary>Yechim</summary>

**Xato:** `breed` property **ikkalasida ham bor** — shuning uchun narrowing ishlamaydi. `pet.breed` truthy bo'lishi `Dog` ga toraytirmaydi.

```typescript
// ✅ Variant 1: distinguishing property bilan
function makeSound(pet: Pet): void {
  if ("bark" in pet) {
    pet.bark();  // TS biladi: Dog
  } else {
    pet.meow();  // TS biladi: Cat
  }
}

// ✅ Variant 2: discriminated union
type Dog = { type: "dog"; breed: string; bark(): void };
type Cat = { type: "cat"; breed: string; meow(): void };
type Pet = Dog | Cat;

function makeSound(pet: Pet): void {
  switch (pet.type) {
    case "dog": pet.bark(); break;
    case "cat": pet.meow(); break;
  }
}
```

**Qoida:** `in` operator faqat **farqlovchi** property bilan ishlaydi. Agar property barcha member larda bor bo'lsa — discriminated union ishlatish.

</details>

### 3. `in` operator narrowing — output savoli (Daraja: Middle)

**Savol:** Quyidagi kodning output ini ayting:

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
console.log(logEvent({ type: "scroll", position: 100 }));
```

<details>
<summary>Yechim</summary>

```
Click at (10, 20)
Scroll: 100
```

- `"x" in event` — `x` faqat `click` member da bor → TS `click` ga narrow qiladi
- `"key" in event` — `key` faqat `keypress` da bor → TS `keypress` ga narrow qiladi
- Qolgan holatda faqat `scroll` qoladi → TS avtomatik narrow qiladi

Discriminated union da `type` property bo'yicha `switch` ishlatish ko'pincha aniqroq:

```typescript
function logEvent(event: Event): string {
  switch (event.type) {
    case "click": return `Click at (${event.x}, ${event.y})`;
    case "keypress": return `Key: ${event.key}`;
    case "scroll": return `Scroll: ${event.position}`;
  }
}
```

</details>

### 4. Discriminated union + exhaustive switch (Daraja: Middle)

**Savol:** E-commerce order statusi uchun discriminated union yarating. Har bir status o'ziga xos ma'lumotlarga ega bo'lsin. `getOrderMessage` funksiyasi exhaustive check bilan barcha holatlarni handle qilsin:

```typescript
// Order type ni yarating:
// - "pending": items ro'yxati
// - "processing": estimatedTime (daqiqa)
// - "shipped": trackingNumber
// - "delivered": deliveredAt (Date)
// - "cancelled": reason

// getOrderMessage funksiyasini yozing:
// - Har bir status uchun tegishli xabar
// - assertNever bilan exhaustive check
```

<details>
<summary>Yechim</summary>

```typescript
type Order =
  | { status: "pending"; orderId: string; items: string[] }
  | { status: "processing"; orderId: string; estimatedTime: number }
  | { status: "shipped"; orderId: string; trackingNumber: string }
  | { status: "delivered"; orderId: string; deliveredAt: Date }
  | { status: "cancelled"; orderId: string; reason: string };

function assertNever(value: never): never {
  throw new Error(`Unhandled status: ${JSON.stringify(value)}`);
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
```

**Tushuntirish:**

- Har bir `case` da TS faqat tegishli member property lariga ruxsat beradi
- `default` da `order` tipi `never` — barcha case lar qoplangan
- Yangi status qo'shilsa — `assertNever` compile-time xato beradi
- `orderId` barcha member larda umumiy — lekin discriminant emas (literal emas)

</details>

### 5. Exhaustive switch + never — state machine (Daraja: Middle+)

**Savol:** Traffic light state machine yarating. Har bir holatdan faqat ruxsat etilgan holatga o'tish mumkin bo'lsin. Noto'g'ri o'tish compile-time da xato bersin:

```typescript
// TrafficLight type ni yarating:
// - "red" → faqat "green" ga o'tishi mumkin
// - "green" → faqat "yellow" ga
// - "yellow" → faqat "red" ga

// transition funksiyasini yozing:
// transition("red")    → "green"
// transition("green")  → "yellow"
// transition("yellow") → "red"
// Barcha holatlar exhaustive check bilan qoplangan bo'lsin
```

<details>
<summary>Yechim</summary>

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
    case "red": return "green" as NextState[S];
    case "green": return "yellow" as NextState[S];
    case "yellow": return "red" as NextState[S];
    default: return assertNever(state as never);
  }
}

// Type-safe:
const next1 = transition("red");    // type: "green"
const next2 = transition("green");  // type: "yellow"
const next3 = transition("yellow"); // type: "red"

// Sequence:
function runCycle(start: TrafficLight, steps: number): TrafficLight[] {
  const result: TrafficLight[] = [start];
  let current: TrafficLight = start;
  for (let i = 0; i < steps; i++) {
    current = transition(current);
    result.push(current);
  }
  return result;
}

runCycle("red", 5);
// ["red", "green", "yellow", "red", "green", "yellow"]
```

**Tushuntirish:**

- `NextState` mapped type — har bir holatning keyingi holatini belgilaydi
- Generic `S extends TrafficLight` — aniq literal tipni saqlaydi
- `transition("red")` qaytargan qiymat tipi **aniq** `"green"` — `TrafficLight` emas
- `assertNever` — yangi holat qo'shilsa compile-time da ushlaydi

</details>

---

## Xulosa

- Union (`|`) — OR, birortasi. Intersection (`&`) — AND, barchasi
- Discriminated union — umumiy literal property (tag) bo'yicha narrowing
- Exhaustive check — `never` + `assertNever` bilan barcha variant lar qoplangan
- `in` operator — faqat farqlovchi property bilan narrowing ishlaydi
- Custom type guard (`value is T`) — `filter`, `find` kabi method larda type-safe narrowing
- UnionToIntersection — batafsil [interview/12 — Conditional Types](12-conditional-types.md) da

[Asosiy bo'limga qaytish →](../05-unions-intersections.md)
