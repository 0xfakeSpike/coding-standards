# Configuration

This document sets rules for configuration sources, validation, dynamic adjustment, feature flags, and the secrets boundary. For configuration injection at the container level, see [Containers](../../general/containers.md).

## Configuration layers

### Separate configuration from code [Required]

Every value that varies by deployment environment is configuration, such as database addresses, external service addresses, and credentials. Provide these values through environment variables or a configuration center. Do not write them in code.

Internal wiring that does not vary by deployment, such as route tables and dependency assembly, is not configuration and stays in code.

**Why:** With configuration in code, changing environments means changing code, and credentials end up in version control.

**Example:** The test: if the codebase were made public right now, no credentials would leak.

**Source:** Twelve-Factor App

### Merge configuration sources by priority [Recommended]

| Priority | Source | Content |
|---|---|---|
| High | Runtime dynamic configuration | Values that need adjusting without a restart |
| Medium | Environment variables or deployment platform injection | Differences per deployment |
| Low | Defaults in code | Safe defaults that are reasonable for all environments |

Use one priority order across the project and record it in the project docs. Do not scatter configuration sources through business code. Read configuration in one place, and have business code receive an already parsed configuration object.

### Do not group configuration by environment name [Recommended]

Do not write branches like `if env == "production"`, and do not maintain sets of configuration named after environments. Set each configuration item independently.

**Why:** As environments multiply, the number of combinations gets out of control, and behavior that exists only in one environment appears easily.

**Example:**

Avoid:

```json
{
  "production": { "timeoutMs": 2000, "retry": true },
  "staging": { "timeoutMs": 5000, "retry": false }
}
```

Prefer:

```text
PAYMENT_TIMEOUT_MS=2000
PAYMENT_RETRY_ENABLED=true
```

**Source:** Twelve-Factor App

### Commit a configuration sample [Recommended]

Commit `.env.example` with only variable names and example values, no real values. Update it when adding configuration items, and state their meaning and units.

### Put units in configuration names [Recommended]

For durations and sizes, state the unit in the name, such as `REQUEST_TIMEOUT_MS` and `MAX_UPLOAD_BYTES`.

## Validation

### Validate at startup and fail immediately when values are missing [Required]

At startup, validate the presence, type, and value range of all configuration. On validation failure, exit and state which item is wrong. Error messages do not output the configuration value itself.

**Why:** Problems left until runtime often surface only when a rarely used path runs, and by then traffic is already coming in.

**Example:**

Avoid:

```text
Service started successfully, error only on the first payment: payment url is empty
```

Prefer:

```text
Startup failed: PAYMENT_TIMEOUT_MS must be a positive integer, current value is invalid
```

### Do not give required configuration a default [Required]

When required items such as database addresses and credentials have no reasonable default, do not write a placeholder default. With a default pointing to a local address, a missing setting becomes a wrong connection instead of a startup failure.

### Make defaults safe [Required]

For configuration items with defaults, choose the more conservative side, such as debug mode off by default and validation on by default.

## Dynamic configuration

### Make configuration dynamic only when truly needed [Recommended]

If configuration can ship with a deployment, do not make it dynamic. What needs dynamic adjustment is usually values that must change quickly during incidents, such as rate limit thresholds and ops flags.

**Why:** Static configuration is reviewed and rolled back with the code. Dynamic configuration adds a distribution system and another kind of production state.

**Source:** Fowler: Feature Toggles, Azure: Throttling

### Validate dynamic configuration before updating [Required]

Validate new values before they take effect. On validation failure, keep the current value and raise an alert. Do not let the process run with invalid configuration.

### Keep running when the configuration center is unavailable [Required]

**Applies to:** production service

After the service starts, an unavailable configuration store does not affect operation. Keep using the last valid configuration in memory. Cache configuration reads, and do not read remotely on every request.

**Source:** Azure: External Configuration Store

### Keep a record of dynamic configuration changes [Recommended]

**Applies to:** production service

Record who changed what to what value and when, and set access permissions. Configuration changes can cause incidents just like releases, so troubleshooting must be able to match them up in time.

**Source:** Azure: External Configuration Store

### Log the effective configuration at startup [Recommended]

At startup, output a summary of non-sensitive configuration to make it easy to confirm what is actually running in production. For sensitive items, output only whether they are set.

## Feature flags

### Delete old-to-new switchover flags once the switch is complete [Required]

Feature flags introduced to switch from old to new are temporary. When the switch is complete, delete the flag and the old path together. For flag types, owners, and removal dates, see [Release and rollback](../../general/release.md).

**Why:** Leftover flags leave people unsure which path is running, and branch combinations keep adding to test costs.

**Source:** Team

### Treat flags and configuration separately [Recommended]

| Type | Lifecycle | Stored in |
|---|---|---|
| Release flag | Short-term, delete after use | Static configuration preferred |
| Ops flag | Kept long-term as needed | Dynamic configuration |
| Ordinary configuration | Long-term | Configuration sources |

Flags are inventory with a carrying cost. The fewer the better.

**Source:** Fowler: Feature Toggles

### Make flag state queryable [Recommended]

Provide a way to query current flag state, such as an admin endpoint or startup logs. During troubleshooting, you must be able to tell which path an instance is taking.

**Source:** Fowler: Feature Toggles

## Secrets boundary

When printing configuration and validation errors, mask sensitive items. See [Logging](../observability/logging.md).

Keep secrets out of code, images, and ordinary configuration. See [Cryptography and secrets](../security/crypto-and-secrets.md).

### Prefer runtime retrieval or file mounts [Recommended]

Prefer a secrets management service or mounted files when available. Environment variables are visible to processes in the same container and can end up in logs and dumps. Use them only when no other way is workable.

**Source:** OWASP Secrets Management Cheat Sheet
