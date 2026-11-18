# Piano editoriale — typescript

Requisiti specifici dei capitoli di questa directory. Per le regole editoriali comuni e i criteri di accettazione, seguire [AGENTS.md](../../AGENTS.md).

## `06-typescript-core.md` — TypeScript Core

**Da coprire:**
- TypeScript vs JavaScript, superset, static analysis, type erasure, inference e runtime limits.
- Primitive types, array, tuple, optional properties, readonly, literal types, nullability.
- `any`, `unknown`, `never`, `void`; `strictNullChecks`; `null` vs `undefined` in typing.
- **`type`** e **`interface`**: estensione, intersection, declaration merging e criteri di scelta.
- **`enum`**: numeric, string, const enum, reverse mapping dove applicabile, implicazioni runtime e caveat di transpilation.
- Enum vs literal union (`as const` per oggetti costanti) e trade-off.
- Union, intersection, discriminated union, type narrowing, `in`, `typeof`, `instanceof`, user-defined type guards.
- Structural typing, compatibility, type assertion vs conversion reale.
- Object/function types e call signatures.

**Confronti/snippet:** `type` vs `interface` vs `enum`, `any` vs `unknown` vs `never`, type assertion vs guard, enum vs literal unions.

## `07-typescript-advanced.md` — TypeScript avanzato

**Da coprire:**
- Generic functions/classes/interfaces, generic constraints, defaults e type parameters.
- `keyof`, `typeof` (type query), indexed access types, `as const`, `satisfies`.
- Utility types: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `ReturnType`, `Parameters`, `Awaited`.
- Mapped types, conditional types, distributivity, `infer`, template literal types.
- Function overloads, generic inference, typed callbacks, variance/compatibility (overview Expert).
- Classes, abstract classes, access modifiers, `readonly`, interfaces vs abstract classes.
- Decorators: standard vs legacy semantics; differenze dai decorator Angular.
- `tsconfig.json`: `target`, `module`, `lib`, `strict`, `paths`, `moduleResolution`, declaration output.
- `.ts`, `.tsx`, `.d.ts`, ambient declarations, source maps, `isolatedModules`, bundler interactions.

**Confronti/snippet:** generic constraints, mapped/conditional types, `satisfies` vs assertion, overload vs union input.
