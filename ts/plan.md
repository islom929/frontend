# TS Course — Mavzular Ro'yxati

---

### QISM 1: TypeScript ASOSLARI

---

### `01-ts-intro.md` — TypeScript Nima va Nima Uchun
**Sub-sectionlar:**
- TypeScript nima — JavaScript superset, Microsoft tomonidan
- TypeScript vs JavaScript — type safety, tooling, refactoring
- TypeScript compiler (tsc) qanday ishlaydi — pipeline
- Compilation: TS source → Scanner → Parser → AST → Type Checker → Emitter → JS
- Type erasure — runtime da types yo'q
- Structural typing vs Nominal typing — TS structural ishlatadi (qisqacha intro, chuqur `25-type-compatibility.md` da)
- tsconfig.json asoslari — minimal setup
- `strict: true` nima qiladi — barcha strict flaglar ro'yxati
- TypeScript Playground — online sinab ko'rish
- `.ts` vs `.tsx` vs `.d.ts` vs `.mts` vs `.cts` farqlari
- TypeScript versiyalar tarixi — major milestones (2.0 strict, 3.0 project refs, 4.0 variadic tuples, 5.0 decorators)

**Keyingi bo'limga ilova:** → `02-primitive-types.md`

### `02-primitive-types.md` — Primitive Types va Type Annotations
**Sub-sectionlar:**
- Type annotations — `let name: string = "hello"`
- Type inference — TS o'zi aniqlaydi, qachon annotation kerak, qachon kerak emas
- Primitive types: `string`, `number`, `boolean` — JS dan farqi
- `null` va `undefined` — `strictNullChecks` bilan farqi
- `any` — nima uchun yomon, qachon ishlatish (migration da), implicit any
- `unknown` — `any` ning xavfsiz versiyasi, type guard kerak
- `never` — hech qachon bo'lmaydigan type (exhaustive check, throw funksiyalar)
- `void` — funksiya hech narsa qaytarmaydi
- `bigint` va `symbol` — qo'shimcha primitive types, unique symbol
- Literal types — `"hello"`, `42`, `true` — aniq qiymat type
- `as const` bilan literal type inference — widening ni to'xtatish
- Type assertions — `value as Type`, `<Type>value`, qachon ishlatish, xavflari
- Non-null assertion — `value!` (xavfli, qachon kerak)
- Double assertion — `value as unknown as TargetType` (nima uchun kerak, xavfi)
- Type widening va narrowing — TS qanday kengaytiradi/toraytiradi, `const` vs `let`

**Keyingi bo'limga ilova:** → `03-arrays-tuples-enums.md`

### `03-arrays-tuples-enums.md` — Arrays, Tuples va Enums
**Sub-sectionlar:**
- Array types — `string[]` vs `Array<string>` — farqi yo'q, convention
- Readonly arrays — `readonly string[]`, `ReadonlyArray<T>`
- Tuples — `[string, number]` — fixed-length
- Optional tuple elements — `[string, number?]`
- Rest elements — `[string, ...number[]]`
- Named tuples — `[name: string, age: number]` (documentation uchun)
- `as const` — readonly tuple yaratish, widening to'xtatish
- Enums — `enum Direction { Up, Down }` — numeric va string
- Const enums — `const enum` — inline replace, JS da yo'q
- Enums under the hood — IIFE ga compile bo'ladi, reverse mapping
- Enum pitfalls — reverse mapping xavflari, numeric enum shovqini
- Enum alternativalar — union literal types (`type Dir = "up" | "down"`)
- Qachon enum, qachon union type ishlatish — decision matrix
- `satisfies` bilan enum/union validation (qisqa eslatma, chuqur `06` da)

**Keyingi bo'limga ilova:** → `04-objects-interfaces.md`

---

### QISM 2: TYPE SYSTEM CHUQUR

---

### `04-objects-interfaces.md` — Object Types va Interfaces
**Sub-sectionlar:**
- Object type annotations — `{ name: string; age: number }`
- Optional properties — `age?: number`
- Readonly properties — `readonly id: number`
- Index signatures — `[key: string]: number` — dynamic keys
- `Record` vs index signature — qachon qaysi biri (qisqa taqqoslash)
- Interface declaration — `interface User { ... }`
- Interface extending — `interface Admin extends User { ... }`
- Multiple interface extending — `extends A, B`
- Interface merging (declaration merging) — bir xil nomli interface lar birlashadi
- Interface vs Type Alias — qachon qaysi biri, farqlari jadvali
  - Interface: extend, merge, class implements
  - Type: union, intersection, mapped, conditional, primitives
- Excess property checking — literal object da qo'shimcha property xato
- Workaround: variable ga assign, type assertion, index signature
- Recursive interfaces — daraxt strukturalar
- `PropertyKey` type — `string | number | symbol`

**Keyingi bo'limga ilova:** → `05-unions-intersections.md`

### `05-unions-intersections.md` — Type Aliases va Union/Intersection
**Sub-sectionlar:**
- Type aliases — `type UserID = string | number`
- Union types (`|`) — "bu yoki u" — `string | number`
- Narrowing union types — `typeof`, `in`, `instanceof`
- Intersection types (`&`) — "ham bu, ham u" — type birlashtirish
- Intersection bilan conflict — `{a: string} & {a: number}` → `never`
- Discriminated Unions (Tagged Unions) — `type` field bilan
  - `type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number }`
  - `switch(shape.kind)` — exhaustive check
- Custom type guards — `function isCircle(s: Shape): s is Circle`
- `in` operator type guard — `"radius" in shape`
- `instanceof` type guard — class instances
- `typeof` type guard — primitive types
- Exhaustive checking — `never` bilan — switch default da
- `assertNever` pattern — compile-time xato
- Union types va function overloads — qachon qaysi biri

**Keyingi bo'limga ilova:** → `06-type-narrowing.md`

### `06-type-narrowing.md` — Type Narrowing va Control Flow
**Sub-sectionlar:**
- Type narrowing nima — union type ni aniqroq type ga toraytirish
- Control flow analysis — TS compiler kodni tahlil qiladi
- Truthiness narrowing — `if (value)` — falsy values
- Equality narrowing — `if (a === b)` — ikkalasi bir type
- `typeof` guards — `if (typeof x === "string")`
- `instanceof` guards — `if (x instanceof Date)`
- `in` operator — `if ("name" in obj)`
- Assignment narrowing — `x = "hello"` — x string bo'ladi
- User-defined type guards — `is` keyword bilan funksiya
- Assertion functions — `asserts value is string` — throw qiladi yoki true
- `satisfies` operator (TS 5.0+) — type check qiladi lekin type ni saqlab qoladi
  - `const config = { ... } satisfies Config` — literal type saqlanadi
  - `satisfies` vs `as const` vs type annotation — farqlari
- Control flow va closures — closure ichida narrowing yo'qolishi
- Discriminated unions va narrowing — eng kuchli pattern
- Narrowing limitations — callback ichida, async funksiyalarda

**Keyingi bo'limga ilova:** → `07-functions.md`

---

### QISM 3: FUNCTIONS VA GENERICS

---

### `07-functions.md` — Functions TypeScript da
**Sub-sectionlar:**
- Function type annotations — `function greet(name: string): string`
- Return type annotation va inference — qachon yozish kerak
- Optional parameters — `function greet(name?: string)`
- Default parameters — `function greet(name: string = "World")`
- Rest parameters — `function sum(...nums: number[])`
- Function overloads — bir nechta signature + implementation
  - Overload signatures
  - Implementation signature (uni ko'rmaydi)
  - Overload va union type farqi — qachon qaysi biri
  - Overload resolution order — aniqroq birinchi
- Call signatures — object type da funksiya
- Construct signatures — `new` bilan chaqirish
- `this` parameter — `function handler(this: HTMLElement, e: Event)`
- `void` vs `undefined` — callback da farqi
- Function type expressions — `type Callback = (data: string) => void`
- Generic functions — `function identity<T>(arg: T): T` (intro, chuqur `08` da)
- Callback types — event handlers, Promise callbacks
- Assignability of functions — parameter bivariance, `strictFunctionTypes`

**Keyingi bo'limga ilova:** → `08-generics.md`

### `08-generics.md` — Generics
**Sub-sectionlar:**
- Generics nima — type parameter, "type o'zgaruvchisi"
- Generic functions — `function identity<T>(arg: T): T`
- Generic inference — TS o'zi T ni aniqlaydi
- Explicit type argument — `identity<string>("hello")`
- Generic interfaces — `interface Box<T> { value: T }`
- Generic classes — `class Stack<T> { ... }`
- Generic constraints — `<T extends HasLength>` — T ni cheklash
- `keyof` bilan generics — `<T, K extends keyof T>`
- Index Access Types — `T[K]` — property type olish
  - `type NameType = User["name"]` — lookup type
  - `T[keyof T]` — barcha value types union
  - `T[number]` — array element type
- `typeof` type operator — qiymatdan type olish
  - `typeof someVariable` — type context da
  - `ReturnType<typeof someFunction>` — kombinatsiya
- Multiple type parameters — `<T, U>`
- Default type parameters — `<T = string>`
- `const` type parameters (TS 5.0) — `function foo<const T>(arg: T)` — literal type inference
  - `as const` ga muqobil — funksiya chaqiruvida
  - Qachon ishlatish kerak
- Generic utility patterns:
  - List/Collection
  - Result/Option
  - Repository
  - Factory

**Keyingi bo'limga ilova:** → `09-advanced-generics.md`

### `09-advanced-generics.md` — Advanced Generics
**Sub-sectionlar:**
- Conditional types — `T extends string ? "yes" : "no"` (intro, chuqur `12` da)
- `infer` keyword — type ichidan type "olish"
  - `T extends Array<infer U> ? U : never`
  - `T extends (...args: infer P) => infer R ? ...`
- Distributive conditional types — union bilan qanday ishlaydi
  - `ToString<string | number>` = `ToString<string> | ToString<number>`
- Non-distributive — `[T] extends [string]`
- Mapped types — `{ [K in keyof T]: T[K] }` (intro, chuqur `13` da)
  - Property modifier: `+?`, `-?`, `+readonly`, `-readonly`
  - Key remapping: `as`
- Template literal types — `` `${string}Changed` `` (intro, chuqur `14` da)
  - String manipulation: `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`
- Recursive types — `type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> }`
- Variadic tuple types — `[...T, ...U]`
  - `function concat<T extends unknown[], U extends unknown[]>(a: T, b: U): [...T, ...U]`
- Generic constraints best practices — over-constraining, under-constraining
- Real-world patterns — API typing, state management, ORM
- Higher-kinded types — TS da cheklovlar, workaround patterns

**Keyingi bo'limga ilova:** → `10-classes.md`

---

### QISM 4: OOP TypeScript DA

---

### `10-classes.md` — Classes TypeScript da
**Sub-sectionlar:**
- Class anatomy — JS class + TS qo'shimchalar
- Type annotations — properties, constructor, methods
- Access modifiers:
  - `public` — default, hamma joydan
  - `private` — faqat class ichida
  - `protected` — class va subclass
- `readonly` modifier — faqat constructor da set
- Parameter properties — `constructor(public name: string)` — shorthand
- Static members — `static count: number`
- Static initialization blocks (ES2022) — `static { ... }` class ichida
- Abstract classes — `abstract class Animal { abstract speak(): void }`
- Class implements interface — `class User implements IUser`
  - Implements faqat shape tekshiradi, type narrowing bermaydi
- Class extends class — inheritance + `super`
- `override` keyword (TS 4.3+) — parent method ni qayta yozish
  - `noImplicitOverride` tsconfig option
- `this` type — method chaining, fluent interfaces
- Generics bilan classes — `class Repository<T> { ... }`
- `accessor` keyword (TS 4.9+) — auto-accessor fields
- Classes at runtime — JS ga compile, propertilar saqlanadi, access modifiers yo'qoladi

**Keyingi bo'limga ilova:** → `11-advanced-oop.md`

### `11-advanced-oop.md` — Advanced OOP Patterns
**Sub-sectionlar:**
- Mixins — TypeScript da ko'p meros
  - Mixin function pattern
  - `applyMixins` helper
  - Type-safe mixin declaration
- Abstract factory — generic factory
- Multiple interfaces — class bir nechta interface implement
- `#` (ES private) vs `private` keyword — runtime vs compile-time
  - `#` — WeakMap ga compile bo'ladi (real private)
  - `private` — type check da ishlaydi, runtime da ochiq
- Class expressions — `const MyClass = class { ... }`
- `this` as return type — fluent/chainable API
- Builder pattern — step-by-step object yaratish (batafsil `21` da)
- Composition vs Inheritance — TS da qanday implement
  - Composition bilan interface + injection
  - Real-world example
- Intersection types va classes — class type ni kengaytirish
- `satisfies` bilan class property validation

**Keyingi bo'limga ilova:** → `12-conditional-types.md`

---

### QISM 5: ADVANCED TYPES

---

### `12-conditional-types.md` — Conditional Types Chuqur
**Sub-sectionlar:**
- Conditional type syntax — `T extends U ? X : Y`
- Nima uchun kerak — type logic, type-level if/else
- `infer` keyword chuqur:
  - Return type olish — `ReturnType` qanday ishlaydi
  - Parameter types olish — `Parameters` qanday ishlaydi
  - Promise unwrap — `Awaited` qanday ishlaydi
  - Array element type — `T extends (infer U)[] ? U : never`
  - Tuple elements — first, last, rest
  - `infer` bilan constraints — `infer U extends string` (TS 4.7+)
- Distributive conditional types:
  - Qanday ishlaydi — naked type parameter
  - `Exclude`, `Extract` qanday ishlaydi
  - Non-distributive qilish — `[T] extends [U]`
- Recursive conditional types — string manipulation, deep types
- Conditional types va function overloads — qachon qaysi biri
- Real-world use cases — overloaded return types, type filtering
- Conditional types limitations — tsc performance, depth limits

**Keyingi bo'limga ilova:** → `13-mapped-types.md`

### `13-mapped-types.md` — Mapped Types Chuqur
**Sub-sectionlar:**
- Mapped type syntax — `{ [K in keyof T]: T[K] }`
- `keyof` operator — object type dan key union
- Index Access Types — `T[K]` property type olish (tie-in `08` bilan)
- Property modifiers:
  - `readonly` qo'shish/olib tashlash — `+readonly`, `-readonly`
  - `?` qo'shish/olib tashlash — `+?`, `-?`
- `Partial`, `Required`, `Readonly`, `Record` — ichidan qanday ishlaydi
- Key remapping — `as` keyword (TS 4.1+)
  - `{ [K in keyof T as NewKey]: T[K] }`
  - Filtering keys — `as K extends "name" ? K : never`
  - Template literal bilan — `` as `get${Capitalize<string & K>}` ``
- Mapped types + conditional types — qudratli kombinatsiya
- Custom mapped types:
  - `Getters<T>` — barcha propertylar uchun getter
  - `EventMap<T>` — event handlers
  - `Validators<T>` — validation funksiyalar
- Mapped types va union types — distributive behavior
- Homomorphic mapped types — structure saqlovchi (Partial, Readonly)
- Performance — murakkab mapped types va compiler

**Keyingi bo'limga ilova:** → `14-template-literal-types.md`

### `14-template-literal-types.md` — Template Literal Types
**Sub-sectionlar:**
- Template literal type syntax — `` type Greeting = `Hello ${string}` ``
- String manipulation types — `Uppercase<T>`, `Lowercase<T>`, `Capitalize<T>`, `Uncapitalize<T>`
- Pattern matching — `` T extends `${infer First}.${infer Rest}` ? ... ``
- Union distribution — `` type Events = `${"mouse" | "key"}${"up" | "down"}` ``
  - = `"mouseup" | "mousedown" | "keyup" | "keydown"`
- Type-safe event systems:
  - `` on(event: `${string}Changed`, handler: () => void) ``
  - Event name dan handler type aniqlash
- Type-safe routing:
  - Route parameter extraction — `/users/:id/posts/:postId`
  - Type-safe route params
- Template literals + mapped types:
  - Getter/setter generation
  - Event handler generation
- Real-world patterns:
  - CSS class names
  - API endpoint typing
  - i18n key validation
  - SQL query builder types
  - dotted path types (`"user.profile.name"`)

**Keyingi bo'limga ilova:** → `15-utility-types.md`

---

### QISM 6: UTILITY TYPES VA TYPE MANIPULATION

---

### `15-utility-types.md` — Built-in Utility Types
**Sub-sectionlar:**

Har bir utility type uchun: nima qiladi + implementation ichidan + real-world use case + gotchas.

- `Partial<T>` — barcha optional
- `Required<T>` — barcha required
- `Readonly<T>` — barcha readonly
- `Record<K, V>` — key-value mapping
- `Pick<T, K>` — tanlangan propertylar
- `Omit<T, K>` — tanlangan propertylarni olib tashlash
- `Exclude<T, U>` — union dan chiqarish
- `Extract<T, U>` — union dan ajratish
- `NonNullable<T>` — null/undefined chiqarish
- `ReturnType<T>` — funksiya return type
- `Parameters<T>` — funksiya parametrlari tuple
- `ConstructorParameters<T>` — constructor parametrlari
- `InstanceType<T>` — class instance type
- `Awaited<T>` — Promise unwrap (recursive)
- `ThisParameterType<T>` va `OmitThisParameter<T>`
- `NoInfer<T>` (TS 5.4+) — inference to'xtatish
  - Qachon kerak — default value inference ni cheklash
  - Real-world pattern — event emitter, configuration
- Utility types kombinatsiyasi — bir nechta birga ishlatish
- Utility types performance — chuqur nesting ogohlantirish

**Keyingi bo'limga ilova:** → `16-custom-utility-types.md`

### `16-custom-utility-types.md` — Custom Utility Types Yaratish
**Sub-sectionlar:**
- Nima uchun custom utility types kerak
- `DeepPartial<T>` — recursive Partial
- `DeepReadonly<T>` — recursive Readonly
- `DeepRequired<T>` — recursive Required
- `Mutable<T>` — Readonly ni teskari
- `Nullable<T>` — `T | null`
- `ValueOf<T>` — object values union
- `Flatten<T>` — nested type ni tekislash
- `PathKeys<T>` — `"user.name.first"` tipidagi nested keylar
- `PathValue<T, P>` — nested key bo'yicha type olish
- `Brand<T, B>` / `Opaque<T, B>` — branded/nominal types
  - `type UserId = Brand<number, "UserId">`
  - Nima uchun kerak — `userId` va `postId` ni aralashtirib yubormaslik
- Type-level programming patterns:
  - Type-level arithmetic (Recursion bilan)
  - String parsing (Template literals + Recursion)
  - Tuple manipulation (Head, Tail, Concat, Reverse)
  - Type-level state machines
- Performance — murakkab type lar va `tsc` tezligi
  - Type instantiation depth limit
  - Recursive type optimization
  - Tail-call optimization for recursive types (TS 4.5+)

**Keyingi bo'limga ilova:** → `17-modules.md`

---

### QISM 7: MODULES VA NAMESPACES

---

### `17-modules.md` — Module System TypeScript da
**Sub-sectionlar:**
- ES Modules TypeScript da — `import`/`export` + types
- `import type` va `export type` — type-only imports (TS 3.8+)
  - `import type { User } from "./user"`
  - Nima uchun kerak — bundle size, circular deps
- `import { type User }` — inline type import (TS 4.5+)
- `verbatimModuleSyntax` (TS 5.0+) — yangi standart
  - `import type` ni majburiy qiladi
  - `importsNotUsedAsValues` va `preserveValueImports` ni almashtiradi
  - Nima uchun yaxshiroq — aniq va predictable
- Module resolution strategies:
  - `node` — Node.js klasik (CJS)
  - `node16` / `nodenext` — ESM support
  - `bundler` — Webpack/Vite uchun (TS 5.0+)
  - Farqlari — qaysi holat da qaysi biri
- `moduleDetection` — `auto`, `force`, `legacy`
- `isolatedModules` — fayl-bo'yicha transpilation uchun
  - Nima cheklovlar qo'yadi — const enum, namespace merge
  - Nima uchun kerak — esbuild, SWC, Babel
- Path aliases — `paths` + `baseUrl` tsconfig da
  - `"@components/*": ["src/components/*"]`
- Barrel exports — `index.ts` pattern
  - Afzalliklari va muammolari (tree shaking)
- Ambient modules — `declare module "*.css"`
- Module augmentation — mavjud module type larini kengaytirish
  - `declare module "express" { interface Request { userId: string } }`
- Global augmentation — `declare global { ... }`
- Dynamic imports va type-safety — `const mod = await import("./module")`
- Circular dependencies va types — type-only import yechim
- `.mts` / `.cts` fayl kengaytmalari — ESM/CJS aniq belgilash

**Keyingi bo'limga ilova:** → `18-declaration-files.md`

### `18-declaration-files.md` — Declaration Files (.d.ts)
**Sub-sectionlar:**
- Declaration files nima — JS uchun type information
- `.d.ts` vs `.ts` — faqat type, implementation yo'q
- `declare` keyword:
  - `declare const`, `declare function`, `declare class`
  - `declare module` — module declarations
  - `declare global` — global scope ga qo'shish
  - `declare namespace` — namespace declarations
- Declaration file yaratish — qo'lda yoki avtomatik
- `declaration: true` — tsc avtomatik generate
- `declarationMap: true` — source map for declarations
- `isolatedDeclarations` (TS 5.5+) — explicit return types, tezkor declaration emit
  - Nima uchun kerak — monorepo performance
  - Qanday ishlaydi — fayl-bo'yicha declaration emit
- DefinitelyTyped — `@types/*` packages
  - `@types/node`, `@types/react`, `@types/express`
  - Versiya moslashtirish
- `typeRoots` va `types` tsconfig da
- Ambient declarations — JS kutubxonalar uchun type yozish
  - UMD library
  - Global library
  - Module library
- Triple-slash directives — `/// <reference types="..." />`
- Declaration file testing — `dtslint`, `tsd`, `@arethetypeswrong/cli`

**Keyingi bo'limga ilova:** → `19-decorators.md`

---

### QISM 8: DECORATORS, METADATA VA PATTERNS

---

### `19-decorators.md` — Decorators
**Sub-sectionlar:**
- Decorators nima — meta-programming, annotation
- Ikki standart:
  - Legacy decorators — `experimentalDecorators: true` (Angular, NestJS)
  - TC39 Stage 3 Decorators — yangi standart (TS 5.0+)
- Legacy Decorators:
  - Class decorator — `@sealed class Foo {}`
  - Method decorator — `@log greet() {}`
  - Property decorator — `@required name: string`
  - Parameter decorator — `greet(@inject name: string)`
  - Accessor decorator — `@configurable get name() {}`
- TC39 Decorators:
  - Syntax farqi — legacy dan qanday farq qiladi
  - `context` parameter — `ClassMethodDecoratorContext`, `ClassFieldDecoratorContext`
  - Metadata API — `Symbol.metadata`
  - `accessor` keyword bilan — auto-accessor decorators
  - Cheklovlar — parameter decorators hali yo'q
- Decorator factories — `@log("verbose")` — parametrli
- Decorator composition — `@a @b @c` — tartib (pastdan yuqoriga)
- Decorator execution order — class da qanday tartibda ishlaydi
- Real-world examples:
  - `@sealed` — class ni extend qilish mumkin emas
  - `@log` — method chaqiruvlarni log qilish
  - `@validate` — input validation
  - `@memoize` — result caching
  - `@injectable` — DI uchun
  - `@debounce` — method debouncing
  - `@deprecated` — warning log

**Keyingi bo'limga ilova:** → `20-reflect-metadata-di.md`

### `20-reflect-metadata-di.md` — Reflect Metadata va DI
**Sub-sectionlar:**
- Reflect Metadata API — `reflect-metadata` package
- `Reflect.defineMetadata`, `Reflect.getMetadata`
- `emitDecoratorMetadata` — tsc metadata emit qiladi
  - `design:type`, `design:paramtypes`, `design:returntype`
- Dependency Injection nima — IoC (Inversion of Control)
- DI container yaratish — step-by-step
  - `@injectable` decorator
  - `@inject` decorator
  - Container class — resolve dependencies
  - Scope: singleton, transient, request
- InversifyJS — popular TS DI container
- TSyringe — lightweight alternative
- NestJS DI system — qanday ishlaydi
  - `@Module`, `@Injectable`, `@Inject`
  - Provider types — useClass, useValue, useFactory
- DI va testing — mock qilish osonligi
- DI patterns — constructor injection, property injection, method injection
- Decorator-based vs function-based DI — zamonaviy trend

**Keyingi bo'limga ilova:** → `21-design-patterns.md`

### `21-design-patterns.md` — Design Patterns TypeScript da
**Sub-sectionlar:**
- Har bir pattern uchun TS-specific implementation

**Creational Patterns:**
- Factory Pattern — generic factory, type inference
- Abstract Factory — product families
- Singleton Pattern — private constructor, static instance
- Builder Pattern — fluent API, method chaining, `this` return type
  - Conditional steps — required fields enforce (compile-time)
  - Step builder — har bir qadamda faqat kerakli methodlar

**Structural Patterns:**
- Adapter Pattern — interface conversion
- Facade Pattern — murakkab API ni soddalash
- Proxy Pattern — access control, lazy loading
- Decorator Pattern (OOP) — runtime extension

**Behavioral Patterns:**
- Observer Pattern — typed event emitter, generics bilan
  - `EventEmitter<Events extends Record<string, unknown[]>>`
- Strategy Pattern — interface + dependency injection
- Command Pattern — typed command + handler
- State Machine:
  - Discriminated unions bilan state definition
  - Type-safe transitions
  - xstate integration
- Mediator Pattern — component communication
- Chain of Responsibility — middleware pipeline typing

**Cross-cutting:**
- Repository Pattern — generic CRUD
  - `interface Repository<T> { findById(id: string): Promise<T | null> }`
- Result/Either Pattern:
  - `type Result<T, E> = Ok<T> | Err<E>`
  - Railway-oriented programming
  - Error handling without exceptions
  - neverthrow library

**Keyingi bo'limga ilova:** → `22-tsconfig.md`

---

### QISM 9: TOOLING VA ECOSYSTEM

---

### `22-tsconfig.md` — tsconfig.json Mastery
**Sub-sectionlar:**
- Compiler options to'liq:
  - **Strict family:**
    - `strict` — barcha strict flaglarni yoqadi
    - `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`
    - `strictPropertyInitialization`, `noImplicitAny`, `noImplicitThis`
    - `alwaysStrict`, `useUnknownInCatchVariables`
  - **Target va Lib:**
    - `target` — `ES5`, `ES2015`, ..., `ESNext`
    - `lib` — qaysi global types mavjud
    - `target` vs `lib` farqi — qachon ajratib sozlash kerak
  - **Module:**
    - `module` — `commonjs`, `esnext`, `node16`, `nodenext`, `preserve`
    - `moduleResolution` — `node`, `node16`, `bundler`
    - `esModuleInterop`, `allowSyntheticDefaultImports`
    - `verbatimModuleSyntax` (TS 5.0+) — yangi standart
    - `isolatedModules` — bundler compatibility
    - `moduleDetection` — `auto`, `force`, `legacy`
  - **Paths:**
    - `baseUrl`, `paths` — import aliases
    - `rootDir`, `outDir` — source/output directories
    - `rootDirs` — virtual directories
  - **Declaration:**
    - `declaration`, `declarationMap`, `emitDeclarationOnly`
    - `sourceMap`, `inlineSourceMap`
    - `isolatedDeclarations` (TS 5.5+) — tezkor declaration emit
  - **Checking:**
    - `noUnusedLocals`, `noUnusedParameters`
    - `noImplicitReturns`, `noFallthroughCasesInSwitch`
    - `exactOptionalPropertyTypes` — `undefined` vs missing property
    - `noUncheckedIndexedAccess` — index signature xavfsizligi
    - `noImplicitOverride` — override keyword majburiy
  - **Emit:**
    - `noEmit` — faqat type check
    - `erasableSyntaxOnly` (TS 5.8+) — faqat o'chirib tashlanadigan syntax
    - `removeComments`, `stripInternal`
    - `downlevelIteration` — for-of, spread uchun
  - **Interop:**
    - `allowJs`, `checkJs` — JS fayllar bilan ishlash
    - `skipLibCheck` — declaration files ni tekshirmaslik (performance)
    - `forceConsistentCasingInFileNames` — fayl nomlari katta-kichik harf
- `include`, `exclude`, `files` — qaysi fayllar compile bo'ladi
- `extends` — tsconfig inheritance
  - `@tsconfig/recommended`, `@tsconfig/node20` kabi presetlar
- Project references — `references`, `composite`
  - Monorepo da foydalanish
  - `tsc --build` — incremental builds
- Incremental builds — `incremental`, `tsBuildInfoFile`
- ESLint + TypeScript:
  - `@typescript-eslint/parser`
  - `@typescript-eslint/eslint-plugin`
  - `typescript-eslint` v8 — flat config
  - Recommended rules
- tsconfig best practices — project type bo'yicha:
  - React app (Vite, Next.js)
  - Node.js API
  - Library/Package
  - Monorepo

**Keyingi bo'limga ilova:** → `23-type-safe-patterns.md`

### `23-type-safe-patterns.md` — Type-Safe Patterns
**Sub-sectionlar:**
- Branded/Opaque types — nominal typing TS da
  - `type UserId = number & { __brand: "UserId" }`
  - Constructor function — `function createUserId(id: number): UserId`
  - Runtime validated brands — Zod + Brand
- Exhaustive pattern matching — `switch` + `never`
- `const` assertions — `as const` — immutable va literal types
- Type-safe builders — compile-time required field checking
- Type-safe event emitters — generic event map
- Runtime validation + types:
  - Zod — schema → type
  - io-ts — fp-ts bilan
  - Valibot — lightweight
  - `z.infer<typeof schema>` pattern
  - ArkType — type-first validation
- Error handling patterns:
  - Result type — `Ok<T> | Err<E>`
  - Typed exceptions
  - neverthrow library
  - Effect-TS — functional error handling

**Keyingi bo'limga ilova:** → `24-migration-tooling.md`

### `24-migration-tooling.md` — Migration va Advanced Tooling
**Sub-sectionlar:**
- JS → TS migration strategiyasi:
  1. `allowJs: true` — aralash project
  2. `checkJs: true` — JS fayllarni tekshirish
  3. `.js` → `.ts` bosqichma-bosqich
  4. Strict mode ga o'tish — `strict: true` ga qadam-baqadam
- JSDoc annotations — TypeScript siz type checking
  - `@param`, `@returns`, `@type`, `@typedef`
  - VS Code da JSDoc IntelliSense
- Pragma comments:
  - `@ts-ignore` — keyingi qator xatosini yashirish
  - `@ts-expect-error` — xato bo'lishi kerak (test uchun)
  - `@ts-nocheck` — butun faylni tekshirmaslik
  - `@ts-check` — JS faylda type checking yoqish
- Performance — tsc compilation tezligi:
  - `incremental: true`
  - `skipLibCheck: true`
  - Project references
  - `isolatedModules`
  - `isolatedDeclarations` (TS 5.5+) — tezkor declaration emit
- Runtime execution:
  - `ts-node` — runtime transpilation
  - `tsx` — esbuild based, tezroq
  - `tsc --watch` — development workflow
  - Node.js `--experimental-strip-types` (22+) — native TS support
- Tezkor transpilation:
  - SWC — Rust-based, tsc dan 20x tez
  - esbuild — Go-based, tezkor bundler
  - Faqat transpilation — type checking yo'q, `isolatedModules` kerak
- Zamonaviy toollar:
  - Biome — linter + formatter (ESLint + Prettier o'rniga)
  - oxlint — Rust-based linter
  - Bun — runtime + bundler + package manager (native TS)
  - Deno — native TS support
- Type coverage — `type-coverage` package
  - Any usage tracking
  - Migration progressini o'lchash

**Keyingi bo'limga ilova:** → `25-type-compatibility.md`

### `25-type-compatibility.md` — Type Compatibility va Variance
**Sub-sectionlar:**
- Structural typing — TS ning asosiy type checking mexanizmi
  - Nominal vs Structural — boshqa tillar bilan taqqoslash (Java nominal, TS structural)
  - "Duck typing" — agar shape mos kelsa, type mos keladi
  - Excess property checking — faqat literal objects da
- Type compatibility rules:
  - Object compatibility — kerakli propertylar bor bo'lsa mos
  - Function parameter compatibility — kamroq parameter mos (callback safety)
  - Return type compatibility — subtype mos
  - Optional va required — compatibility qoidalari
- **Covariance** — type hierarchy bir yo'nalishda
  - Array `T[]` — covariant (agar `Dog extends Animal`, `Dog[] → Animal[]` mos)
  - Return types — covariant (subtype qaytarish mumkin)
  - `readonly` arrays — xavfsiz covariance
- **Contravariance** — type hierarchy teskari yo'nalishda
  - Function parameters — contravariant (`strictFunctionTypes` bilan)
  - Callback parameters — `(animal: Animal) => void` → `(dog: Dog) => void` mos emas
- **Bivariance** — ikki yo'nalishda ham (xavfli)
  - Method parameters — bivariant (TS da method shorthand)
  - `strictFunctionTypes` — function property ni contravariant qiladi
  - Method shorthand vs function property — `foo(x: T)` vs `foo: (x: T) => void`
- **Invariance** — hech qanday yo'nalishda mos kelmaydi
  - Mutable data structures — invariant bo'lishi kerak (lekin TS da covariant)
  - `in`/`out` modifiers (TS 4.7+) — explicit variance annotations
    - `interface Producer<out T> { get(): T }` — covariant
    - `interface Consumer<in T> { accept(value: T): void }` — contravariant
    - `interface Processor<in out T> { process(value: T): T }` — invariant
- Function compatibility chuqur:
  - Parameter count — kamroq mos (excess callback params ignored)
  - Parameter types — contravariant (strict mode da)
  - Return types — covariant
  - Rest parameters va overloads
- Enum compatibility — numeric enum xavflari
- Class compatibility — structural, private/protected ta'siri
- Generic compatibility — constraints va inference
- Real-world ahamiyati:
  - Nima uchun ba'zi type errorlar tushunarsiz — variance tufayli

**Keyingi bo'limga ilova:** → `26-ts-5x-features.md`

### `26-ts-5x-features.md` — TypeScript 5.x Yangiliklari
**Sub-sectionlar:**

Har bir feature uchun: nima, nima uchun, oldin qanday edi, endi qanday, code example.

**TypeScript 5.0:**
- TC39 Stage 3 Decorators — yangi standart (batafsil `19` da)
- `const` type parameters — `<const T>` (batafsil `08` da)
- `satisfies` operator — TS 4.9 da qo'shilgan, 5.0 da keng tarqaldi (batafsil `06` da)
- `verbatimModuleSyntax` — module import/export aniqlik (batafsil `17` da)
- `moduleResolution: bundler` — Vite/Webpack uchun
- All enums are union enums — enum behavior yaxshilandi
- `--moduleResolution bundler`
- `@satisfies` va `@overload` JSDoc tags

**TypeScript 5.1:**
- Easier implicit returns for `undefined` — `undefined` qaytaruvchi funksiyalar
- Unrelated types for getters/setters — getter va setter turli type bo'lishi mumkin
- JSX improvements — custom JSX elements
- `typeRoots` consulted for `/// <reference types="..." />`

**TypeScript 5.2:**
- `using` declarations — Explicit Resource Management
  - `using file = openFile()` — auto-dispose
  - `await using db = connectDB()` — async dispose
  - `Symbol.dispose` va `Symbol.asyncDispose`
  - `DisposableStack` va `AsyncDisposableStack`
  - Real-world: file handles, DB connections, locks
- Decorator metadata — `Symbol.metadata`

**TypeScript 5.3:**
- Import Attributes — `import data from "./data.json" with { type: "json" }`
- `resolution-mode` in import types
- `switch(true)` narrowing

**TypeScript 5.4:**
- `NoInfer<T>` utility type (batafsil `15` da)
- Preserved narrowing in closures — closure ichida narrowing saqlanadi (ba'zi holatlarda)
- `Object.groupBy` va `Map.groupBy` types

**TypeScript 5.5:**
- `isolatedDeclarations` — tezkor declaration emit (batafsil `18` va `22` da)
  - Explicit return types majburiy
  - Monorepo performance uchun
- Inferred type predicates — TS avtomatik type guard aniqlaydi
  - `array.filter(x => x !== null)` — endi to'g'ri type beradi
- Regular expression syntax checking — regex da type check
- `@import` tag in JSDoc

**TypeScript 5.6:**
- Disallowed Nullish and Truthy Checks — `if (Promise)` kabi xatolarni topadi
- Iterator helper methods — `Iterator` va `IterableIterator` types
- `--noUncheckedSideEffectImports` — side effect import larni tekshirish

**TypeScript 5.7:**
- Path rewriting for relative paths — `.ts` → `.js` output da
- `--target es2024` — `SharedArrayBuffer`, `Atomics.waitAsync`
- Checked Imports — `--noUncheckedSideEffectImports` yaxshilandi

**TypeScript 5.8:**
- `--erasableSyntaxOnly` — faqat erasable syntax (enum, namespace kabi)
  - Node.js `--experimental-strip-types` bilan ishlatish uchun
  - `const enum` va `namespace` ishlamaydi
- `--libReplacement` — custom lib definitions
- Granular checks for branches in return expressions

**Version selection guide:**
- Qaysi versiya qaysi project uchun mos
- Breaking changes — versiyalar orasida
- Migration tips — eski versiyadan yangisiga

