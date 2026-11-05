# Code design

## Architecture and soundness

### Architecture determines simplicity [Required]

Whether code is simple depends first on whether the architecture is sound, not on cleanup afterwards. When designing, first consider whether concepts and responsibilities are clear.

**Source:** Team

### Working does not mean sound [Required]

If the system works but its concepts are awkward, change it anyway.

**Signals:**

- It takes extra explanation to say why the design is this way.
- The same concept means different things in different modules.
- New requirements always have to work around the existing structure.

**Source:** Team

### Choose the best location [Required]

Assess where the change belongs most. Do not put it somewhere just because "it can be done here right now".

**Key questions:**

1. Which module's responsibility is this?
2. If it goes here, which module ends up knowing something it should not know?
3. Where does similar logic live now?

**Source:** Team

## Abstraction

### Do not add unnecessary wrappers [Required]

Do not keep functions that only forward a call or structures that only add one layer of wrapping.

**Why:** Each wrapper makes readers jump one more time and adds no information.

**Criteria:** A wrapper should hide complexity, unify a rule, or express a concept. If it does none of these, delete it.

**Source:** Team

### Prefer composition over inheritance [Recommended]

Inheritance makes subclasses depend on the parent's implementation details. When you need to reuse behavior, prefer composition.

### Do not abstract early [Recommended]

Abstract when a real second or third consumer appears.

**Source:** Fowler YAGNI

### Do not use patterns for their own sake [Recommended]

Use design patterns to solve problems that have already appeared, not to prove a design is "proper". The indirection a pattern adds must bring real benefit.

## Dependencies

### Keep dependency direction one-way [Recommended]

Allow no circular dependencies between modules. High-level business code depends on an abstraction only when an implementation really needs to be replaceable. Do not define interfaces in advance for this.

### Pass dependencies in from outside [Recommended]

Pass in dependencies that need to be replaced or tested from outside. Do not create them directly in business logic.

## Data and side effects

### Keep side effects at the boundary [Recommended]

Write core logic as pure functions where possible and put I/O in the outer layer, so the logic is easy to test.

### Prefer immutable data [Recommended]

Do not modify shared data by default. When you must modify it, keep the scope small and explicit.

## Design principles quick reference

| Principle | Meaning | Common violation signal |
|---|---|---|
| Single responsibility | A module has only one reason to change | A file handles everything |
| Open/closed | Add behavior through extension | Every new type requires changing conditionals in several places |
| Liskov substitution | Subtypes can replace their parent type | A subclass throws "not supported" |
| Interface segregation | Consumers depend only on the interfaces they need | Many empty methods are implemented |
| Dependency inversion | Business logic does not depend on concrete implementation details | The business layer operates drivers directly |

These principles are tools for judgment, not goals. When you add a layer to satisfy a principle, go back to "Do not add unnecessary wrappers" and reassess.

## Common anti-patterns

| Anti-pattern | Symptom | Action |
|---|---|---|
| God object | One module carries many responsibilities | Split by responsibility |
| Anemic shell | A type only forwards to another type | Delete this layer |
| Shotgun surgery | One requirement changes many files | Consolidate duplicated logic |
| Feature envy | A function reads heavily from another module's data | Move the logic to where the data lives |
| Boolean parameter branching | One function does two things based on a flag | Split it into two functions |
