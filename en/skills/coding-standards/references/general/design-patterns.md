# Design patterns

This file covers when to use a design pattern, which one to pick, and how light a form to write it in. For where a change goes and whether to add a wrapper, see [code-design.md](code-design.md). For module boundaries, see [responsibilities.md](responsibilities.md).

Code that reads as convoluted usually does not suffer from sloppy details. It is missing the right structure: a lookup written as a long chain of conditions, a state machine scattered across several booleans, a pipeline written as nested blocks. Comments and smaller functions do not fix this. The structure has to change. The reverse is also true: forcing a pattern where there is no problem makes code just as convoluted.

## Principles

### When code is convoluted, look for the missing structure first [Required]

When writing or reviewing code, if a piece of code makes you jump back and forth or needs comments to explain it, first use the signal table below to find the structure it is missing, then decide how to change it. Do not ease it by adding comments, adding conditions, or mechanically splitting functions.

**Signals:**

- The same set of branches on a type or state appears in several places.
- Adding a type or a step means changing conditions in several places.
- Understanding a function means keeping a combination of several flags in your head, or working out from a chain of conditions which step it has reached.
- The main flow is buried under precondition checks, logging, retries and other code unrelated to the business logic.

**Source:** Team

### A pattern must bring real benefit [Recommended]

Use patterns to solve problems that have already appeared, not to prove a design is "proper". When you introduce one, you should be able to say what it reduces: fewer duplicated branches, rules kept in one place, or one place to change when a new type is added. If you cannot say, do not use it.

**Criteria:** Would the code be harder to read or change without this pattern? If not, delete it.

**Common counterexamples:**

- A strategy interface with one implementation and no need to replace it.
- A factory that only creates one kind of object.
- A decorator or proxy that adds nothing.
- An event with a single receiver that the sender already knows.

### Choose the lightest form [Recommended]

A pattern can be written in lighter or heavier forms. Choose in this order: data (a lookup table), functions, a type with methods, a class hierarchy. Do not use a later form when an earlier one is clear enough.

**Example:** To compute fees by payment channel, a "channel → fee function" table is enough. You do not need a `FeeStrategy` interface, one class per channel and a factory. Turn a channel into a type only when it needs its own state and several methods.

### Do not copy Java-style pattern structures [Required]

When a pattern becomes code, use the idiomatic form of the language. TypeScript, Python and Go all have first-class functions, modules and closures, so the class structures that textbooks designed for Java are mostly extra layers in these languages. See each language's `patterns.md` in the language layer for the matching forms.

**Signals:**

- An interface with a single implementation, or paired names like `IUserService` and `UserServiceImpl`.
- One strategy with an interface, an abstract base class, one class per case, and a factory or registry on top.
- A singleton built with `getInstance()`.
- `Utils`, `Helper` or `Manager` classes that hold only static methods.
- Getters and setters that only pass a field through.
- An abstract base class and inheritance hierarchy built to reuse a few lines.
- A builder for an object with only a few fields.

**Why:** Each extra layer of class structure makes readers jump one more time, and in these languages those layers add no guarantee.

**Source:** Team

### Migrate everything when you introduce a pattern [Recommended]

When a new structure replaces an old approach, move all the old branches over and delete the old approach. Do not let the two coexist.

**Why:** With two approaches, readers first have to work out which one to read, and new requirements have no clear home. See "Keep one copy of each thing" in [responsibilities.md](responsibilities.md).

## Signal table

| Signal | Consider first | Lightest form |
|---|---|---|
| A chain of conditions only maps a key to a value or handler | Lookup table | A dictionary or map |
| Branches on the same type field appear in several places, and types keep being added | Organize behavior by type (strategy, polymorphism) | One function per type, kept in a map |
| The types are fixed, and the operations on them keep growing | One exhaustive branch per operation (visitor) | One function per operation, with a `switch` on the type inside |
| Several booleans or nullable fields add up to one state, or a chain of `if` statements works out which step has been reached | State machine | One state field, with a transition table or a `switch` on the state |
| A request passes through several steps, any of which may stop it | Pipeline, middleware (chain of responsibility) | A list of functions with the same signature |
| Two flows differ in only one or two steps | Pass in the steps that differ (template method) | A flow function that takes function parameters |
| Logging, timing, retries or permission checks are copied into many functions | Decorator, middleware | A wrapper function |
| Business code handles third-party fields and formats everywhere | Adapter | One conversion function at the boundary |
| Code that creates different implementations by condition is scattered around | Factory | One creation function at the entry point |
| After an action, the sender calls unrelated modules one by one | Observer, events | A callback list or domain events |
| Operations need queuing, undo, replay or auditing | Command | Represent the operation as data |
| A tree is handled differently for single nodes and groups | Composite | Nodes and groups implement the same operations |
| Call sites pass a string of optional parameters with unclear meaning | Options object | Named parameters or a config structure |
| The same value is checked for null in many places | Null object, default value | A default or no-op implementation at the boundary |

The table says what to consider first, not what you must use. Check each case against "A pattern must bring real benefit".

## Branches

### Map values with a table, not a chain of conditions [Recommended]

When a chain of conditions only maps a key to a value or a handler, write it as a table. Adding an entry changes one line, and tests can iterate over the whole table.

Do not force it into a table when each branch holds its own complex logic.

**Example:** To return a message for an error code, write an "error code → message" map with a default for missing codes, not twenty `if` statements.

### When branches on a type repeat, check whether types or operations are growing [Recommended]

When the same type field is checked in two or more places, first look at which side will grow:

- **Types grow, operations stay mostly fixed:** every new type means changing these branches. Keep each type's behavior together, with one set of functions per type or one interface with an implementation per type. Check the type in only one place, where the value is parsed or created.
- **Types stay fixed, operations grow:** write one function per operation, with an exhaustive `switch` on the type inside. A new operation is one new function, and when a type is occasionally added, the compiler or linter points out every place that misses it. Do not turn this into polymorphism or visitor classes.

When there is only one branch point and each branch is short, keep the `switch` and make it exhaustive.

**Example:** Membership tiers change often, and pricing, display text and refund rules all branch on the tier, so a new tier means changing three places. Instead, give each tier one set of pricing, text and refund behavior, so a new tier adds one set.

**Example:** Messages come in only three types, text, image and file, which rarely change, but the validation, rendering, export and moderation done on them keep growing. Write one function per operation, with a `switch` on the message type inside.

### When null checks are everywhere, supply a default or no-op at the boundary [Recommended]

When the same value is checked for null in many places, handle it once where it enters business code: give optional configuration a default, and give an optional dependency an implementation that does nothing, such as a logger that writes nothing or a notifier that sends nothing. Code after that no longer checks for null.

Do this only when "missing" is itself a valid state. When the value should never be empty, find out why it is empty, as in "Do not patch" in [principles.md](principles.md), instead of hiding it with a default. A default must not loosen security limits, see "Make defaults safe" in [configuration.md](../backend/architecture/configuration.md).

## State

### Use a state machine when state is inferred from flags or conditions [Recommended]

Either of these means a state machine is missing:

- An object's lifecycle is pieced together from several booleans or nullable fields (`isPaid`, `isShipped`, `canceledAt`). Illegal combinations appear and the checks spread everywhere.
- Procedural code uses a chain of `if` statements to work out which step it has reached. Readers have to piece all the conditions together to see the flow, and every new step means changing the conditions before it.

Use one explicit state field instead, and choose the form by case:

- **Transition table:** when moving between states is only a mapping of "current state plus event gives the next state", list the allowed transitions in a table.
- **`switch` on the state:** when each state has its own handling logic, `switch` on the state. Each `case` handles only one state, does that step's work and returns the next state. Make the branches exhaustive, as shown in the language layer.

Either way, apply transitions in one place and reject illegal ones with an error.

**Example:** An order can be canceled only while awaiting payment. Write one entry in the transition table, "awaiting payment + cancel → canceled", instead of repeating `!isShipped && !isCanceled` in the cancel button, the API and the scheduled job.

**Example:** A file upload goes through four steps: get credentials, upload chunks, merge, verify. The procedural version decides which step to run with `if (!token)`, `if (token && !uploaded)` and `if (uploaded && !merged)` before each one. Instead, use the states "awaiting credentials, uploading, merging, verifying, done", and in a loop `switch` on the current state. Each `case` does only its own step, then returns the next state.

For each language, see its `patterns.md` in the language layer. For UI states, see [Handle all UI states](../frontend/components.md) in the frontend layer.

## Flow

### Use a pipeline or middleware for sequential steps [Recommended]

When a request passes through validation, authentication, rate limiting, conversion and similar steps in turn, and any step may stop it, write each step as a function with the same signature and run them from a list. The main flow is no longer nested, and adding or removing a step only changes the list.

When the framework already provides middleware or interceptors, use them.

### When flows differ in only a few steps, pass those steps in [Recommended]

When two flows differ in only one or two steps, write one flow function and pass the differing steps in as parameters. Do not copy the whole flow and edit each copy, and do not switch inside the flow with a boolean parameter.

**Example:** Exporting CSV and exporting Excel are both "query, filter, format, upload", and only formatting differs. Write one export flow that takes a format function, not two nearly identical export functions.

### Wrap cross-cutting logic instead of copying it into every function [Recommended]

When logging, timing, retries, caching, permission checks or other logic unrelated to the business repeats across functions, add it with a wrapper function, decorator or middleware, and keep only business logic in the business functions.

When it appears in only one or two places, write it inline. For the rules on retries themselves, see [resilience.md](../backend/reliability/resilience.md).

## Boundaries and creation

### Convert external formats once at the boundary [Recommended]

Convert data from third-party SDKs, external APIs and legacy systems into internal types in one adapter at the boundary. Business code works only with internal types and does not check external fields here and there. Changing a vendor only changes the adapter.

For the backend approach, see "Isolate the outside world with ports" in [layering.md](../backend/architecture/layering.md).

### Keep creation with selection logic in one place [Recommended]

When configuration or input decides which implementation to create, put that selection logic in a single creation function, usually at the program entry point or where dependencies are wired. Business code receives the created object and does not decide which one to create.

When there is only one implementation, create it directly. Do not add a factory.

## Notification

### Use events when the sender should not know the receivers [Recommended]

When an action must notify several modules whose responsibilities are unrelated to the sender's, the sender publishes an event and each receiver subscribes. Receivers decide for themselves when to act, consistent with "Reduce coupling" in [responsibilities.md](responsibilities.md). A sender that calls each receiver in turn depends on more and more modules.

When the sender needs the result, or the steps must happen in order, call directly or orchestrate explicitly. Do not use events. For domain events across modules, see [domain-modeling.md](../backend/architecture/domain-modeling.md).

## In the language layer

The matching forms for each language: [TypeScript](../languages/typescript/patterns.md), [Python](../languages/python/patterns.md), [Go](../languages/go/patterns.md).
