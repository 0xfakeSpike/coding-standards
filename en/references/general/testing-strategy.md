# Testing strategy

This document covers only language-independent strategy. For test frameworks and how to write tests, see the language layer.

## Layers

### Keep most tests at the low levels [Recommended]

Unit tests clearly outnumber integration tests, and end-to-end tests are the fewest. Avoid the "ice cream cone" shape, with heavy reliance on UI tests and very few unit tests.

| Level | Focus |
|---|---|
| Unit test | Pure logic, edge cases |
| Integration test | Serialization, storage, external boundaries, module collaboration |
| End-to-end test | Core user paths |

**Source:** Fowler: Test Pyramid, Practical Test Pyramid

### Add low-level tests when high-level tests fail [Recommended]

When a high-level test finds a problem and no low-level test failed, first reproduce it with a low-level test, then fix it.

**Source:** Fowler: Test Pyramid

## Writing tests

### Test behavior, not implementation details [Recommended]

Assert on inputs, outputs, and user-visible results. Otherwise refactoring causes many false failures.

### Tests must be able to fail when the code is wrong [Required]

A test with no effective assertions only adds maintenance cost.

**Source:** Google Code Review Guide

### Keep tests independent [Recommended]

Do not depend on execution order or share mutable state.

**Source:** Fowler: Eradicating Non-Determinism

### Cover error paths and edge cases [Recommended]

This includes empty values, boundary values, insufficient permissions, and dependency failures. Choose by risk.

### Write a regression test before fixing a defect [Recommended]

First write a test that reproduces the problem, watch it fail, then fix it. When it cannot reasonably be automated, such as pure style tweaks and intermittent races, state how to verify manually.

**Source:** Fowler: Test Pyramid

### One test case, one behavior [Recommended]

Use the "arrange-act-assert" structure. The test case name describes the scenario and the expected result.

## Test doubles

### Replace only system boundaries [Recommended]

With too many doubles, the test verifies only the doubles themselves.

| Type | Purpose |
|---|---|
| Stub | Returns preset results |
| Fake | A working simplified implementation |
| Mock | Verifies call behavior |
| Spy | Records calls |

### Do not use mocks in place of product implementations [Required]

Delete mock data and fake implementations from product code.

**Source:** Team

### Pair remote service doubles with contract tests [Recommended]

Otherwise the double may drift from the real service.

**Source:** Fowler: Eradicating Non-Determinism

## Non-determinism

### Quarantine and fix flaky tests [Recommended]

**Applies to:** team project

Move flaky tests out of the main flow and limit how long they stay quarantined. Do not rely on reruns to pass. Delete flaky tests that stay unfixed for a long time.

**Why:** Once people get used to ignoring failures, tests lose their value.

**Source:** Fowler: Eradicating Non-Determinism

### Do not use bare waits [Recommended]

For async waits, use callbacks or polling with a timeout. Do not write fixed-duration waits.

**Source:** Fowler: Eradicating Non-Determinism

### Control time and randomness [Recommended]

Tests can control the clock. See the language layer for how. Fix random seeds, except for fuzz tests and property tests. Use doubles for third-party services. For databases and message queues, follow the language layer.

**Source:** Fowler: Eradicating Non-Determinism

## Maintenance

For the scope of local verification, see "Keep verification fast" in [change-workflow.md](../collaboration/change-workflow.md).

### Delete outdated tests [Required]

Update tests when behavior changes on purpose. Do not keep tests with no protective value.

**Source:** Team

### Use coverage to find blind spots [Recommended]

Do not treat the coverage number as a goal. Chasing the number alone produces worthless tests.
