# TypeScript types

## Type system usage

### No `any`, use `unknown` for unknown types [Recommended]

`any` disables checking and spreads along the call chain. When a type is unknown, use `unknown` and narrow before use. Do not use `{}` either: it means any non-nullish value.

**Exceptions:** Local use is allowed when building test doubles.

**Tooling:** `@typescript-eslint/no-explicit-any`, `no-unsafe-assignment`, `no-unsafe-member-access`.

**Source:** Google TypeScript Style Guide, typescript-eslint Docs

### Use `@ts-expect-error`, not `@ts-ignore` [Recommended]

After the problem is fixed, `@ts-expect-error` reports an error as a reminder to delete it, while `@ts-ignore` stays forever. Write the reason after the directive.

**Tooling:** `@typescript-eslint/ban-ts-comment`. For configuration, see [style.md](style.md).

**Source:** Google TypeScript Style Guide, typescript-eslint Docs

### Narrow with runtime checks, not assertions [Recommended]

Prefer `typeof`, `instanceof`, `in`, discriminant fields, and equality comparison. `as` and `!` produce no runtime check, so errors are left until runtime. Do not bypass checks with `as unknown as T`.

**Example:**

Avoid:

```ts
function readName(value: unknown): string {
  return (value as User).name!;
}
```

Prefer:

```ts
function readName(value: unknown): string {
  if (typeof value === 'object' && value !== null && 'name' in value && typeof value.name === 'string') {
    return value.name;
  }
  throw new TypeError('value is missing name');
}
```

**Exceptions:** `as const` is not an unsafe assertion. When local facts truly guarantee the type, `as` or `!` is allowed, with the reason in a one-line comment next to it.

**Tooling:** `@typescript-eslint/no-non-null-assertion`, `no-unnecessary-type-assertion`.

**Source:** Google TypeScript Style Guide, TypeScript Handbook

### Type predicates must actually check [Recommended]

`x is T` and `asserts x is T` are the author's promise to the compiler, and the compiler does not verify the function body. Predicates must cover all the fields of `T` that are used. For complex structures, use a validation library instead.

**Source:** TypeScript Handbook

### Use type annotations on object literals, not `as` [Recommended]

`const foo: Foo = {...}` reports extra and missing fields, and `{...} as Foo` does not. Use `satisfies` when you want the check and also want to keep the literal type.

**Tooling:** `@typescript-eslint/consistent-type-assertions`.

**Source:** Google TypeScript Style Guide

### Use discriminated unions for mutually exclusive states [Recommended]

Make illegal combinations unrepresentable. Do not assemble state from multiple optional fields.

**Example:**

```ts
type LoadState =
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: User };
```

**Source:** TypeScript Handbook

### Handle every union branch [Recommended]

Add a `never` fallback to `switch`. When a member is added, the compiler points out every place that misses it.

**Example:**

```ts
function describeState(state: LoadState): string {
  switch (state.status) {
    case 'loading':
      return 'Loading';
    case 'error':
      return state.error.message;
    case 'success':
      return state.data.name;
    default: {
      const unhandled: never = state;
      throw new Error(`Unhandled state ${JSON.stringify(unhandled)}`);
    }
  }
}
```

**Tooling:** `@typescript-eslint/switch-exhaustiveness-check`.

**Source:** TypeScript Handbook, typescript-eslint Docs

### Use the simplest type construct [Recommended]

Use `interface` for object shapes, and `type` for unions, tuples, and utility types. Write return types on exported functions and leave local variables to inference. Derive fields with `Pick`, `Omit`, and `Partial`, not by copying by hand. When interface extension can express it, do not write custom mapped and conditional types: they are hard to read and rename tools cannot track them.

**Tooling:** `@typescript-eslint/consistent-type-definitions`.

**Source:** Google TypeScript Style Guide

## Nullish values

### Use one nullish convention per project [Recommended]

Internal code uses `undefined` or an omitted property to mean "none", consistent with `Map.get`, optional parameters, and optional chaining. Use `null` only where an external protocol explicitly requires it, such as JSON fields and database columns, and convert at the boundary.

**Why:** When both are mixed, every caller has to check both. `JSON.stringify` drops properties whose value is `undefined` and keeps `null`, so the two mean different things in transit.

### Keep `null` and `undefined` out of type aliases [Recommended]

Write `Order | undefined` at the point of use, not `type MaybeOrder = Order | undefined`. Handle nullish values close by, without passing them through many layers.

**Source:** Google TypeScript Style Guide

### Distinguish optional properties from `undefined` values [Recommended]

Under `exactOptionalPropertyTypes`, `name?: string` does not accept an explicit `undefined`. When you really need to pass `undefined`, write `name: string | undefined`, which means "must be provided, can be empty".

**Source:** TSConfig Reference

### Do not rely on truthiness for nullish checks [Required]

When `0`, `''`, or `false` are valid values, `if (count)` treats `0` as missing and `if (name)` treats an empty string as missing. Compare explicitly, or use `?.` and `??`. To check for both `null` and `undefined` you may write `value == null`, the only exception for `==`.

**Tooling:** `@typescript-eslint/strict-boolean-expressions`, `prefer-nullish-coalescing`, ESLint `eqeqeq` with `{ null: 'ignore' }`.

**Source:** TypeScript Handbook, Google TypeScript Style Guide

## Generics

### Relate at least two places with a type parameter [Recommended]

Type parameters exist to relate types. When one appears only once, use a concrete type or `unknown`. A generic that appears only in the return value is a hidden type assertion.

**Example:**

Avoid:

```ts
function parseJson<T>(text: string): T {
  return JSON.parse(text);
}
```

Prefer:

```ts
function parseJson<T>(text: string, schema: z.ZodType<T>): T {
  return schema.parse(JSON.parse(text));
}
```

**Tooling:** `@typescript-eslint/no-unnecessary-type-parameters`.

**Source:** typescript-eslint Docs, Google TypeScript Style Guide

### Express required capabilities with constraints [Recommended]

Put the members the function body uses in `extends`, and constrain key-object relationships with `keyof`. Do not write an unconstrained `T` and then assert inside the function body.

**Example:**

```ts
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}
```

**Source:** TypeScript Handbook

## Enums

### Prefer literal unions or `as const` objects [Recommended]

They have no extra runtime semantics. `enum` cannot be handled by Node.js type stripping and also errors under `erasableSyntaxOnly`. Do not use `const enum`: it conflicts with single-file transpilation and is invisible to JavaScript consumers.

**Example:**

```ts
const OrderStatus = { Pending: 'pending', Paid: 'paid' } as const;
type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
```

**Exceptions:** When a project already uses `enum` throughout, keep it and give members literal values.

**Tooling:** `@typescript-eslint/prefer-literal-enum-member`.

**Source:** TypeScript Handbook, TSConfig Reference, Google TypeScript Style Guide

## Data classes

### Use interfaces, not classes, for plain data [Recommended]

The type system is structural: a plain object with the right fields can be assigned to a class type, so a class gives no extra guarantee. Write a class only when it has behavior and invariants. Use `readonly` for fields that do not change, and `readonly T[]` for parameters.

**Source:** Google TypeScript Style Guide

### Use branded types for same-shaped identifiers with different meanings [Recommended]

With structural types, `userId` and `orderId` are both `string` and can be passed in place of each other by mistake. When mixing them would cause defects, use `string & { readonly __brand: 'UserId' }`, and create values only in validation functions.

## Type boundary validation

### Validate external data at the boundary [Required]

Types do not exist at runtime. Receive request bodies, API responses, stored content, and URL parameters as `unknown`, validate them with Zod, Valibot, or similar before they enter the business layer, and do not recheck in inner layers. Derive types from the schema to keep a single definition. Parse `process.env` into a config object once at startup, and exit immediately on failure.

**Example:**

```ts
const orderSchema = z.object({ id: z.string(), totalCents: z.number().int() });
type Order = z.infer<typeof orderSchema>;

const order: Order = orderSchema.parse(await response.json());
```

**Tooling:** `no-unsafe-assignment` reports the `any` returned by `response.json()`.

**Source:** MDN
