# Naming principles

This document covers only language-independent principles. For case and specific formats, see the language layer.

### Make meaning obvious at a glance [Required]

Readers can tell the purpose without looking at the implementation.

| Avoid | Prefer |
|---|---|
| `data` | `pendingOrders` |
| `flag` | `isExpired` |
| `process()` | `calculateInvoiceTotal()` |

**Source:** Team

### Name the key in map variables [Required]

The name must say what is looked up by what.

| Avoid | Prefer |
|---|---|
| `users` | `userById` |
| `cache` | `priceBySku` |

**Source:** Team

### Use one word for one concept [Recommended]

Do not mix several words for the same entity, and do not use one word for different concepts. When business terms are easy to confuse, maintain a glossary in project docs.

**Why:** Inconsistent words raise the cost of searching and understanding.

### Update names when responsibilities change [Required]

When a responsibility changes, change the name with it. Do not leave outdated names behind.

**Source:** Team

### Follow language conventions [Recommended]

Do not carry one language's naming style into another.

### Part-of-speech conventions [Recommended]

| Item | Convention |
|---|---|
| Function | Verb or verb-object phrase |
| Type | Noun |
| Boolean | An expression that answers "yes or no" |
| Collection | Plural, or a word with a collection meaning |

### Use antonyms in pairs [Recommended]

| Pairs |
|---|
| `start` / `stop` |
| `open` / `close` |
| `add` / `remove` |
| `create` / `delete` |
| `begin` / `end` |
| `enable` / `disable` |

### Make units explicit [Recommended]

Prefer expressing the unit in the type, such as a duration type. When the type cannot express it, put the unit in the name, such as `timeoutMs`.

### Match name length to scope [Recommended]

The larger the scope, the more complete the name. Local variables that span a few lines can be short.

### Use only common abbreviations [Recommended]

Use only industry-standard abbreviations, such as `id`, `url`, and `http`. Explain business abbreviations in project docs.

## Naming smells

| Smell | Problem |
|---|---|
| `data`, `info`, `temp` | Carries no information |
| `manager`, `helper`, or business logic in `utils` | Unclear responsibility |
| `newXxx` or `xxx2` to tell new and old implementations apart (constructors excepted) | The name does not say what differs |
| `handleData` | Both the action and the object are vague |
| Negated booleans, such as `isNotReady` | Double negatives when checked |
