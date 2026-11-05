# TypeScript / JavaScript idioms

## Idioms

### No pass-through functions or pointless wrappers [Required]

Delete functions, classes, and modules that only forward a call unchanged. Callers use the original API.

**Example:**

Avoid:

```ts
function getUser(id: string): Promise<User> {
  return userRepository.findById(id);
}
```

Prefer:

```ts
const user = await userRepository.findById(id);
```

**Source:** Team

### Default to `const`, never `var` [Recommended]

Use `let` only when reassignment is needed. Declare one variable per statement.

**Tooling:** ESLint `prefer-const`, `no-var`.

**Source:** Google TypeScript Style Guide

### Use `===` and `!==` [Recommended]

The implicit conversion of `==` is hard to predict. For the exception, see the nullish check rule in [types.md](types.md).

**Tooling:** ESLint `eqeqeq`.

**Source:** Google TypeScript Style Guide

### Use `??` for defaults, not `||` [Required]

`||` replaces valid values such as `0`, `''`, and `false`. Write `options.pageSize ?? 20`.

**Tooling:** `@typescript-eslint/prefer-nullish-coalescing`.

### Use function declarations for named functions [Recommended]

Write top-level named functions with `function` and callbacks as arrow functions. Use `const fn: Handler = () => {}` only when a type annotation is needed. Do not rebind `this`. Use arrow functions or explicit parameters.

**Source:** Google TypeScript Style Guide

### Use object parameters when the call site is unclear [Recommended]

`createUser(data, true)` does not show what `true` means. Write `createUser(data, { sendWelcomeMail: true })`, or split it into two functions.

### Do not mutate parameters [Recommended]

Produce new values with spread, `map`, `filter`, and `toSorted`. Spread is a shallow copy. For a deep copy use `structuredClone`, which cannot copy functions.

**Source:** MDN

## Collections

### Use `Map` for dynamic keys and `Set` for sets [Recommended]

Use `Map` when keys come from runtime data. Plain objects expose properties on the prototype chain, and their keys can only be strings or Symbols. Use `Record` when the set of keys is statically known.

**Source:** Google TypeScript Style Guide, MDN

## Strings

### Build strings with template literals [Recommended]

Do not concatenate multiple pieces with `+`. Put only `string` and `number` in templates. Format objects first.

**Tooling:** `@typescript-eslint/restrict-template-expressions`, `restrict-plus-operands`.

**Source:** typescript-eslint Docs

### `length` is not the character count [Required]

`length` is the number of UTF-16 code units. When truncating or counting user text, process by grapheme with `Intl.Segmenter`, or emoji and combining characters get cut. Use `Intl.Collator` for user-facing sorting. `<` compares by code unit only.

## Dates and times

### Store and transmit instants in UTC [Required]

Use ISO 8601 strings or timestamps in storage and APIs, and do time zone conversion in the presentation layer. `Date` is a UTC timestamp internally, but methods such as `getHours` interpret it in the process's time zone.

**Source:** MDN

### Know the `Date` parsing rules [Required]

`new Date('2026-01-01')` parses as UTC, and `new Date('2026-01-01T00:00')` parses as local time. Months start at 0. Setters mutate the original object. When parsing external formats, use validated ISO 8601 strings and do not rely on non-standard formats.

**Source:** MDN

### Prefer `Temporal` [Recommended]

When the runtime supports it, use `Temporal` for time zones, calendars, and date arithmetic. Otherwise use a mature date library. Do not hand-write time zone and month arithmetic. A type existing does not mean the runtime has it.

**Source:** MDN, TypeScript Release Notes

## Numbers and money

### No decimal arithmetic on money with `number` [Required]

`number` is an IEEE 754 double-precision float: `0.1 + 0.2 !== 0.3`. Use integers in the smallest unit or a decimal library, and put the unit in the name, such as `totalCents`.

**Source:** MDN

### Do not use `number` for big integers [Required]

Precision is lost above `Number.MAX_SAFE_INTEGER` (2^53 - 1). Transmit 64-bit database IDs as strings and compute with `bigint`. `JSON.stringify` throws on `bigint`, and `JSON.parse` loses precision on large numbers.

**Source:** MDN

### Check the result of string-to-number conversion [Required]

Use `Number(text)` and check `Number.isFinite`. `parseInt('12px')` returns 12, so use `parseInt` only when you need a non-decimal radix, and state the radix. An empty string converts to 0 with `Number`, so check for it separately.

**Source:** Google TypeScript Style Guide, MDN
