# Interview: TypeScript Asoslari

> TypeScript nima, compiler pipeline, type erasure, structural typing, strict mode, tsconfig.json va versiya tarixi bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Output prediction savollari](#output-prediction-savollari)
- [Coding challenges](#coding-challenges)
- [Bug fix savollari](#bug-fix-savollari)

---

## Nazariy savollar

### Savol 1: TypeScript nima va JavaScript dan qanday farq qiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript — Microsoft tomonidan 2012-yilda yaratilgan, JavaScript ustiga statik type system qo'shuvchi typed superset. Har qanday valid JavaScript kodi avtomatik TypeScript kodi hisoblanadi. Tiplar compile-time da tekshiriladi, runtime da to'liq o'chiriladi (type erasure).

### To'liq tushuntirish

TypeScript va JavaScript ning fundamental farqi — **type checking qachon bo'lishi**. JavaScript dynamically typed til — tiplar runtime da aniqlanadi (`typeof` operator runtime da ishlaydi). TypeScript statically typed — tiplar compile-time da `tsc` tomonidan tekshiriladi.

Bu farqning amaliy oqibatlari:

1. **Type Safety** — `TypeError: Cannot read property 'x' of undefined` kabi runtime xatolar compile-time da ushlanadi
2. **IDE Tooling** — autocomplete (IntelliSense), go-to-definition, find-all-references, rename symbol — type information tufayli aniq ishlaydi
3. **Refactoring** — funksiya signature o'zgartirilsa, compiler barcha nomuvofiqliklarni darhol ko'rsatadi
4. **Code as Documentation** — type annotation lar funksiya nima qabul qiladi va nima qaytarishini tushuntiradi
5. **Scalability** — katta codebase larda yangi developer larning kodni tushunishi osonlashadi

TypeScript ning ikkita asosiy qiymati: xatolarni runtime dan oldin ushlash va IDE tooling — katta loyihalarda developer productivity ni keskin oshiradi.

### Kod misol

```typescript
// JavaScript — type ma'lumoti yo'q
function greet(name) {
  return "Hello, " + name;
}

greet(42);       // → "Hello, 42" (kutilmagan)
greet();         // → "Hello, undefined" (runtime da)
greet("World");  // → "Hello, World"

// TypeScript — compile-time xato
function greetTs(name: string): string {
  return "Hello, " + name;
}

greetTs(42);
// ❌ Argument of type 'number' is not assignable to parameter of type 'string'

greetTs();
// ❌ Expected 1 arguments, but got 0

greetTs("World"); // ✅ → "Hello, World"
```

### Edge Cases

- TypeScript valid JavaScript ni qabul qiladi, lekin `strict: true` bilan `noImplicitAny` yoqilsa parametrlar uchun type annotation majburiy
- TypeScript faqat compile-time da type tekshiradi — runtime da API javob yoki `JSON.parse` natijasi tekshirilmaydi (`zod`/`valibot` kabi runtime validation kerak)
- TypeScript type system Turing-complete — conditional types va `infer` orqali ixtiyoriy hisoblash mumkin, lekin faqat compile-time da

### Follow-up savollar

1. **"TypeScript runtime overhead bormi?"** — Yo'q. Type erasure tufayli compile qilingan JavaScript da type annotation lar umuman yo'q. Runtime performance bir xil.
2. **"JavaScript loyihani bir kunda TypeScript ga o'tkazsa bo'ladimi?"** — Bosqichma-bosqich migration tavsiya etiladi: `allowJs: true, checkJs: false` bilan boshlash, keyin fayllarni `.js` dan `.ts` ga o'tkazish.

</details>

---

### Savol 2: TypeScript Compiler (tsc) qanday ishlaydi? Pipeline bosqichlarini tushuntiring [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

tsc source code ni JavaScript ga 5 bosqichda aylantiradi: Scanner (lexer) → Parser (AST) → Binder (symbols) → Checker (type checking) → Emitter (JS output + .d.ts + source maps).

### To'liq tushuntirish

tsc ning ikki asosiy vazifasi bor: **type checking** va **emit**. Bu vazifalar mustaqil — `noEmit: true` faqat type check qiladi, esbuild/SWC esa faqat emit qiladi (type check qilmasdan).

Pipeline bosqichlari:

1. **Scanner (Lexer)** — source matnini token larga ajratadi. Whitespace va comment lar `trivia` deb e'tiborga olinmaydi.
2. **Parser** — token oqimidan AST (Abstract Syntax Tree) hosil qiladi. Recursive descent algoritmi. Syntax error recovery — xato uchrasa ham parsing davom etadi (IDE real-time feedback uchun).
3. **Binder** — AST bo'ylab yuradi, har declaration uchun `Symbol` yaratadi, scope tree quradi. Symbol — bitta entity haqidagi barcha ma'lumotni birlashtiradi.
4. **Checker** — tsc ning eng katta qismi. Type inference, assignability check, generic resolution, control flow narrowing, function overload resolution, diagnostic generation. Lazy evaluation ishlatadi — kerak bo'lganda hisoblaydi.
5. **Emitter** — AST dan JavaScript source code hosil qiladi. Type annotation lar o'chiriladi, TS-specific construct'lar (enum, namespace) JS ga aylantiriladi, `target` ga qarab downlevel compile qilinadi (`async/await` → generator agar `target: ES5`).

`noEmitOnError: true` default qiymat — type xato bo'lsa emit qilinmaydi.

### Kod misol

```typescript
// Source:
let age: number = 25;

// Scanner output (tokenlar):
// LetKeyword → Identifier("age") → ColonToken → NumberKeyword
// → EqualsToken → NumericLiteral(25) → SemicolonToken

// Parser output (AST, soddalashtirilgan):
// VariableStatement
//   └── VariableDeclarationList
//         └── VariableDeclaration
//               ├── name: Identifier("age")
//               ├── type: TypeReference("number")  ← TS qo'shimchasi
//               └── initializer: NumericLiteral(25)

// Emitter output:
// let age = 25;  // type annotation o'chirildi
```

TypeScript Compiler API orqali scanner ni ko'rish:

```typescript
import * as ts from "typescript";

const scanner = ts.createScanner(ts.ScriptTarget.Latest, true);
scanner.setText(`let age: number = 25;`);

let token = scanner.scan();
while (token !== ts.SyntaxKind.EndOfFileToken) {
  console.log(ts.SyntaxKind[token], JSON.stringify(scanner.getTokenText()));
  token = scanner.scan();
}
// → LetKeyword "let", Identifier "age", ColonToken ":", ...
```

### Edge Cases

- Parser xato uchrasa ham AST hosil qiladi — incomplete AST, lekin IDE inline diagnostics chiqaradi
- Checker `checker.ts` faylida joylashgan — TypeScript source kodining yarmidan ko'pi
- Emitter source map (`.js.map`) va declaration (`.d.ts`) fayllarni alohida flag bilan hosil qiladi
- esbuild/SWC `tsc` dan tezroq — chunki ular faqat syntax parsing va type erasure qiladi, type check qilmaydi

### Follow-up savollar

1. **"Nima uchun katta loyihada tsc sekin ishlaydi?"** — Checker ning to'liq type resolution barcha fayllarni o'rganadi. Project references (`tsconfig.json` `references`) yoki incremental compilation (`tsc --build`) bu muammoni hal qiladi.
2. **"esbuild type tekshirmasdan compile qiladi-yu, xatosiz qoldirib ketadi-mi?"** — Ha. Production build da `tsc --noEmit` (CI) + esbuild (build) kombinatsiyasi standart pattern.

<details>
<summary><strong>Deep Dive</strong></summary>

`tsc` ning Checker qismi **lazy evaluation** ishlatadi — barcha type larni oldindan hisoblamasdan, kerak bo'lganda hisoblaydi. Bu katta loyihalarda zarur: agar har bir generic instantiation darhol resolve qilinsa, compilation vaqti eksponensial o'sardi.

Checker da type assignability `isTypeAssignableTo(source, target)` funksiyasi orqali tekshiriladi — `Subtype`, `Assignability`, `Identity` aloqalari structural rules bo'yicha. Generic type'lar solishtirilganda type argument lar variance qoidalari (covariance, contravariance, bivariance, invariance) asosida tekshiriladi — bu structural comparison ning bir qismi.

Source map generation `Emitter` da source position information ni TypeScript AST node lariga bog'laydi — debugging vaqtida compiled JS dagi qatordan original TS qatoriga o'tish uchun.

</details>

</details>

---

### Savol 3: Type erasure nima va nima uchun muhim? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type erasure — compile jarayonida barcha type ma'lumotlari (annotation, interface, type alias, generic) butunlay o'chiriladi. Runtime da TypeScript haqida hech qanday iz qolmaydi — faqat sof JavaScript.

### To'liq tushuntirish

TypeScript ning dizayn maqsadi — mavjud JavaScript runtime larini o'zgartirmaslik. Brauzerlar va Node.js faqat JavaScript ni bajaradi. TypeScript yangi runtime yaratish o'rniga, compile-time da foydali bo'lib, keyin "yo'qolib ketadi".

Type erasure ning oqibatlari:

1. **Runtime da type tekshirish yo'q** — `if (value instanceof MyInterface)` ishlamaydi
2. **Runtime performance ta'siri nol** — annotation lar JS ga tushmaydi
3. **JavaScript interop** — compile qilingan TS kodi har qanday JS bilan ishlaydi
4. **Tashqi data validation kerak** — API javob, `JSON.parse`, `localStorage` — bularning hech biri TypeScript nazoratidan o'tmaydi

Erasure qoidalari (qaysi narsalar o'chiriladi):

- Type annotation lar: `: string`, `: number`, `: User`
- Interface va type alias declaration lar
- Generic type parameter lar: `<T>`
- Type assertion: `as Type`, `satisfies Type`
- Non-null assertion: `value!`
- `import type` / `export type`
- `readonly`, `public`, `private`, `protected` modifier lar (compile-time only)

Saqlanadigan TS construct'lar (JS ga compile):

- `enum` → IIFE + reverse mapping
- `namespace` → IIFE
- Decorators → wrapper function
- Constructor parameter properties (`constructor(public x: number)`)

### Kod misol

```typescript
// TypeScript source
interface User {
  name: string;
  age: number;
}

function greet(user: User): string {
  return `Hi ${user.name}`;
}

const u: User = { name: "Ali", age: 25 };
greet(u);

// Compiled JavaScript (type erasure)
// interface User — butunlay yo'q
function greet(user) {
  return `Hi ${user.name}`;
}

const u = { name: "Ali", age: 25 };
greet(u);
```

`instanceof` interface bilan ishlamaydi — type erasure oqibati:

```typescript
interface Cat { meow(): void; }
interface Dog { bark(): void; }

function makeSound(animal: Cat | Dog) {
  // ❌ ISHLAMAYDI — interface runtime da yo'q
  // if (animal instanceof Cat) { ... }
  // Compile error: 'Cat' only refers to a type

  // ✅ Property tekshirish bilan narrowing
  if ("meow" in animal) {
    animal.meow(); // TS narrowing — animal: Cat
  } else {
    animal.bark(); // TS narrowing — animal: Dog
  }
}
```

### Edge Cases

- `private` modifier compile-time only — JavaScript da `instance.privateField` runtime da ochiq qoladi. Haqiqiy runtime private uchun ES2022 `#` private fields kerak
- TypeScript ning `class` runtime da saqlanadi (ES2015 class ga compile bo'ladi) — shuning uchun `instanceof MyClass` ishlaydi
- `const enum` boshqacha — qiymatlar inline qilinadi, IIFE hosil qilmaydi (lekin `isolatedModules` bilan re-export muammosi bor)
- Decorator `experimentalDecorators` legacy va TC39 Stage 3 standart decorator'lar — ikkalasi ham runtime kod hosil qiladi

### Follow-up savollar

1. **"`as Type` bilan compiler ni aldab bo'ladimi?"** — Ha. `as` runtime da o'chiriladi, tekshirilmaydi. Aslida noto'g'ri assertion bo'lsa, runtime da crash bo'ladi. Faqat 100% ishonchli bo'lganda ishlatish kerak.
2. **"Runtime type validation uchun nima ishlatilishi kerak?"** — `zod`, `valibot`, `io-ts`, `yup` — bular schema asosida runtime da data tekshiradi va TypeScript type bilan integratsiyalanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

Type erasure'ni `tsc` ning Emitter qismi amalga oshiradi. Emitter AST bo'ylab yurib, type-related node lar (TypeReference, InterfaceDeclaration, TypeAliasDeclaration, GenericTypeParameter) ni AST dan olib tashlaydi. Visitor pattern ishlatadi — har node turi uchun alohida handler.

TS 5.8 dan boshlab `erasableSyntaxOnly` flag kiritildi — Node.js 22.6+ ning `--experimental-strip-types` flag bilan native TypeScript support uchun. Bu flag enum, namespace, constructor parameter properties, `experimentalDecorators` ni taqiqlaydi (chunki ular faqat type erasure orqali o'chmaydi — runtime kod hosil qiladi). Node.js 23.6+ da `--experimental-strip-types` default yoqilgan.

Babel'ning `@babel/preset-typescript` ham erasure asosida ishlaydi — type check qilmasdan annotation larni o'chirib JS chiqaradi. Bu TypeScript Compiler dan tez, lekin type safety yo'q.

</details>

</details>

---

### Savol 4: Structural typing va nominal typing farqi nima? TypeScript qaysi birini ishlatadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Nominal typing** (Java, C#) — ikki tip mos kelishi uchun bir xil nomga yoki aniq inheritance bog'lanishiga ega bo'lishi kerak. **Structural typing** (TypeScript, Go interfaces) — tuzilma (shape) mos bo'lsa yetarli, nomlar ahamiyatsiz. TypeScript structural typing ishlatadi.

### To'liq tushuntirish

Dasturlash tillarida type compatibility ni aniqlashning ikki fundamental yondashuvi:

**Nominal Typing** (Java, C#, Swift, Rust): nomlar muhim. Tuzilma bir xil bo'lsa ham, nom farq qilsa — tiplar mos kelmaydi.

```java
// Java — Nominal Typing
class UserId { int value; }
class ProductId { int value; }

UserId userId = new ProductId(1); // ❌ Compile error!
```

**Structural Typing** (TypeScript): shape mos bo'lsa yetarli. JavaScript ning duck typing yondashuvini compile-time da rasmiylashtiradi.

TypeScript structural typing ni tanladi chunki JavaScript ning tabiati shu — object ning qaysi class dan yaratilgani emas, qanday property/method lari borligi muhim.

**Structural typing afzalliklari:**
- JavaScript pattern'lari bilan tabiiy mos kelish
- Object literal va class instance ni bir xil qabul qilish
- Refactoring oson — nom o'zgartirilsa, shape o'zgarmaganda kod buzilmaydi

**Kamchiliklar:**
- Mantiqan farqli tiplar (UserId vs ProductId — ikkalasi `number`) aralashib ketishi mumkin
- Yechim: **branded types** pattern

### Kod misol

```typescript
interface Point {
  x: number;
  y: number;
}

class Coordinate {
  constructor(public x: number, public y: number) {}
}

function logPoint(point: Point): void {
  console.log(`(${point.x}, ${point.y})`);
}

const coord = new Coordinate(10, 20);
logPoint(coord); // ✅ Coordinate Point ni implement qilmagan, lekin shape mos
logPoint({ x: 5, y: 15 }); // ✅ Object literal ham ishlaydi
```

Structural typing pitfall — branded types yechimi:

```typescript
// ❌ Structural typing UserId va ProductId ni aralashtiradi
type UserId = number;
type ProductId = number;

function getUser(id: UserId) { /* ... */ }

const pid: ProductId = 42;
getUser(pid); // ✅ TS xato bermaydi — ikkalasi number

// ✅ Branded types — nominal-like behavior
type Brand<T, B> = T & { readonly __brand: B };
type UserIdBranded = Brand<number, "UserId">;
type ProductIdBranded = Brand<number, "ProductId">;

function getUserBranded(id: UserIdBranded) { /* ... */ }

const pidBranded = 42 as ProductIdBranded;
getUserBranded(pidBranded); // ❌ ProductId UserId ga mos kelmaydi
```

### Edge Cases

- **Excess property checking** — object literal to'g'ridan-to'g'ri berilganda TS qo'shimcha property larni tekshiradi. Variable orqali berilsa — tekshirmaydi (structural typing default xulqi)
- `class A implements I` — `implements` o'zi structural check qiladi, lekin nominal aloqa hosil qilmaydi
- Empty type `{}` — barcha qiymatlarga mos keladi (`null` va `undefined` dan tashqari) — structural typing ning chekka holat
- `class` private field (`#` ES2022) — runtime da haqiqiy private, lekin TypeScript `private` modifier faqat compile-time

### Follow-up savollar

1. **"Excess property checking nima va qachon o'tkazib yuboriladi?"** — Object literal funksiya argumenti yoki variable annotation ga to'g'ridan-to'g'ri berilganda TS qo'shimcha property larni xato deb qaraydi. Variable orqali berilsa tekshirilmaydi. `as` assertion ham buni o'tkazib yuboradi.
2. **"Nima uchun TypeScript nominal typing ni qo'shmagan?"** — JavaScript ning duck typing tabiati bilan mos kelmaydi. Branded types va opaque types pattern lari nominal-like behavior berishga imkon yaratadi.

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript ning structural compatibility check'i `checker.ts` da `isRelatedTo()` funksiyasida implement qilingan. U `Assignable`, `Subtype`, `Identity` aloqalarini hisoblaydi.

Algoritm rekursiv: A type B ga assignable bo'lishi uchun B ning har bir required property A da bo'lishi va shu property lar uchun B[K] A[K] ga assignable bo'lishi kerak. Optional property lar mavjud bo'lmasligi mumkin.

Variance qoidalari structural check ning bir qismi:
- **Covariance** — `readonly Array<Dog>` `readonly Array<Animal>` ga assignable
- **Contravariance** — `(x: Animal) => void` `(x: Dog) => void` ga assignable (`strictFunctionTypes` bilan)
- **Bivariance** — method syntax `(x: Animal) => void` ikki tomonga ham mos (backward compat)
- **Invariance** — `Box<Dog>` `Box<Animal>` ga assignable emas (mutable)

TS 4.7 `in`/`out` variance annotation lar — generic type parameter uchun variance ni aniq belgilash imkonini berdi.

</details>

</details>

---

### Savol 5: `strict: true` nima qiladi? Qaysi flag larni yoqadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`strict: true` — tsconfig.json da barcha strict type-checking flag larni birdan yoqadigan meta-flag. Yangi loyihada doim yoqish kerak. 8 ta flag ni yoqadi: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, `useUnknownInCatchVariables`.

### To'liq tushuntirish

Har flag ning ta'siri:

| Flag | Ta'sir |
|------|--------|
| `strictNullChecks` | `null` va `undefined` har tipdan ajratiladi — `string` ga `null` assign mumkin emas, faqat `string \| null` ga |
| `noImplicitAny` | Tip aniqlanmasa `any` deb taxmin qilmaslik — aniq annotation kerak |
| `strictFunctionTypes` | Function **type syntax** (`(x: Dog) => void`) parametrlari contravariant tekshiriladi. Method syntax bivariant qoladi (backward compat) |
| `strictBindCallApply` | `.bind()`, `.call()`, `.apply()` parametrlarini to'g'ri tekshiradi |
| `strictPropertyInitialization` | Class property lar constructor da initialize qilinishi shart |
| `noImplicitThis` | `this` type noma'lum bo'lsa xato |
| `alwaysStrict` | Faylni strict mode da parse qiladi va output ga `"use strict"` qo'shadi. ES module output uchun pragma semantik jihatdan ortiqcha — ESM spec bo'yicha har doim strict mode |
| `useUnknownInCatchVariables` | `catch(e)` da `e` ning type `unknown` (TS 4.4 da `strict` ostiga qo'shildi) |

`strict: true` yozib, alohida flag ni `false` qilish mumkin — migration paytida foydali:

```json
{
  "compilerOptions": {
    "strict": true,
    "strictPropertyInitialization": false
  }
}
```

### Kod misol

`strictNullChecks` ta'siri:

```typescript
// strictNullChecks: false (xavfli)
function getLength(str: string): number {
  return str.length; // str null bo'lsa → runtime crash
}
getLength(null); // ✅ Compile da xato yo'q, runtime da TypeError

// strictNullChecks: true (xavfsiz)
function getLengthSafe(str: string | null): number {
  if (str === null) return 0; // ✅ null check majburiy
  return str.length;
}
getLengthSafe(null); // ✅ → 0
```

`noImplicitAny` ta'siri:

```typescript
// ❌ Parameter 'data' implicitly has an 'any' type
function process(data) {
  return data.name;
}

// ✅ Aniq type
function processTyped(data: { name: string }): string {
  return data.name;
}
```

`useUnknownInCatchVariables` ta'siri:

```typescript
try {
  riskyOperation();
} catch (error) {
  // error: unknown — narrowing kerak
  if (error instanceof Error) {
    console.log(error.message); // ✅
  }
}
```

### Edge Cases

- `strict: false` bilan boshlangan eski loyihada `strict: true` yoqilsa — yuzlab xato bir zumda. Migration bosqichma-bosqich: `strictNullChecks: true` avval, keyin `noImplicitAny`, va hokazo
- `strictPropertyInitialization` — abstract class va `declare` keyword bilan property uchun init talab qilinmaydi
- `useUnknownInCatchVariables: false` — TS 4.4 dan oldingi xulq (`catch(e)` da `e: any`)
- ECMAScript module fayllar (`.mjs`/`.mts` yoki `"type": "module"` paketda) spec bo'yicha har doim strict mode — `alwaysStrict` bu yerda semantik o'zgarish bermaydi (emit qilingan `"use strict"` ortiqcha)

### Follow-up savollar

1. **"`strict: true` ni boshqa flag lar bilan kuchaytirish mumkinmi?"** — Ha: `noUncheckedIndexedAccess` (array index — `T | undefined`), `exactOptionalPropertyTypes` (`?` va `| undefined` farqlanadi), `noImplicitOverride`, `noFallthroughCasesInSwitch`. Bular `strict` ichida emas, alohida yoqilishi kerak.
2. **"`strictFunctionTypes` faqat function syntax uchun ishlaydi-mi?"** — Ha. Method syntax (`handle(x: Dog): void`) bivariant qoladi backward compat uchun. Bu nuance asosan event handler va callback signature larida ahamiyatli.

<details>
<summary><strong>Deep Dive</strong></summary>

`strict` flag lar TypeScript versiyalari bo'yicha qo'shilgan:

- **TS 2.0** — `strictNullChecks`, `noImplicitAny`
- **TS 2.3** — `strict` meta-flag kiritildi
- **TS 2.6** — `strictFunctionTypes`
- **TS 2.7** — `strictPropertyInitialization`, `strictBindCallApply`
- **TS 4.2** — `noPropertyAccessFromIndexSignature` (alohida, `strict` ostida emas)
- **TS 4.4** — `useUnknownInCatchVariables` `strict` ostiga qo'shildi, `exactOptionalPropertyTypes` kiritildi (alohida, `strict` ostida emas)

`strictFunctionTypes` ning contravariance check'i `checker.ts` da `compareSignaturesRelated()` funksiyasida amalga oshiriladi. Method syntax (`{ handle(x: Dog): void }`) bivariant bo'lib qoldirilgan chunki TypeScript 1.x da DOM library va array method signature lar shu shaklda yozilgan — strictness qo'shilsa, mavjud kodning yarmi buzilgan bo'lardi.

</details>

</details>

---

### Savol 6: `any` va `unknown` farqi nima? Qachon qaysi birini ishlatish kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`any` type checking ni butunlay o'chiradi — har qanday operatsiya ruxsat etiladi, type safety yo'q. `unknown` — "tip noma'lum, avval tekshir" — narrowing siz hech qanday operatsiya mumkin emas. Yangi kodda doim `unknown` ishlatish kerak.

### To'liq tushuntirish

| Xususiyat | `any` | `unknown` |
|-----------|-------|-----------|
| Boshqa tipga assign | ✅ Ruxsat | ❌ Faqat `any`/`unknown` ga |
| Property/method call | ✅ Tekshirilmaydi | ❌ Narrowing kerak |
| Type safety | ❌ Yo'q | ✅ Bor |
| Asosiy use case | JS → TS migration | API boundary, `catch(e)`, tashqi data |

`any` TypeScript ning "escape hatch" — type system ni chetlab o'tish uchun. Tarixda JavaScript loyihalarni asta-sekin TypeScript ga o'tkazish uchun kiritilgan. Lekin har joyda ishlatilsa, TypeScript ning eng katta foydasi yo'qoladi.

`unknown` TS 3.0 da qo'shilgan — `any` ning xavfsiz alternativasi. Boshqa tipga assign qilish mumkin emas, faqat narrowing dan keyin operatsiya qilinadi.

### Kod misol

`any` — xavfli:

```typescript
function processAny(data: any) {
  data.user.profile.getName(); // ✅ Compile da OK
  // Runtime: TypeError: Cannot read property 'user' of undefined
}

processAny(null); // 💥 Runtime crash
```

`unknown` — xavfsiz:

```typescript
function processUnknown(data: unknown) {
  // data.name; ❌ 'data' is of type 'unknown'

  if (typeof data === "object" && data !== null && "name" in data) {
    console.log(data.name); // ✅ narrowing dan keyin
  }
}
```

`catch` blokda — `useUnknownInCatchVariables` yoqilganda:

```typescript
try {
  await fetchUser();
} catch (error) {
  // error: unknown (TS 4.4+)

  if (error instanceof Error) {
    console.log(error.message); // ✅
  } else {
    console.log("Unknown error:", String(error));
  }
}
```

### Edge Cases

- `any` "viral" — `any` bilan ishlash natijasi ham `any` bo'ladi (`anyValue.property.method` → `any`)
- `unknown` "non-viral" — uni narrowing qilmaguncha hech qanday operatsiya qilib bo'lmaydi
- `unknown` ni union ga qo'shilsa, butun union `unknown` ga aylanadi (`unknown | string` → `unknown`)
- `any` `never` dan tashqari har tipga assignable, `unknown` faqat `any` va `unknown` ga
- TypeScript ning ba'zi API lari (masalan `JSON.parse`, `fetch().then(r => r.json())`) `any` qaytaradi — manual `unknown` cast tavsiya etiladi

### Follow-up savollar

1. **"`unknown` dan keyin doim manual narrowing kerakmi? Soddaroq yo'l bormi?"** — `zod` yoki `valibot` kabi schema validator lar `unknown` dan typed object ga conversion qiladi: `const user = UserSchema.parse(data)` → `user: User`. Bu manual `typeof` check lardan ko'ra deklarativ.
2. **"`any` ni qachon ishlatish kerak?"** — Migration paytida vaqtincha, third-party library uchun type berishda (agar `@types` bo'lmasa), yoki test mock larda. Boshqa hamma joyda `unknown` yoki aniq type.

<details>
<summary><strong>Deep Dive</strong></summary>

`unknown` ni implementation qilish — TypeScript 3.0 ning eng murakkab type system yangiligi edi. `unknown` "top type" deb ataladi — barcha tiplar `unknown` ga assignable, lekin `unknown` faqat `any` va `unknown` ga assignable. Type theory tilida: `T <: unknown` har T uchun.

`any` ham top type, ham bottom type kabi xulq qiladi — `any` `never` dan tashqari har tipga assignable, va har tip `any` ga assignable. Bu "noisy" property type system'ning soundness'ini buzadi. `unknown` aynan shu muammoni hal qilish uchun kiritildi.

Narrowing `unknown` da `typeof`, `instanceof`, `in` operator, type predicate (`x is T`) orqali ishlaydi. Checker `getNarrowedType()` funksiyasida control flow analysis bo'yicha union type larni filter qiladi.

`useUnknownInCatchVariables` (TS 4.4) kiritilishi sababi — JavaScript da `throw "string"` yoki `throw 42` qonuniy, shuning uchun `e: any` semantically noto'g'ri edi. `unknown` to'g'riroq — har throw qiymati `unknown` deb qaraladi va narrowing kerak.

</details>

</details>

---

### Savol 7: `.ts`, `.tsx`, `.d.ts`, `.mts`, `.cts` fayl kengaytmalari farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`.ts` — standart TypeScript fayl, `.js` ga compile. `.tsx` — JSX support (React), `.js` ga compile, angle-bracket assertion ishlamaydi. `.d.ts` — faqat type declaration, kod yo'q, compile qilinmaydi. `.mts` — ESM module, `.mjs` ga compile. `.cts` — CommonJS module, `.cjs` ga compile.

### To'liq tushuntirish

| Kengaytma | Maqsad | Compile natijasi |
|-----------|--------|------------------|
| `.ts` | Standart TypeScript source | `.js` |
| `.tsx` | TypeScript + JSX (React) | `.js` |
| `.d.ts` | Declaration fayl (type-only) | Compile qilinmaydi |
| `.mts` | TypeScript ES Module | `.mjs` |
| `.cts` | TypeScript CommonJS | `.cjs` |

**`.tsx` muhim cheklov** — angle-bracket type assertion ishlamaydi:

```typescript
// .ts faylda — ikkalasi ishlaydi
const value1 = <string>someValue;
const value2 = someValue as string;

// .tsx faylda — faqat `as` ishlaydi
// const value = <string>someValue; ❌ JSX deb tushunadi
const value = someValue as string; // ✅
```

**`.d.ts` ishlatilishi:**
- JavaScript kutubxonalar uchun type information (`@types/node`, `@types/lodash`)
- Compile qilingan TypeScript library ning type larini export qilish
- Global ambient declaration (`declare global { ... }`, `declare module "*.png"`)

**`.mts` va `.cts`** — Node.js ESM/CJS module formatini aniq belgilash uchun. `moduleResolution: "Node16"` yoki `"NodeNext"` bilan ishlatiladi. Bir loyihada `.mts` va `.cts` aralashtirib ishlatish mumkin.

### Kod misol

`.d.ts` — type declaration:

```typescript
// types/global.d.ts
declare module "*.png" {
  const src: string;
  export default src;
}

declare global {
  interface Window {
    analytics: { track: (event: string) => void };
  }
}

// app.ts da
import logo from "./logo.png"; // ✅ type: string
window.analytics.track("page_view"); // ✅
```

`.mts` vs `.cts` Node.js da:

```typescript
// server.mts → server.mjs (ESM)
import { readFile } from "node:fs/promises";
export async function loadConfig() { /* ... */ }

// config.cts → config.cjs (CommonJS)
const path = require("node:path");
module.exports = { configPath: path.join(__dirname, "config") };
```

`.tsx` JSX bilan:

```typescript
// Button.tsx
import { useState } from "react";

interface Props {
  label: string;
  onClick: () => void;
}

export function Button({ label, onClick }: Props) {
  return <button onClick={onClick}>{label}</button>;
}
```

### Edge Cases

- `.d.ts` fayl `tsc` tomonidan emit qilinmaydi, lekin `declaration: true` bilan TypeScript library lar `.d.ts` fayllarini avtomatik generate qiladi
- `.mts` import qilinganda extension yozilishi shart: `import { x } from "./module.mjs"` (Node.js ESM qoidasi)
- `.tsx` faqat `jsx` tsconfig option o'rnatilganda compile bo'ladi (`"jsx": "react-jsx"` zamonaviy default)
- `.d.cts` va `.d.mts` ham mavjud — alohida CJS va ESM uchun declaration fayllar

### Follow-up savollar

1. **"Bir paketda `.mts` va `.cts` aralashtirib ishlatish mumkinmi?"** — Ha, lekin ehtiyot bo'lish kerak. ESM moduldan CJS modulni `import` qilish mumkin, lekin teskari — faqat `dynamic import()` orqali. `package.json` "exports" field bilan to'g'ri rout qilish kerak.
2. **"`.d.ts` fayl qo'lda yozilishi shart-mi?"** — Yo'q. TypeScript library yozsangiz, `declaration: true` bilan `.d.ts` avtomatik generate qilinadi. Faqat third-party JS library uchun yoki ambient global type lar uchun qo'lda yozish kerak.

</details>

---

### Savol 8: tsconfig.json ning asosiy bo'limlari va eng muhim flag lari qaysilar? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

tsconfig.json TypeScript loyihasining konfiguratsiya fayli. Asosiy bo'limlari: `compilerOptions` (compile sozlamalari), `include`/`exclude`/`files` (qaysi fayllar), `extends` (meros olish), `references` (project references). Eng muhim flag lar: `target`, `module`, `strict`, `outDir`, `esModuleInterop`, `skipLibCheck`, `isolatedModules`.

### To'liq tushuntirish

Asosiy bo'limlar:

1. **`compilerOptions`** — compiler sozlamalari (target, module, strict, va yuzlab boshqa flag)
2. **`include`** — qaysi fayllar compile qilinadi (glob: `src/**/*.ts`)
3. **`exclude`** — qaysi fayllar tashlab ketiladi (`node_modules`, `dist`)
4. **`files`** — aniq fayl ro'yxati (alternativ `include` ga)
5. **`extends`** — boshqa tsconfig dan meros olish (`@tsconfig/node20`, `@tsconfig/strictest`)
6. **`references`** — project references (monorepo)

Eng muhim compilerOptions:

| Flag | Maqsad |
|------|--------|
| `target` | Output JavaScript versiyasi (`ES2022`, `ESNext`) |
| `module` | Module system (`NodeNext`, `ESNext`, `CommonJS`) |
| `moduleResolution` | Module qidirish strategiyasi (`NodeNext`, `bundler`) |
| `strict` | Strict type checking meta-flag |
| `outDir` | Compile natijalari papkasi (`./dist`) |
| `rootDir` | Source fayllar papkasi (`./src`) |
| `esModuleInterop` | CJS/ESM interop ni soddalashtirish (default import) |
| `skipLibCheck` | `.d.ts` fayllarni tekshirmasdan o'tkazish (performance) |
| `isolatedModules` | Babel/esbuild bilan moslik (per-file compile) |
| `noUncheckedIndexedAccess` | Array/object index — `T \| undefined` |
| `forceConsistentCasingInFileNames` | Fayl nomi case sensitivity |

### Kod misol

Production-ready tsconfig.json (Node.js):

```jsonc
{
  "compilerOptions": {
    // Target va Module
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",

    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "sourceMap": true,

    // Type Checking
    "strict": true,
    "noUncheckedIndexedAccess": true,

    // Interop
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "isolatedModules": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

React (Vite) — qo'shimcha sozlamalar:

```jsonc
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"]
}
```

`extends` orqali base config:

```jsonc
// tsconfig.json
{
  "extends": "@tsconfig/node20/tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist"
  }
}
```

### Edge Cases

- `tsc` argumentsiz ishga tushirilganda joriy papka va ota papkalarda `tsconfig.json` ni qidiradi
- `tsc hello.ts` (fayl bilan) `tsconfig.json` ni **o'qimaydi** — faqat default sozlamalar
- `extends` array sifatida ham bo'lishi mumkin (TS 5.0+) — bir nechta tsconfig dan meros olish
- `references` monorepo da incremental build uchun — bog'liq loyihalar `tsc --build` bilan to'g'ri tartibda compile bo'ladi
- `isolatedModules: true` ba'zi TypeScript feature larni cheklaydi: `const enum` export, namespace value re-export — Babel/esbuild bilan moslik uchun

### Follow-up savollar

1. **"`extends` bilan boshqa loyihadagi tsconfig dan meros olsa, qaysi field lar override bo'ladi?"** — `compilerOptions` ichidagi field lar shallow merge bo'ladi (yangi qiymat eski'ni almashtiradi). `include`, `exclude`, `files` to'liq override bo'ladi (merge emas). `references` ham override.
2. **"`noEmit: true` bilan tsc nima qiladi?"** — Faqat type check qiladi, JavaScript chiqarmaydi. CI da type check uchun ishlatiladi: `tsc --noEmit` keyin esbuild/SWC bilan production build.

<details>
<summary><strong>Deep Dive</strong></summary>

`moduleResolution` strategiyalari:

- **`classic`** — eski, faqat backward compat uchun
- **`node10`** (avval `node`) — Node.js CJS resolution
- **`node16`/`nodenext`** — Node.js ESM resolution, `.mjs`/`.cjs` farqlash, `package.json` "exports" field
- **`bundler`** (TS 5.0+) — Vite/esbuild/Webpack bundler lar uchun, extension talab qilmaydi, "exports" field qo'llab-quvvatlaydi

`module` flag `moduleResolution` bilan birga ishlatiladi:
- `module: "NodeNext"` → `moduleResolution: "NodeNext"` (default)
- `module: "ESNext"` → bundler bilan
- `module: "CommonJS"` → faqat CJS output

`isolatedModules` flag Babel va esbuild ning per-file compilation cheklovini majburlaydi. Bu flag yoqilganda:
- `const enum` export qilib bo'lmaydi (chunki inline qilish butun loyihani ko'rish kerak)
- `import type` ishlatish kerak — type-only import lar JS ga tushmasligi uchun
- Namespace value re-export taqiqlangan

TS 5.5 ning `isolatedDeclarations` flag — `.d.ts` generation ni faylga local qiladi. Library yozuvchilar uchun parallel type emission.

</details>

</details>

---

### Savol 9: TypeScript Playground nima va qachon ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript Playground — brauzerda TypeScript kodni yozib, real-time compile natijasini ko'rish imkonini beradigan rasmiy online muhit (typescriptlang.org/play). VS Code ning Monaco Editor + browserda ishlovchi `tsc` Web Worker dan iborat.

### To'liq tushuntirish

Playground imkoniyatlari:

1. **TS → JS real-time** — chap tomonda source, o'ng tomonda compiled output
2. **Compiler Options UI** — `tsconfig.json` sozlamalarini click bilan o'zgartirish
3. **Errors panel** — type xatolar inline va pastda
4. **Type hover** — cursor ni o'zgaruvchi ustiga qo'yganda inferred type
5. **Share URL** — kod URL ga encode bo'ladi, ulashish mumkin
6. **TypeScript version selector** — har xil versiyalarda sinab ko'rish (1.x dan zamonaviy 5.x gacha)
7. **AST viewer plugin** — kodning AST tuzilmasini ko'rish
8. **Examples library** — TypeScript ning har bir feature i uchun tayyor misol

Ishlatish use case lari:
- Yangi TS feature ni o'rnatishsiz sinab ko'rish
- Type xato sababini tushunish (compile natijasi orqali)
- `target` flag ta'sirini ko'rish (ES5 vs ES2022)
- StackOverflow/GitHub Issues da TS muammosini ulashish
- Compile output ni o'rganish (enum, decorator, async/await)

### Kod misol

Playground da enum compile natijasini ko'rish — type erasure va runtime kod orasidagi farq:

```typescript
// Source (chap tomon)
enum Direction {
  Up,
  Down,
  Left,
  Right
}

const dir: Direction = Direction.Up;
console.log(Direction[dir]); // → "Up" (reverse mapping)
```

Compile output (o'ng tomon):

```javascript
// IIFE + reverse mapping
var Direction;
(function (Direction) {
    Direction[Direction["Up"] = 0] = "Up";
    Direction[Direction["Down"] = 1] = "Down";
    Direction[Direction["Left"] = 2] = "Left";
    Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));

const dir = Direction.Up;
console.log(Direction[dir]); // → "Up"
```

`as const` bilan solishtirish:

```typescript
// Source
const Direction = {
  Up: "UP",
  Down: "DOWN",
} as const;

// Compile output — IIFE yo'q, oddiy object
const Direction = {
  Up: "UP",
  Down: "DOWN",
};
```

### Edge Cases

- Playground browser memory da `tsc` ni to'liq ishga tushiradi — compiler bundle brauzerga yuklanadi
- Web Worker ichida ishlaydi — main thread bloklanmaydi
- In-memory virtual file system — multi-file project simulate qilish mumkin (`.d.ts` fayl qo'shish)
- Versiya tanlash — `nightly` build ham mavjud (preview features)
- AST viewer faqat plugin orqali — default yoqilmagan

### Follow-up savollar

1. **"Playground da fayllar orasida import qilish mumkinmi?"** — Ha. "Add file" tugmasi bilan qo'shimcha fayl yaratish va `import { x } from "./other"` orqali ishlatish mumkin. Virtual FS shu uchun.
2. **"Playground da `tsconfig.json` ni nazorat qilish mumkinmi?"** — Ha, "TS Config" panel orqali barcha compiler options ni o'zgartirish mumkin. URL ham bu sozlamalarni saqlaydi.

</details>

---

### Savol 10: TypeScript ning eng muhim versiya milestone lari qaysilar? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Eng muhim milestone lar: TS 2.0 (`strictNullChecks`), 2.8 (conditional types + `infer`), 3.0 (`unknown`), 3.4 (`as const`), 4.0 (variadic + labeled tuples), 4.1 (template literal types), 4.9 (`satisfies`), 5.0 (TC39 decorators + const type parameters), 5.4 (`NoInfer<T>`), 5.5 (inferred type predicates), 5.8 (`erasableSyntaxOnly`).

### To'liq tushuntirish

| Versiya | Yil | Muhim xususiyat |
|---------|-----|-----------------|
| **1.0** | 2014 | Birinchi stable — generics, classes, modules |
| **1.6** | 2015 | `.tsx`, intersection types (`&`), user-defined type guards |
| **2.0** | 2016 | `strictNullChecks`, `never` type, discriminated unions, `readonly` |
| **2.1** | 2016 | `keyof`, mapped types, `Partial`, `Readonly` |
| **2.3** | 2017 | `strict: true` meta-flag |
| **2.8** | 2018 | Conditional types (`T extends U ? X : Y`), `infer` keyword |
| **3.0** | 2018 | Project references, `unknown` type |
| **3.4** | 2019 | `as const` (const assertions) |
| **3.7** | 2019 | Optional chaining (`?.`), nullish coalescing (`??`), assertion functions |
| **4.0** | 2020 | Variadic tuple types, labeled tuple elements |
| **4.1** | 2020 | Template literal types, key remapping, recursive conditional types |
| **4.3** | 2021 | `override` keyword |
| **4.4** | 2021 | `useUnknownInCatchVariables` `strict` ostiga qo'shildi |
| **4.7** | 2022 | `in`/`out` variance annotations, Node.js ESM support |
| **4.9** | 2022 | `satisfies` operator, `accessor` keyword |
| **5.0** | 2023 | TC39 Decorators (Stage 3), `const` type parameters, `moduleResolution: "bundler"` |
| **5.2** | 2023 | `using` declarations (Explicit Resource Management) |
| **5.4** | 2024 | `NoInfer<T>` utility type, preserved narrowing in closures |
| **5.5** | 2024 | `isolatedDeclarations`, inferred type predicates |
| **5.8** | 2025 | `erasableSyntaxOnly` — Node.js native TS support |

**Eng katta milestone lar tushuntirilishi:**

**TS 2.0 — `strictNullChecks`** — Bu versiyadan oldin `null` va `undefined` har tipga assign qilish mumkin edi. `strictNullChecks` bu muammoni hal qildi — `string` va `string | null` farqli tip lar. Minglab runtime xatolarni compile-time da ushlash imkonini berdi.

**TS 2.8 — Conditional types va `infer`** — Type system ga "if/else" kiritildi. `Exclude`, `Extract`, `ReturnType` kabi utility type lar conditional types ustida qurilgan. `infer` keyword type-level destructuring beradi.

**TS 4.1 — Template literal types** — String manipulation type darajasida: `` `${string}Changed` ``. Type-safe routing, event system, CSS class validation uchun asos.

**TS 4.9 — `satisfies`** — Oddiy annotation literal type larni widening qiladi, `as const` esa o'zgartirib bo'lmaydigan qiladi. `satisfies` o'rtadagi yechim — type check qiladi lekin inference saqlanadi.

**TS 5.0 — TC39 Decorators** — Standart decorator lar (Stage 3) qo'llab-quvvatlandi. Avvalgi `experimentalDecorators` legacy bo'lib qoldi (Angular, NestJS hozircha legacy ishlatadi).

**TS 5.8 — `erasableSyntaxOnly`** — Node.js 22.6+ `--experimental-strip-types` flag bilan native TypeScript support uchun. `enum`, `namespace`, constructor parameter properties — runtime kod hosil qiladi, native mode da ishlamaydi. `erasableSyntaxOnly: true` ularni compile-time da taqiqlaydi.

### Kod misol

`as const` (TS 3.4) vs `satisfies` (TS 4.9) farqi:

```typescript
// as const — literal + readonly
const config1 = {
  url: "/api",
  retries: 3,
} as const;
// type: { readonly url: "/api"; readonly retries: 3 }
// config1.retries = 5; ❌ readonly

// satisfies — type check + inference saqlanadi
const config2 = {
  url: "/api",
  retries: 3,
} satisfies { url: string; retries: number };
// type: { url: string; retries: number }
// config2.retries = 5; ✅ mutable, lekin type check qilingan
```

Conditional types (TS 2.8):

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type R = ReturnType<() => string>; // string
type N = ReturnType<number>;        // never
```

Template literal types (TS 4.1):

```typescript
type Event = "click" | "hover";
type Handler = `on${Capitalize<Event>}`;
// "onClick" | "onHover"
```

### Edge Cases

- TS 2.0 ning `strictNullChecks` ni mavjud loyihaga qo'shish — yuzlab xato bir zumda. Bosqichma-bosqich migration kerak
- TS 4.0 labeled tuple elements faqat IDE hint uchun — runtime ta'siri yo'q (type erasure)
- TS 5.0 decorator lar legacy bilan mos kelmaydi — migration uchun alohida loyiha kerak
- `const enum` va `enum` semantic farqi versiyalar bo'ylab o'zgarmagan, lekin TS 5.0 da `--isolatedModules` cheklovi qattiqlashdi

### Follow-up savollar

1. **"TS 4.9 `satisfies` va `as const` qachon qaysi birini ishlatish?"** — `as const` o'zgarmas, literal, readonly kerak bo'lganda (config object). `satisfies` type validation kerak lekin inference saqlanishi kerak bo'lganda (object literal). Ko'p hollarda `satisfies` afzal.
2. **"TS 5.x da eng amaliy yangilik qaysi?"** — `const` type parameter (5.0) — generic literal inference uchun foydali. `using` declarations (5.2) — resource management (file handle, lock) uchun. `NoInfer<T>` (5.4) — generic inference ni cheklash uchun.

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript ning versiya cadence — har 3-4 oyda minor release. Major versiya (5.x → 6.x) breaking change bo'lganda chiqariladi, lekin TypeScript siyosati — backward compat ni maksimum saqlash.

`erasableSyntaxOnly` (TS 5.8) Node.js ning native TypeScript support i bilan integratsiya uchun kiritildi. Node.js 22.6 `--experimental-strip-types` flag bilan TypeScript fayllarini to'g'ridan-to'g'ri bajarmaydi — faqat type annotation larni o'chirib JS sifatida bajaradi. Bu mexanizm faqat "erasable" syntax bilan ishlaydi — `enum`, `namespace`, constructor parameter properties runtime kod hosil qiladi va o'chmaydi.

TS 5.0 da migration vaqtida `moduleResolution: "bundler"` kiritildi — Vite, esbuild, Webpack bundler lar uchun. `bundler` mode `node10`/`node16` dan farqi: extension talab qilmaydi (`import { x } from "./module"`), `package.json` "exports" field qo'llab-quvvatlaydi, lekin Node.js native ESM qoidalarini majburlamaydi.

TS 5.5 ning **inferred type predicates** — agar function body `return value === null`, `return Array.isArray(x)` kabi narrowing pattern bo'lsa, TypeScript avtomatik `value is T` predicate hosil qiladi. Avval bu qo'lda yozish kerak edi.

</details>

</details>

---

## Output prediction savollari

### Savol 11: Quyidagi kodning compile natijasi nima va nima uchun? [Middle]

```typescript
interface Animal {
  name: string;
  sound(): string;
}

function makeSound(animal: Animal): string {
  if (animal instanceof Animal) {
    return animal.sound();
  }
  return "unknown";
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Compile-time xato: `'Animal' only refers to a type, but is being used as a value here`. `instanceof` faqat class (constructor function) bilan ishlaydi. `Animal` — interface, type erasure dan keyin runtime da mavjud emas.

### To'liq tushuntirish

Interface va type alias compile-time construct lar — `tsc` emitter ularni AST dan butunlay olib tashlaydi. `instanceof` esa runtime operator — JavaScript da `obj instanceof Constructor` deb yoziladi, va `Constructor` runtime da mavjud function bo'lishi kerak.

Bu xato Type Erasure ning amaliy oqibati. Runtime da interface tekshirish uchun ikki yondashuv bor:

1. **Property tekshirish** (structural check) — `typeof`, `in`, manual `typeof obj.method === "function"`
2. **User-defined type guard** — `function isAnimal(x: unknown): x is Animal { ... }`

### Kod misol

To'g'ri variantlar:

```typescript
interface Animal {
  name: string;
  sound(): string;
}

// ✅ Variant 1: Property tekshirish bilan narrowing
function makeSoundV1(animal: unknown): string {
  if (
    typeof animal === "object" && animal !== null &&
    "name" in animal && "sound" in animal &&
    typeof (animal as Record<string, unknown>).sound === "function"
  ) {
    return (animal as Animal).sound();
  }
  return "unknown";
}

// ✅ Variant 2: User-defined type guard
function isAnimal(value: unknown): value is Animal {
  return (
    typeof value === "object" && value !== null &&
    "name" in value && "sound" in value &&
    typeof (value as Record<string, unknown>).sound === "function"
  );
}

function makeSoundV2(animal: unknown): string {
  return isAnimal(animal) ? animal.sound() : "unknown";
}

// ✅ Variant 3: Class ishlatish (class runtime da mavjud)
class Dog implements Animal {
  constructor(public name: string) {}
  sound() { return "Woof!"; }
}

const dog = new Dog("Rex");
console.log(dog instanceof Dog); // ✅ true
```

### Edge Cases

- `class A implements I` — `implements` compile-time check, lekin `A` runtime da mavjud, shuning uchun `instanceof A` ishlaydi
- Abstract class — `instanceof` ishlaydi (abstract class runtime da JS class)
- `class A { #x = 1 }` — `#x in obj` syntax brand check sifatida ishlaydi (TS 4.5+, ergonomic brand checks). Runtime da `#x` private field mavjudligini tekshiradi — odatda `obj` `A` orqali yaratilganini bildiradi
- `typeof Animal` value sifatida ishlatilsa ham xato: `'Animal' only refers to a type` — chunki interface'ning value space da hech qanday ko'rinishi yo'q

### Follow-up savollar

1. **"`abstract class` bilan `instanceof` ishlaydimi?"** — Ha. Abstract class JS da oddiy class ga compile bo'ladi, faqat `new AbstractClass()` taqiqlanadi (compile-time). Instance ning prototype chain da abstract class bor.

</details>

---

### Savol 12: Quyidagi kod TypeScript ga JavaScript ga qanday compile bo'ladi? [Middle]

```typescript
interface Config {
  host: string;
  port: number;
  debug?: boolean;
}

type Environment = "development" | "production" | "staging";

function createConfig(env: Environment): Config {
  const config: Config = {
    host: "localhost",
    port: env === "production" ? 443 : 3000,
  };

  if (env !== "production") {
    config.debug = true;
  }

  return config;
}

const cfg: Config = createConfig("development");
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type erasure: `interface Config` va `type Environment` butunlay o'chiriladi, barcha `: Config`, `: Environment`, `: string`, `: number` annotation lar ham. Faqat JavaScript logikasi qoladi.

### To'liq tushuntirish

`tsc` Emitter AST dan type-related node larni olib tashlaydi:

- `InterfaceDeclaration` (`interface Config`) — yo'q
- `TypeAliasDeclaration` (`type Environment`) — yo'q
- `TypeReference` (`: Config`, `: number`) — yo'q
- Function return type annotation (`: Config`) — yo'q

Qolgan kod — oddiy JavaScript: function declaration, variable, conditional, object literal, return statement. Runtime da type system iz qoldirmaydi.

### Kod misol

Compile output:

```javascript
// interface Config — butunlay o'chirildi
// type Environment — butunlay o'chirildi

function createConfig(env) {
  const config = {
    host: "localhost",
    port: env === "production" ? 443 : 3000,
  };

  if (env !== "production") {
    config.debug = true;
  }

  return config;
}

const cfg = createConfig("development");
```

`Config` interface optional `debug?: boolean` — JavaScript da `config.debug` faqat `env !== "production"` bo'lganda mavjud. TypeScript optional property bilan bu noaniqlikni ishlay oladi (`exactOptionalPropertyTypes` flag bilan farqlash mumkin).

### Edge Cases

- `target: ES5` bilan const `var` ga aylanardi (eski TS, hozir default ES2017+)
- `noUnusedParameters: true` bo'lsa `env` ishlatilmasa xato berishi mumkin
- `removeComments: true` bilan barcha comment lar ham o'chiriladi
- Source map (`.js.map`) generation bilan compile output ga `//# sourceMappingURL=...` qo'shiladi

### Follow-up savollar

1. **"Type erasure dan keyin runtime'da `cfg` ning type'ini bilish mumkinmi?"** — Yo'q. `typeof cfg === "object"` qaytaradi, lekin u `Config` interface ekanini tekshirish runtime da imkonsiz. `zod` kabi schema validator ishlatish kerak.

</details>

---

## Coding challenges

### Savol 13: Structural typing — har chaqiruv natijasi va sabab [Junior+]

**Savol:** Quyidagi kodda har bir `greet` chaqiruvi uchun natija qanday? Qaysi biri compile xato beradi?

```typescript
interface Greetable {
  name: string;
  greet(): string;
}

function greet(obj: Greetable): string {
  return obj.greet();
}

class Person {
  constructor(public name: string) {}
  greet() { return `Hi, I'm ${this.name}`; }
}

class Robot {
  name = "R2D2";
  greet() { return `Beep, I'm ${this.name}`; }
  recharge() { return "Charging..."; }
}

const plain = {
  name: "Anonymous",
  greet() { return "Hello!"; },
  extra: 42,
};

// A
greet(new Person("Ali"));
// B
greet(new Robot());
// C
greet(plain);
// D
greet({ name: "Test", greet() { return "Hey"; }, extra: 42 });
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A, B, C ishlaydi — structural typing tufayli shape mos kelsa kifoya. D compile xato beradi — object literal to'g'ridan-to'g'ri berilganda excess property checking yoqiladi.

### To'liq tushuntirish

```
A: "Hi, I'm Ali"          ✅ Person shape Greetable ga mos
B: "Beep, I'm R2D2"       ✅ Robot ham Greetable shape ga mos
                             (qo'shimcha recharge() ta'sir qilmaydi)
C: "Hello!"                ✅ plain variable orqali — excess property
                             check YO'Q (variable assignment)
D: ❌ Compile error        Excess property 'extra' — object literal
                             to'g'ridan-to'g'ri berilganda TS qo'shimcha
                             property larni tekshiradi
```

**Excess property checking (EPC)** — TypeScript ning maxsus xulqi. Object literal funksiya argumenti yoki variable annotation ga to'g'ridan-to'g'ri berilganda, qo'shimcha property lar xato deb qaraladi. Bu structural typing ning "qattiqlashtirilgan" varianti — type level da yo'q property lar bo'lmasligini ta'minlash uchun.

EPC qachon ishlamaydi:
- Variable orqali berilganda (C variant)
- `as` assertion bilan
- Spread operator orqali (`{...plain}`)
- Index signature mavjud bo'lsa (`[key: string]: unknown`)

### Kod misol

EPC ni o'tkazib yuborish:

```typescript
interface Config { host: string; }

// ❌ Compile error — excess property
const c1: Config = { host: "localhost", port: 3000 };

// ✅ Variable orqali — EPC o'tkaziladi
const obj = { host: "localhost", port: 3000 };
const c2: Config = obj;

// ✅ as assertion
const c3 = { host: "localhost", port: 3000 } as Config;

// ✅ Index signature
interface ConfigOpen {
  host: string;
  [key: string]: unknown;
}
const c4: ConfigOpen = { host: "localhost", port: 3000 };
```

### Edge Cases

- EPC `satisfies` operator bilan ham ishlaydi (TS 4.9+) — `satisfies` bilan EPC saqlanadi
- Nested object da EPC faqat outer level da — ichki object excess property tekshirilmasligi mumkin
- `?` optional property `undefined` qiymat bilan berilsa EPC chiqarmaydi
- Object literal bilan rest parameter — `function f(opts: Config, ...rest)` — opts uchun EPC yoqiladi

### Follow-up savollar

1. **"EPC ni o'chirish mumkinmi?"** — Yo'q, bu TypeScript ning hardcoded xulqi. Faqat yuqoridagi pattern lar bilan o'tkazib yuborish mumkin.

</details>

---

### Savol 14: User-defined type guard yozing [Middle]

**Savol:** `isUser` type guard funksiyasini yozing. `unknown` tipdan `User` tipga xavfsiz tekshirish qilsin:

```typescript
interface User {
  name: string;
  email: string;
  age: number;
}

function isUser(value: unknown): /* return type? */ {
  // Implement qiling
}

const data: unknown = JSON.parse('{"name":"Ali","email":"a@t.com","age":25}');

if (isUser(data)) {
  console.log(data.name);
  console.log(data.email);
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Return type `value is User` (type predicate). Funksiya tanasida `typeof`, `in`, va property type tekshirish orqali har field validate qilinadi. Type predicate TypeScript ga narrowing signal beradi.

### To'liq tushuntirish

`value is User` — **type predicate**. Funksiya `true` qaytarsa, TypeScript caller scope da `value` ni `User` deb hisoblaydi. Bu user-defined type guard.

Type guard tanasida har property uchun:
1. Object ekanligini tekshirish (`typeof === "object"`)
2. `null` emasligini (`typeof null === "object"` — JS legacy bug)
3. Property mavjudligini (`"name" in value`)
4. Property tipini (`typeof value.name === "string"`)

`Record<string, unknown>` intermediate cast — `as any` dan xavfsizroq.

### Kod misol

```typescript
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;

  const v = value as Record<string, unknown>;
  return (
    "name" in v && typeof v.name === "string" &&
    "email" in v && typeof v.email === "string" &&
    "age" in v && typeof v.age === "number"
  );
}

const data: unknown = JSON.parse('{"name":"Ali","email":"a@t.com","age":25}');

if (isUser(data)) {
  // data: User (narrowed)
  console.log(data.name.toUpperCase()); // ✅ string method
  console.log(data.age * 2);             // ✅ number arithmetic
}

// Invalid input
console.log(isUser({ name: "Ali", email: "a@t.com" })); // false — age yo'q
console.log(isUser({ name: "Ali", email: "a@t.com", age: "25" })); // false — age string
console.log(isUser(null)); // false
console.log(isUser("string")); // false
```

### Edge Cases

- TS 5.5+ — funksiya tanasida narrowing pattern bo'lsa, TypeScript avtomatik type predicate hosil qiladi (inferred type predicates). Lekin aniq yozish IDE uchun aniqroq
- Type guard `false` qaytarsa, TypeScript `value` ni `User` dan **chiqarib tashlaydi** — `unknown` qoladi
- Type guard narrowing faqat keyingi statement larga ta'sir qiladi. TS 5.4 dan boshlab closure ichida ham saqlanadi, agar variable (parameter yoki `let`) closure yaratilishidan oldin oxirgi marta assign qilingan va hech qaysi nested function ichida qayta o'zgartirilmagan bo'lsa
- `value is User` semantic — agar funksiya noto'g'ri implement qilingan bo'lsa, runtime crash bo'lishi mumkin (TS ishonadi)

### Follow-up savollar

1. **"`zod` bilan solishtiring — qaysi biri afzal?"** — `zod` deklarativ schema bilan validation va type inference birlashtiradi: `const UserSchema = z.object({...})`, `type User = z.infer<typeof UserSchema>`. Manual type guard yozish faqat kichik loyihalar yoki performance critical joylarda mantiqli.

</details>

---

### Savol 15: `unknown` dan xavfsiz nested data extraction [Middle+]

**Savol:** API dan kelgan `unknown` response dan xavfsiz tarzda `users` ro'yxatini chiqaring:

```typescript
interface ApiUser {
  id: number;
  name: string;
  active: boolean;
}

function extractUsers(response: unknown): ApiUser[] {
  // Implement qiling:
  // - response.data.users tuzilmasini xavfsiz extract qiling
  // - Format noto'g'ri bo'lsa bo'sh array
  // - Har element individual tekshirilsin
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har qadamda `unknown` dan bitta daraja chuqurroqqa narrowing. `Record<string, unknown>` intermediate cast. Element filtering uchun `.filter` ichida type predicate. Defensive coding — har qadamda fallback `[]`.

### To'liq tushuntirish

Nested unknown data extraction qadamlari:

1. Outer object ekanligi (`typeof === "object"`, `!== null`)
2. `"data"` property mavjudligi
3. `data` object ekanligi
4. `"users"` property mavjudligi
5. `users` array ekanligi (`Array.isArray`)
6. Har element individual `ApiUser` ekanini tekshirish (`.filter` + type predicate)

### Kod misol

```typescript
function extractUsers(response: unknown): ApiUser[] {
  // 1. Outer object check
  if (typeof response !== "object" || response === null || !("data" in response)) {
    return [];
  }

  const data = (response as Record<string, unknown>).data;

  // 2. data object check
  if (typeof data !== "object" || data === null || !("users" in data)) {
    return [];
  }

  const users = (data as Record<string, unknown>).users;

  // 3. Array check
  if (!Array.isArray(users)) {
    return [];
  }

  // 4. Element-level filtering with type predicate
  return users.filter((u): u is ApiUser => {
    if (typeof u !== "object" || u === null) return false;
    const obj = u as Record<string, unknown>;
    return (
      "id" in obj && typeof obj.id === "number" &&
      "name" in obj && typeof obj.name === "string" &&
      "active" in obj && typeof obj.active === "boolean"
    );
  });
}

// Test
const valid = { data: { users: [{ id: 1, name: "Ali", active: true }] } };
console.log(extractUsers(valid)); // → [{ id: 1, name: "Ali", active: true }]

const invalid = { data: { users: [{ id: "1", name: "Ali", active: true }] } };
console.log(extractUsers(invalid)); // → [] (id string, filter o'chiradi)

console.log(extractUsers(null)); // → []
console.log(extractUsers("string")); // → []
```

### Edge Cases

- Production da `zod` yoki `valibot` deklarativ va to'g'ri xato xabarlar beradi
- `.filter(predicate)` array element soni ozaytirishi mumkin — caller bunga tayyor bo'lishi kerak
- Performance: katta array uchun har element uchun `typeof` chaqiruvi sekinroq — bulk validation library lar optimallashtirilgan
- Recursive struktura uchun bu pattern qo'l bilan yozish murakkab — schema validator avtomatlashtiriladi

### Follow-up savollar

1. **"`zod` bilan bu qanday yoziladi?"** —
```typescript
import { z } from "zod";
const ApiUserSchema = z.object({ id: z.number(), name: z.string(), active: z.boolean() });
const ResponseSchema = z.object({ data: z.object({ users: z.array(ApiUserSchema) }) });
function extractUsers(response: unknown): ApiUser[] {
  const result = ResponseSchema.safeParse(response);
  return result.success ? result.data.data.users : [];
}
```
Deklarativ, validation va type inference birga.

</details>

---

### Savol 16: Enum compile output ni yozing [Middle+]

**Savol:** Quyidagi TypeScript kodi JavaScript ga qanday compile bo'ladi? Ikki variantning farqini yozing:

```typescript
// Variant A
enum Color {
  Red,
  Green,
  Blue
}
const c1 = Color.Green;
const name1 = Color[1];

// Variant B
const enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT"
}
const d1 = Direction.Up;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Variant A: enum IIFE yaratadi + reverse mapping (numeric uchun). Variant B: `const enum` qiymat inline bo'ladi, runtime da hech narsa qolmaydi.

### To'liq tushuntirish

**Variant A — oddiy enum** runtime kod hosil qiladi:

```javascript
var Color;
(function (Color) {
    Color[Color["Red"] = 0] = "Red";
    Color[Color["Green"] = 1] = "Green";
    Color[Color["Blue"] = 2] = "Blue";
})(Color || (Color = {}));
const c1 = Color.Green;   // → 1
const name1 = Color[1];   // → "Green" (reverse mapping)
```

Pattern tushuntirish:
```javascript
Color[Color["Red"] = 0] = "Red";
// 1-qadam: Color["Red"] = 0 → forward mapping (assignment 0 qaytaradi)
// 2-qadam: Color[0] = "Red" → reverse mapping
```

**Variant B — `const enum`** to'liq inline:

```javascript
// enum butunlay yo'q
const d1 = "UP";
```

`const enum` qiymatlari compile time da inline qilinadi — runtime object yaratilmaydi.

### Kod misol

```typescript
// Source
enum Color { Red, Green, Blue }
const c = Color.Green;

// Compiled
var Color;
(function (Color) {
    Color[Color["Red"] = 0] = "Red";
    Color[Color["Green"] = 1] = "Green";
    Color[Color["Blue"] = 2] = "Blue";
})(Color || (Color = {}));
const c = Color.Green; // 1

// Source — const enum
const enum Status { Active = "ACTIVE", Inactive = "INACTIVE" }
const s = Status.Active;

// Compiled — inline
const s = "ACTIVE";
```

Solishtirish jadvali:

| Xususiyat | `enum` | `const enum` |
|-----------|--------|--------------|
| Runtime object | ✅ IIFE | ❌ Inline |
| Reverse mapping | ✅ (numeric) | ❌ |
| Bundle size | Kattaroq | Kichikroq |
| Dynamic access (`Color[var]`) | ✅ | ❌ |
| `isolatedModules` | ✅ | ⚠️ Cheklovlar |
| `erasableSyntaxOnly` | ❌ | ❌ |

### Edge Cases

- `const enum` ni `isolatedModules: true` bilan re-export qilib bo'lmaydi — Babel/esbuild butun loyihani ko'ra olmaydi
- String enum reverse mapping yo'q — faqat numeric da
- `enum X { A = compute() }` — computed member bilan enum hali ham xavfli (numeric soundness yo'q)
- `enum` `erasableSyntaxOnly: true` (TS 5.8) bilan taqiqlanadi — Node.js native TypeScript bilan ishlamaydi

### Follow-up savollar

1. **"`const enum` ni production da ishlatish xavfsizmi?"** — Faqat bitta paket ichida (cross-package re-export muammosi yo'q bo'lsa). Library yozsangiz — `enum` yoki `as const` object afzalroq.
2. **"Enum o'rniga zamonaviy alternativa?"** — Union literal type (`type Color = "red" | "green"`) yoki `as const` object. Bundle size 0, tree shaking yaxshi, isolatedModules muammosiz.

</details>

---

### Savol 17: TypeScript versiyasi va feature mapping [Middle]

**Savol:** Quyidagi feature lar qaysi TypeScript versiyalarida kiritilgan? Tartibga qo'ying:

- `satisfies` operator
- `unknown` type
- `as const` (const assertions)
- Template literal types
- TC39 Decorators (Stage 3)
- `using` declarations

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Tartib: `unknown` (TS 3.0, 2018) → `as const` (TS 3.4, 2019) → Template literal types (TS 4.1, 2020) → `satisfies` (TS 4.9, 2022) → TC39 Decorators (TS 5.0, 2023) → `using` (TS 5.2, 2023).

### To'liq tushuntirish

| Versiya | Yil | Feature | Asosiy qiymat |
|---------|-----|---------|---------------|
| **3.0** | 2018 | `unknown` | `any` ning xavfsiz alternativasi |
| **3.4** | 2019 | `as const` | Literal type + readonly inference |
| **4.1** | 2020 | Template literal types | Type-level string manipulation |
| **4.9** | 2022 | `satisfies` | Type check + inference saqlanishi |
| **5.0** | 2023 | TC39 Decorators (Stage 3) | Standart decorator support |
| **5.2** | 2023 | `using` declarations | Explicit Resource Management |

### Kod misol

Har birining tipik ishlatilishi:

```typescript
// TS 3.0 — unknown
function parse(json: string): unknown {
  return JSON.parse(json);
}

// TS 3.4 — as const
const ROLES = ["admin", "user"] as const;
type Role = typeof ROLES[number]; // "admin" | "user"

// TS 4.1 — Template literal types
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">; // "onClick"

// TS 4.9 — satisfies
const config = {
  url: "/api",
  retries: 3,
} satisfies { url: string; retries: number };
// type: { url: string; retries: number }, lekin literal inference saqlanadi

// TS 5.0 — TC39 Decorators
function log(target: any, context: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    console.log(`Calling ${String(context.name)}`);
    return target.apply(this, args);
  };
}

class Service {
  @log
  fetch() { return "data"; }
}

// TS 5.2 — using declarations
class FileHandle {
  [Symbol.dispose]() { console.log("closed"); }
}

function readFile() {
  using file = new FileHandle();
  // Block tugaganda avtomatik file[Symbol.dispose]()
}
```

### Edge Cases

- TS 5.0 decorators TS 4.x decorators (legacy `experimentalDecorators`) bilan mos kelmaydi — migration kerak
- `using` `Symbol.dispose` global metodi talab qiladi — Node.js 20+ yoki polyfill
- `satisfies` `as const` bilan birga ishlatilishi mumkin: `{ ... } as const satisfies T`
- Template literal types `infer` bilan birga — kuchli string parsing pattern lar

### Follow-up savollar

1. **"TS 5.x yangi feature lar ko'p — qaysi biri eng amaliy?"** — Amaliy zarurlik bo'yicha: `satisfies` (har joyda), `const` type parameter (literal inference), `NoInfer<T>` (generic constraint), `using` (resource management).

</details>

---

## Bug fix savollari

### Savol 18: `tsc` versiyasi va VS Code versiyasi mos kelmaslik muammosi [Middle]

**Savol:** Loyihada `tsc --version` 5.4 ko'rsatadi, lekin VS Code da bir kod uchun boshqa xato ko'rsatiladi. Sabab nima va qanday tuzatish kerak?

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

VS Code o'z ichida TypeScript bundle saqlaydi — odatda eskiroq versiya (masalan 5.2). Loyiha `node_modules/typescript` da 5.4 bo'lsa, IDE va `tsc` natijasi farq qiladi. Yechim: VS Code da `Cmd+Shift+P` → "TypeScript: Select TypeScript Version" → "Use Workspace Version".

### To'liq tushuntirish

VS Code TypeScript Language Service ni o'zining bundled `typescript.js` bilan ishlatadi. Bu version `node_modules/typescript` versiyasidan farq qilishi mumkin. Natijada:

- IDE inline error highlighting bir versiya bo'yicha
- `tsc` command-line boshqa versiya bo'yicha
- Yangi feature (`satisfies`, `using`) — IDE bilmasa qizil chiziq, lekin `tsc` to'g'ri compile qiladi
- Eski feature deprecation — IDE yangi versiya bo'lsa ko'rsatadi, eski `tsc` ko'rsatmaydi

### Kod misol

VS Code workspace settings (`.vscode/settings.json`):

```jsonc
{
  // Workspace TypeScript versiyasini majburlash
  "typescript.tsdk": "node_modules/typescript/lib",

  // VS Code IDE da workspace version'ni ko'rsatish
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

CLI da tekshirish:

```bash
# Loyiha versiyasi
npx tsc --version  # Version 5.4.0

# Global VS Code bundled versiya (taxminan)
# VS Code → Status Bar pastida "TypeScript X.Y" ko'rinadi
```

### Edge Cases

- Yarn PnP yoki pnpm bilan `node_modules/typescript/lib` yo'li boshqacha — `.yarn/sdks/typescript/lib` yoki shunga o'xshash
- WebStorm/IntelliJ ham bir xil muammo — IDE settings da TypeScript service version tanlash kerak
- CI da `tsc` `node_modules` versiyasini ishlatadi — odatda muammo bo'lmaydi
- Monorepo da har paket o'z TypeScript versiyasini ishlatishi mumkin — root da `typescript.tsdk` o'rnatish noto'g'ri natija beradi

### Follow-up savollar

1. **"Loyihada qaysi TypeScript versiyasini ishlatish kerak?"** — Imkon qadar zamonaviy (5.x). Migration loyihalarda esa `@tsconfig/recommended` template'lar mavjud. Major version o'zgarishi bilan ehtiyot bo'lish.

</details>

---

### Savol 19: `strict: false → strict: true` migration muammosi [Middle+]

**Savol:** Eski loyihada `strict: false` edi. `strict: true` yoqilgandan keyin 500+ ta type xato chiqdi. Qanday yondashish kerak?

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bosqichma-bosqich migration: `strict` ichidagi 8 flag ni bittadan yoqish (avval `strictNullChecks`, keyin `noImplicitAny`, va hokazo). Har flag uchun xato larni alohida tuzatish. Vaqtinchalik xato suppress qilish uchun `// @ts-expect-error` yoki butun fayl uchun `// @ts-nocheck`.

### To'liq tushuntirish

`strict: false` bilan TypeScript barcha o'zgaruvchi va parameter larni `any` deb qaraydi, `null`/`undefined` har tipga assign bo'ladi. `strict: true` yoqilganda bularning hammasi "noxushlik" sifatida ochiladi.

Migration strategiyasi (sodda → murakkab):

1. **`strictNullChecks`** — eng katta ta'sir. Function parametrlar va return type larga `| null`/`| undefined` qo'shish. Optional chaining (`?.`) ishlatish.
2. **`noImplicitAny`** — har parametrga type annotation qo'shish. Implicit return type larga aniq type.
3. **`strictFunctionTypes`** — function type signature larini kuchaytirish.
4. **`strictBindCallApply`** — `.bind()`, `.call()`, `.apply()` parametrlarini type-safe qilish.
5. **`strictPropertyInitialization`** — class property larni constructor da init yoki `!` definite assignment.
6. **`noImplicitThis`** — `this: ContextType` parametr qo'shish.
7. **`alwaysStrict`** — odatda muammosiz (ESM da avtomatik).
8. **`useUnknownInCatchVariables`** — `catch(e)` blok larida `e instanceof Error` check qo'shish.

### Kod misol

Bosqichma-bosqich tsconfig.json:

```jsonc
// Bosqich 1: strictNullChecks
{
  "compilerOptions": {
    "strict": false,
    "strictNullChecks": true
  }
}

// Bosqich 2: + noImplicitAny
{
  "compilerOptions": {
    "strict": false,
    "strictNullChecks": true,
    "noImplicitAny": true
  }
}

// Bosqich 3+: ...

// Final: strict: true
{
  "compilerOptions": {
    "strict": true
  }
}
```

File-level migration:

```typescript
// @ts-nocheck — bu fayl butunlay type check'dan chiqariladi
function legacyFunction(data) {
  return data.value;
}

// Yoki har xato satrida lokal suppress
function partialMigration(data: unknown) {
  // @ts-expect-error — migration tugagach olib tashlanadi
  return data.value;
}

// Yangi fayllar uchun aniq strict
function newFunction(data: { value: string }): string {
  return data.value;
}
```

### Edge Cases

- `// @ts-expect-error` — vaqtincha xato suppress qilish, lekin xato yo'q bo'lsa o'zi xato beradi (yaxshi cleanup signal)
- `// @ts-ignore` — xatoni ignore qilish, ammo silently — code review da topish qiyin
- `strictPropertyInitialization: false` — Angular kabi framework larda ba'zan kerak (DI bilan property lar fill bo'ladi)
- Third-party library `@types/*` package lari strict mode ga mos kelmasligi mumkin — `skipLibCheck: true` muammoni hal qiladi

### Follow-up savollar

1. **"Qaysi flag eng ko'p xato beradi?"** — Odatda `strictNullChecks` — xatolarning katta qismi shu yerdan. Sababi: API javob, DOM query, object property access — har joyda `null`/`undefined` mumkin.
2. **"Migration vaqtida new fayllar uchun nima qilish kerak?"** — Yangi fayllar uchun `strict: true` ni alohida loyiha sifatida (project references) yoki har faylda `// @ts-check` bilan strict mode majburlash mumkin.

</details>

---

### Savol 20: `skipLibCheck: false` bilan node_modules xatolari [Middle]

**Savol:** `tsc` ishga tushirilganda `node_modules/@types/...` ichidagi fayllar dan yuzlab xato chiqyapti. Mening kodimda esa xato yo'q. Sabab nima?

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`skipLibCheck: false` (default oldin) bilan `tsc` `node_modules` ichidagi barcha `.d.ts` fayllarni ham tekshiradi. Third-party library `.d.ts` larida ba'zan xato bo'ladi. Yechim: `"skipLibCheck": true` qo'yish (production loyihalar uchun tavsiya).

### To'liq tushuntirish

`tsc` default ravishda barcha `.d.ts` fayllarni type check qiladi — shu jumladan `node_modules` ichidagi `@types/*` paketlar. `@types/react`, `@types/node` kabi keng tarqalgan paketlar ham har xil versiya bilan birga ishlatilganda nomuvofiqlik bo'lishi mumkin.

`skipLibCheck: true` — bu tekshiruvni o'tkazib yuboradi. Library `.d.ts` larini "ishonchli" deb qaraydi va faqat sizning `.ts` kodingizni tekshiradi.

**Afzalliklari:**
- Compile vaqti sezilarli kamayadi
- Third-party type xatolar sizni bezovta qilmaydi
- CI da false-positive xatolar yo'qoladi

**Kamchiliklari (nadir):**
- Library yozayotgan bo'lsangiz, o'z `.d.ts` xatolarini ko'rmaslik mumkin
- Library da `peerDependencies` muammosi yashirilishi mumkin

### Kod misol

Tavsiya etilgan tsconfig:

```jsonc
{
  "compilerOptions": {
    "skipLibCheck": true, // Production loyihalar uchun
    "strict": true,
    // ...
  }
}
```

Library yozsangiz — alohida test:

```bash
# Library kodi uchun skipLibCheck: false
tsc --skipLibCheck false

# Production build uchun
tsc --skipLibCheck true
```

### Edge Cases

- `@types/*` paketlar versiya konflikti — `npm ls @types/react` bilan tekshirish
- Monorepo da bir nechta paket bir xil `@types/*` ning har xil versiyasini ishlatishi
- TS 5.x default `skipLibCheck: true` — yangi loyihalarda muammo kamroq
- Yarn PnP / pnpm strict mode da `@types/*` resolution muammosi alohida

### Follow-up savollar

1. **"`skipLibCheck: true` qachon noto'g'ri?"** — Library yozayotganingizda. Sizning `.d.ts` fayllaringizning import bog'liqliklarini tekshirish kerak. Production loyiha uchun esa hech qachon noto'g'ri emas.

</details>

---

### Savol 21: `allowJs` va `checkJs` chalkashligi [Middle]

**Savol:** Loyihada `allowJs: true, checkJs: false` qo'yilgan. JavaScript fayllar compile bo'ladi, lekin xato chiqarmaydi. Migration tugagandan keyin nima qilish kerak?

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Migration bosqichlari: (1) `allowJs: true, checkJs: false` — JS fayllar compile bo'ladi, tekshirilmaydi. (2) Asta-sekin `.js` → `.ts` ga o'tkazish. (3) Migration tugagach `allowJs: false` qilish. Oraliq bosqichda `// @ts-check` bilan JS fayllarni file-level da tekshirish mumkin.

### To'liq tushuntirish

**`allowJs`** — JavaScript fayllarni TypeScript loyihasiga qo'shish (compile va resolution uchun).

**`checkJs`** — JavaScript fayllarni ham type check qilish (JSDoc type comment lar asosida).

Migration strategiya:

1. **Boshlang'ich bosqich**: `allowJs: true, checkJs: false` — JS fayllar mavjud, `import`/`require` ishlaydi, lekin type check yo'q.
2. **JSDoc qo'shish**: ba'zi JS fayllarda `// @ts-check` bilan file-level type check yoqish va JSDoc type annotation qo'shish.
3. **Fayl o'tkazish**: `.js` ni `.ts`/`.tsx` ga aylantirish, type annotation lar qo'shish.
4. **Final**: barcha fayl TypeScript bo'lganda `allowJs: false` (yoki shunchaki olib tashlash — default).

### Kod misol

Boshlang'ich tsconfig (migration boshida):

```jsonc
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "outDir": "./dist",
    "strict": true,
    "include": ["src/**/*"]
  }
}
```

File-level type check (JS fayl ichida):

```javascript
// legacy/utils.js
// @ts-check

/**
 * @param {string} name
 * @param {number} age
 * @returns {string}
 */
export function greet(name, age) {
  return `${name} is ${age}`; // ✅ JSDoc dan type check
}

greet(42, "Ali"); // ❌ Argument types are wrong
```

Final tsconfig (migration tugagach):

```jsonc
{
  "compilerOptions": {
    // allowJs olib tashlandi (default: false)
    "strict": true,
    "include": ["src/**/*.ts", "src/**/*.tsx"]
  }
}
```

### Edge Cases

- `allowJs: true, checkJs: true` bilan `// @ts-nocheck` qo'shish — fayl-level da check disable
- JSDoc `@type` annotation lar TypeScript type bilan to'liq mos kelmasligi mumkin (template literal types qo'llab-quvvatlanmaydi)
- `outDir` bilan `.js` fayllar JS sifatida output ga ko'chiriladi (yoki transpile, `target` bilan)
- `allowJs` bilan node_modules dagi JS bog'liqlik lar avtomatik ravishda qamrab olinadi — bundle size oshishi mumkin

### Follow-up savollar

1. **"JSDoc bilan TypeScript ni butunlay almashtirish mumkinmi?"** — Kichik loyihalar uchun ha — VS Code JSDoc type lar bilan IntelliSense beradi. Lekin advanced type (conditional, mapped, template literal) JSDoc da yo'q yoki cheklangan. Production uchun `.ts` fayl tavsiya etiladi.

</details>

---

### Savol 22: TypeScript Checker'da `getNarrowedType` qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`getNarrowedType()` — `checker.ts` ichidagi control flow analysis funktsiyasi. Har AST node uchun mavjud union type'ni `typeof`, `instanceof`, `in`, type predicate, equality, truthiness check'lar asosida filter qiladi. Compiler scope ichidagi har flow node uchun cached narrowed type yuritadi.

### To'liq tushuntirish

Control flow narrowing — TypeScript'ning eng kuchli soundness mexanizmi. Har AST statement'idan keyin variable type'i o'zgarishi mumkin.

Narrowing manbalari:

| Pattern | Effect |
|---------|--------|
| `typeof x === "string"` | `x` → `string` (primitive narrowing) |
| `x instanceof MyClass` | `x` → `MyClass` (class narrowing) |
| `"prop" in x` | `x` → `T extends { prop: ... }` (property narrowing) |
| `x === null` | `x` → `null` (equality narrowing) |
| `Array.isArray(x)` | `x` → `any[]` (built-in predicate) |
| `isUser(x)` | `x` → `User` (user-defined predicate) |
| `if (x)` | `x` → truthy (truthiness narrowing) |
| `switch(tag)` | tagged union member narrowing |

Compiler ichida `FlowNode` structure har basic block uchun:
- `FlowStart` — function entry
- `FlowAssignment` — variable yangi qiymat olganda
- `FlowCondition` — `if`/`while` shart
- `FlowSwitchClause` — `case` clause
- `FlowMutation` — variable mutate qilinganda

Har use site `getFlowTypeOfReference()` chaqiriladi — flow graph bo'ylab back-traversal narrowed type'ni hisoblaydi.

### Kod misol

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  // Bu joyda shape: Shape (union)

  if (shape.kind === "circle") {
    // FlowCondition narrowing
    // shape: { kind: "circle"; radius: number }
    return Math.PI * shape.radius ** 2;
  }

  if (shape.kind === "square") {
    // shape: { kind: "square"; side: number }
    return shape.side ** 2;
  }

  // Exhaustiveness check — shape: { kind: "triangle"; ... }
  return (shape.base * shape.height) / 2;
}

// Discriminated union exhaustive check
function exhaustive(shape: Shape): number {
  switch (shape.kind) {
    case "circle":  return Math.PI * shape.radius ** 2;
    case "square":  return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
    default:
      // shape: never — barcha case'lar qoplagan
      const _exhaustive: never = shape;
      throw new Error(`Unhandled: ${_exhaustive}`);
  }
}
```

Narrowing closure'da yo'qolardi (TS 5.4'gacha):

```typescript
function process(value: string | null) {
  if (value === null) return;
  // value: string

  setTimeout(() => {
    // TS 5.4'gacha: value: string | null (narrowing yo'qolardi)
    // TS 5.4+: value: string. `value` parameter closure
    // yaratilishidan oldin oxirgi marta belgilangan va hech
    // qaysi nested function ichida qayta assign qilinmaydi —
    // shuning uchun narrowing saqlanadi
    console.log(value.toUpperCase());
  }, 100);
}
```

### Edge Cases

- **Mutation invalidates narrowing** — agar variable narrowing'dan keyin yangi qiymat olsa, narrowing yo'qoladi
- **Function parameter narrowing** — closure orqali yuborilganda TS 5.4'gacha yo'qolardi. TS 5.4 da parameter va `let` variable narrowing'i, agar oxirgi assignment closure'dan oldin bo'lsa va nested function ichida qayta assign qilinmasa, saqlanadi
- **`let` vs `const`** — `const` hech qachon qayta assign qilinmaydi, shuning uchun closure ichidagi narrowing analizi har doim ishonchli. `let` uchun TS 5.4 oxirgi assignment nuqtasini topadi
- **`switch` fallthrough** — `case` ichida `break` yoki `return` bo'lmasa, narrowing keyingi case'ga "joriy" bo'ladi (TS bilmaydi)
- **`while`/`for` loop** — har iteration boshida narrowing reset bo'ladi (mutation imkoniyati tufayli)

### Follow-up savollar

1. **"Nima uchun narrowing callback'da yo'qoladi?"** — JavaScript closure orqali callback istalgan vaqtda ishga tushishi mumkin — compiler callback ichidagi `value` reference'ini original declaration type bo'yicha tekshiradi. TS 5.4 narrowing'ni saqlaydi, agar variable closure yaratilishidan oldin oxirgi marta belgilangan va nested function ichida qayta assign qilinmagan bo'lsa.
2. **"Exhaustive check `default` clause'sida `never` qanday ishlatish kerak?"** — `const _: never = shape;` — agar barcha `case` qoplagan bo'lsa, `shape` `never` ga narrowing qilinadi va assignment compile'da o'tadi. Yangi union member qo'shilganda compile error — yangi case yozish majburlanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Flow graph internals:**

Compiler binder bosqichida AST bo'ylab yurib har statement uchun `FlowNode` yaratadi. Flow graph DAG (Directed Acyclic Graph) — har node oldingi node'larga reference saqlaydi.

```
FlowStart
   ↓
FlowAssignment (let x: string | null = ...)
   ↓
FlowCondition (if x !== null)
   ↓ true branch          ↓ false branch
FlowAssignment            FlowReturn
   ↓
FlowJoin (if blok tugashi)
   ↓
...
```

`getFlowTypeOfReference()` algoritm:
1. Use site'dan boshlanadi
2. Flow graph bo'ylab back-traversal
3. Har `FlowCondition`, `FlowAssignment` da type'ni narrow qiladi
4. `FlowJoin`'da union qaytariladi
5. Type'ning cached natijasi `WeakMap<Node, Type>` da saqlanadi

**Distributive narrowing:**

Conditional type'lardagi distributive xulq narrowing'ga ham ta'sir qiladi. `T extends string ? T : never` har union member uchun alohida evaluate qilinadi:

```typescript
type Extract<T, U> = T extends U ? T : never;
// Extract<"a" | "b" | "c", "a" | "b"> 
//   = ("a" extends "a"|"b" ? "a" : never)
//   | ("b" extends "a"|"b" ? "b" : never)
//   | ("c" extends "a"|"b" ? "c" : never)
//   = "a" | "b" | never
//   = "a" | "b"
```

Narrowing va distributive conditional birga ishlatilsa, `Exclude`, `Extract`, `NonNullable` kabi utility type'lar implement qilinadi.

**`assertNever()` pattern:**

```typescript
function assertNever(value: never): never {
  throw new Error(`Unreachable: ${value}`);
}

switch (shape.kind) {
  case "circle": return ...;
  case "square": return ...;
  default: return assertNever(shape);
}
```

Bu pattern build time'da exhaustive check'ni majburlaydi. `shape: never` `assertNever()` ga assignable bo'lganda — barcha case'lar qoplagan. Yangi union member qo'shilsa — compile error.

**TS 5.5 inferred type predicates:**

```typescript
function isNonNull<T>(x: T | null): x is T {
  return x !== null;
}

// TS 5.5+ — predicate avtomatik infer qilinadi
function isNonNullV2<T>(x: T | null) {
  return x !== null;  // return type: x is T
}
```

Compiler funksiya body'sida narrowing pattern'ni topganda avtomatik `x is T` predicate hosil qiladi. `checker.ts` ichida `getTypePredicateFromBody()` shu pattern'ni aniqlaydi.

</details>

</details>

---

### Savol 23: TypeScript variance — covariance, contravariance, bivariance, invariance [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Variance — generic type parameter'ning subtype relation'ga ta'siri. **Covariance** (`out`): `T extends U` ⇒ `F<T> extends F<U>` (masalan `ReadonlyArray<Dog>` → `ReadonlyArray<Animal>`). **Contravariance** (`in`): teskari — `F<U> extends F<T>` (function parameter'lar). **Bivariance** — ikki yo'nalishda ham OK (method syntax, backward compat). **Invariance** — hech qaysi yo'nalishda assignable emas (mutable container).

### To'liq tushuntirish

Variance type system'da generic parameter'lar bilan subtype'lar orasidagi munosabatni belgilaydi.

Agar `Dog extends Animal`:

| Variance | Misol | Munosabat |
|----------|-------|-----------|
| **Covariant** | `ReadonlyArray<T>` | `ReadonlyArray<Dog>` → `ReadonlyArray<Animal>` |
| **Contravariant** | `(x: T) => void` | `(x: Animal) => void` → `(x: Dog) => void` |
| **Bivariant** | `{ method(x: T): void }` | Ikki yo'nalishda ham |
| **Invariant** | `Array<T>` (mutable) | Hech qaysi yo'nalishda emas |

**Covariance intuition:** `Animal[]` kerak bo'lsa, `Dog[]` yetadi — har element `Animal` (subtype relation saqlanadi). Lekin faqat o'qish uchun (`ReadonlyArray`).

**Contravariance intuition:** `(x: Dog) => void` slot kerak bo'lganda `(x: Animal) => void` callback yaroqli — slot har doim `Dog` uzatadi, callback esa `Animal` (shu jumladan `Dog`) qabul qiladi. Teskari yo'nalish xato: `(x: Animal) => void` slot ga `(x: Dog) => void` callback berib bo'lmaydi — slot ixtiyoriy `Animal` (masalan `Cat`) uzatishi mumkin, callback esa faqat `Dog` qabul qiladi. Parameter type subtype relation'ni teskari yo'nalishda saqlaydi — shuning uchun contravariant.

**Mutable container — invariance:** `Array<Dog>`'ga `Animal` push qilish mumkin emas (`Dog` array). `Array<Animal>` argument sifatida `Array<Dog>` yuborib `array.push(new Cat())` qilish runtime'da type system'ni buzar edi. Shuning uchun mutable `Array<T>` invariant.

### Kod misol

Covariance — `ReadonlyArray`:

```typescript
class Animal { eat() {} }
class Dog extends Animal { bark() {} }

function logAnimals(animals: ReadonlyArray<Animal>) {
  animals.forEach((a) => a.eat());
}

const dogs: ReadonlyArray<Dog> = [new Dog(), new Dog()];
logAnimals(dogs); // ✅ — ReadonlyArray<Dog> assignable to ReadonlyArray<Animal>
```

Contravariance — `strictFunctionTypes`:

```typescript
type AnimalHandler = (x: Animal) => void;
type DogHandler = (x: Dog) => void;

const handleAnimal: AnimalHandler = (a) => a.eat();
const handleDog: DogHandler = (d) => d.bark();

// strictFunctionTypes: true
let h1: DogHandler = handleAnimal;  // ✅ — function syntax, contravariant
let h2: AnimalHandler = handleDog;  // ❌ — Dog method'lari Animal'da yo'q
```

Bivariance — method syntax backward compat:

```typescript
interface BiContainer<T> {
  handle(x: T): void;  // ← method syntax, bivariant
}

interface CoContainer<T> {
  handle: (x: T) => void;  // ← function property, contravariant (strict)
}

// Farqni ko'rsatadigan yo'nalish: narrow param handler'ni
// wide param slot'ga berish. Bivariance ruxsat beradi, strict
// contravariance esa xato deb topadi.

// Method syntax — bivariant: ikki yo'nalishda ham qabul qiladi
let bi: BiContainer<Animal> = { handle: (d: Dog) => d.bark() }; // ✅ bivariant

// Function property — strictFunctionTypes contravariant
let co: CoContainer<Animal> = { handle: (d: Dog) => d.bark() };
//                              ❌ Type '(d: Dog) => void' is not assignable
//                              to '(x: Animal) => void'. (x: Dog) Animal'ni
//                              qabul qila olmaydi — narrow param wide slot'da

// Teskari yo'nalish ikkalasida ham OK (wide param wide/narrow slot'da):
let co2: CoContainer<Dog> = { handle: (a: Animal) => a.eat() }; // ✅
```

Invariance — mutable `Array`:

```typescript
const dogs: Dog[] = [new Dog()];

// ❌ Type 'Dog[]' is not assignable to type 'Animal[]'
//   Mutable Array<T> invariant — push(new Cat()) qilib bo'lmaydi
const animals: Animal[] = dogs;
```

TS 4.7 explicit variance annotation:

```typescript
interface Producer<out T> {     // covariant — faqat output
  produce(): T;
}

interface Consumer<in T> {       // contravariant — faqat input
  consume(x: T): void;
}

interface Bidirectional<in out T> {  // invariant
  transform(x: T): T;
}
```

### Edge Cases

- **`strictFunctionTypes: false`** — barcha function parametr'lar bivariant bo'ladi (backward compat)
- **Method syntax (`handle(x: T): void`)** — har doim bivariant, hatto `strictFunctionTypes: true` bilan ham
- **Function syntax (`handle: (x: T) => void`)** — `strictFunctionTypes`'ga bog'liq (contravariant strict mode'da)
- **Return type** — har doim covariant (return position'da)
- **Constructor parameter** — alohida — class constructor'lari maxsus qoidalar bilan

### Follow-up savollar

1. **"Nima uchun method syntax bivariant qoldirilgan?"** — Backward compat. TS 1.x'da DOM Event handler signature'lari (`addEventListener("click", handler)`) `(e: MouseEvent) => void` shaklida edi va shu signature'larni Library Event Map'ga assign qilish kerak edi. Strict contravariance qo'shilsa, butun DOM API buzilgan bo'lar edi.
2. **"`in out` annotation'larini qachon ishlatish kerak?"** — TS 4.7+ — generic type'ning variance'ini explicit belgilashga imkon beradi. Compiler avtomatik aniqlay olmagan murakkab generic'lar uchun foydali. Library yozuvchilar uchun documentation sifatida ham xizmat qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Variance checking algoritmi:**

`checker.ts` ichida `compareSignaturesRelated()` funksiyasi function signature'lar orasidagi assignability'ni hisoblaydi:

```
compareSignaturesRelated(source, target):
  // Return type — covariant
  if !isAssignableTo(source.returnType, target.returnType):
    return false

  // Parameter — contravariant (strictFunctionTypes)
  for i in params:
    if strictFunctionTypes:
      if !isAssignableTo(target.params[i], source.params[i]):  // ← reversed
        return false
    else:
      // Bivariant — ikki yo'nalishda
      if !isAssignableTo(source.params[i], target.params[i]) &&
         !isAssignableTo(target.params[i], source.params[i]):
        return false
```

**Variance inference (TS 4.7+):**

Compiler generic type'ning variance'ini avtomatik infer qiladi:

```typescript
interface Container<T> {
  produce(): T;        // T in output position → covariant evidence
  consume(x: T): void;  // T in input position → contravariant evidence
}
// Result: Container<T> invariant (ikki ko'rinish)
```

Compiler har generic parameter uchun ishlatilish pozitsiyasidan variance ni aniqlaydi:
- Faqat output position: covariant (`out`)
- Faqat input position: contravariant (`in`)
- Ikkalasi: invariant (`in out`)
- Hech biri (ishlatilmagan): unmeasurable — `unknown`/`never` substitution bilan probe qilinadi

`in`/`out` annotation explicit yozilsa, compiler infer qilingan variance annotation bilan mos kelishini ham tekshiradi.

**Soundness sabab:**

Variance qoidalari type system soundness uchun zarur. Aks holda runtime crash mumkin:

```typescript
// Hypothetical — bivariance mutable Array uchun yo'q
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;  // ❌ ruxsat berilsa (invariance buzilsa)...
animals.push(new Cat());          // animal[]'ga Cat push
dogs[1].bark();                   // 💥 Cat'da bark yo'q — runtime crash
```

Invariance bu scenario'ni compile-time'da to'sadi.

**Real-world variance issues:**

```typescript
// React props — covariant (read-only data)
type ComponentProps<T> = {
  data: T;
  render: (item: T) => React.ReactNode;
};
// data covariant, render contravariant — compiler har biri uchun alohida hisoblaydi

// Promise — covariant
const dogPromise: Promise<Dog> = Promise.resolve(new Dog());
const animalPromise: Promise<Animal> = dogPromise; // ✅ — Promise read-only
```

**TS 4.7 PR:** [microsoft/TypeScript#48240](https://github.com/microsoft/TypeScript/pull/48240) — `in`/`out` modifier'larini qo'shdi. Sabab: complex generic type'larda performance optimization (variance cache'ni qo'lda set qilish), explicit documentation.

</details>

</details>

---

## Xulosa

Bu bo'limdagi savollar TypeScript ning asosiy tushunchalari va compiler infrastructure'ni qamrab oldi:

- **TypeScript fundamentals** — typed superset, type erasure, structural typing, static vs dynamic typing
- **Compiler pipeline** — Scanner → Parser → Binder → Checker → Emitter
- **strict mode** — 8 ta sub-flag, migration strategiyalari
- **any vs unknown** — type safety farqlari va use case lar
- **File extensions** — `.ts`, `.tsx`, `.d.ts`, `.mts`, `.cts`
- **tsconfig.json** — asosiy flag lar, base config, project references
- **Playground** — real-time TS → JS compile, browserda `tsc`
- **Versiya tarixi** — `strictNullChecks` (2.0), conditional types (2.8), `unknown` (3.0), `as const` (3.4), template literal types (4.1), `satisfies` (4.9), TC39 decorators (5.0)
- **Bug fix patterns** — IDE/CLI version mismatch, strict migration, `skipLibCheck`, `allowJs`/`checkJs` chalkashligi
- **Compiler internals** — `getNarrowedType()`, control flow analysis, flow graph
- **Type variance** — covariance, contravariance, bivariance, invariance, `in`/`out` annotation'lari
