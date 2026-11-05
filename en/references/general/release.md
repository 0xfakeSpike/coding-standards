# Release and rollback

## Release

**Applies to:** production service (all rules in this section)

What you release must be reproducible from its build identifier. See "Make builds reproducible" in [dependencies.md](dependencies.md) and "Make builds traceable" in [ci-cd.md](ci-cd.md).

### Release in stages by risk [Recommended]

Release high-risk changes to a small scope first, and expand after confirming metrics are normal.

| Method | When to use |
|---|---|
| Gradual rollout | Increase traffic step by step and watch the impact |
| Blue-green | Fast cutover and switching back are needed |
| Rolling | Routine updates to stateless services |

**Source:** Google SRE: Release Engineering

### Write down the steps before releasing [Recommended]

| Item | Content |
|---|---|
| Change scope | What this release contains |
| Release order | The order of database, configuration, and services |
| Metrics to watch | Metrics that show whether things are normal |
| Rollback method | Trigger conditions and concrete steps |

### Land fixes on trunk first [Recommended]

When a release branch needs a fix, fix trunk first, then cherry-pick to the release branch, so the next version does not lose the fix.

**Source:** Google SRE: Release Engineering

## Feature flags

### Manage flags by type [Recommended]

| Type | Purpose | Lifetime |
|---|---|---|
| Release | Hide unfinished features | Short-term |
| Experiment | A/B experiments | For the length of the experiment |
| Ops | Runtime protection | Keep as needed |
| Permissioning | Open to specific users | Can be long-term |

**Source:** Fowler: Feature Toggles

### Give flags an owner and a cleanup plan [Required]

Release and Experiment flags need an owner and a removal date. Once the switchover is complete, delete the flag and the old path together. Ops and Permissioning flags need an owner and a periodic review date.

**Source:** Team, Fowler: Feature Toggles

### Centralize decision logic [Recommended]

Keep flag checks in one place. Do not scatter `if` through the code.

**Source:** Fowler: Feature Toggles

## Rollback

**Applies to:** production service (all rules in this section)

### Prepare rollback in advance [Required]

Before shipping, confirm the rollback steps work.

### Contain the damage first during incidents [Required]

Restore service first, then analyze the root cause. After containment, a root fix is still required.
