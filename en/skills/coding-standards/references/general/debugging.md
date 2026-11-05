# Debugging

For regression tests during a fix, see "Write a regression test before fixing a defect" in [testing-strategy.md](testing-strategy.md).

### Trace symptoms back to structure [Required]

When a strange problem appears, first check whether the division of responsibilities across packages, files, and functions is sound.

**Check order:**

1. Who is responsible for this?
2. Is the same thing implemented in several places?
3. Does the data have several sources?
4. Are there too many call sites?
5. Do modules have hidden ordering dependencies?

**Why:** Recurring strange problems are often symptoms of structural problems.

**Source:** Team

### Reproduce the problem [Recommended]

After checking the structure, find a reliable way to reproduce it. When you cannot reproduce it, add information that can locate the problem. Do not make changes on a guess.

### Find the root cause before fixing [Required]

You have found the root cause only when you can explain "why it happened". A change that only makes the symptom go away is not a fix.

**Source:** Team

### Change one variable at a time [Recommended]

When you change several things at once, you cannot tell which one had the effect.

### Narrow the scope with evidence [Recommended]

Use logs, breakpoints, commit bisection, or a minimal reproduction to determine the scope of the problem. Record each item as you rule it out.

### Clean up after debugging [Required]

Delete temporary logs, debug switches, and experimental changes.

### Check for similar problems [Recommended]

After the fix, check whether the same problem exists elsewhere. If it exists in several places, prefer consolidating the structure over patching each place.
