# Responsibilities

### Keep boundaries clear [Required]

Every package, module, and file can state its responsibility in one sentence. Move out anything that does not belong in that sentence.

**Source:** Team

### Make names match what things do [Required]

File and function names must match what they actually do. When a name cannot say it clearly, the responsibility is usually the problem. Split what should be split, and merge what should be merged.

**Signals:**

- The name implies "and", or it can only be called `manager`.
- The file's contents have gone beyond its name.
- Two modules always change together and may need to be merged.

**Source:** Team

### Keep one copy of each thing [Required]

Keep only one copy of a business rule, constant, or configuration. When you find similar implementations, consolidate them into one, such as components, formatting logic, constants, and switching logic.

**Why:** Multiple implementations eventually behave differently, and changes easily miss one.

**Example:** When two places each have a PDF reader, they should share one set of components.

**Exceptions:** When code is only similar in shape, but its business meaning and reasons to change differ, you can wait for a third occurrence before abstracting.

**Source:** Team, Beck: Rules of Simple Design

### Reduce coupling [Required]

A module owns its own state and triggers. It does not rely on other modules to do this for it on the side.

**Example:** When module A needs to process incremental data, A checks for increments itself. Do not have module B trigger A as a side effect of writing.

**Source:** Team

### Consolidate call sites [Required]

Each capability has a single implementation entry point. Many call sites are not a problem for shared capabilities such as logging. When call sites multiply for key operations with side effects, analyze each one for whether it is justified.

**Why:** With a single entry point, behavior is easy to reason about and the scope of a change is easy to determine.

**Source:** Team

### Check ownership before changing [Recommended]

Before adding a feature, answer first: does this responsibility already have an owner? If it does, reuse or extend. If it does not, then decide where the new module goes.
