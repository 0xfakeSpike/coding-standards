# CI/CD

## Pipeline

### Keep software deployable at all times [Recommended]

Keep trunk releasable, and make releasing a low-risk routine operation. Continuous delivery does not require every commit to ship automatically.

**Source:** DORA: Continuous Delivery

### Separate build, release, and run [Recommended]

**Applies to:** production service

A build produces an immutable artifact. A release is an artifact plus configuration. Run only executes a released version.

**Source:** Twelve-Factor App

### Order the pipeline from fast to slow [Recommended]

| Stage | Content |
|---|---|
| Check | Format, lint, types |
| Test | Unit tests, integration tests |
| Build | Packaging, images, artifacts |
| Scan | Dependency and image vulnerabilities |
| Deploy | Promote through environments |

### Keep all configuration in version control [Recommended]

**Applies to:** production service

Put application configuration, system configuration, scripts, and database change scripts in version control. Do not commit secrets or local environment configuration. See [git-workflow.md](git-workflow.md).

**Source:** DORA: Continuous Delivery

## Gates

For how local verification and CI divide the work, see "Keep verification fast" in [change-workflow.md](../collaboration/change-workflow.md). For flaky checks, see "Quarantine and fix flaky tests" in [testing-strategy.md](testing-strategy.md).

### Pass CI before merging [Required]

**Applies to:** team project

Do not get around it by skipping tests or disabling rules. The exception is flaky tests quarantined under the [testing strategy](testing-strategy.md), which must be recorded in a ticket or project docs with a deadline.

### State what each gate is for [Required]

Each check must be able to state what problem it prevents. Delete unnecessary checks.

**Source:** Team

### Control existing issues with a baseline [Recommended]

Record the count of violations you cannot clear yet in a baseline. The count may only go down, never up.

## Artifacts and environments

### Build once, deploy to many environments [Recommended]

An artifact that passes testing moves straight to the next environment without a rebuild. Configuration supplies the differences between environments.

**Source:** Twelve-Factor App, Google SRE: Release Engineering

### Make builds traceable [Recommended]

**Applies to:** production service

Be able to trace a production version back to its commit, dependencies, and configuration.

**Source:** Google SRE: Release Engineering

### Let the platform manage pipeline secrets [Required]

Use the CI platform's secrets feature or short-lived credentials. Do not write them to configuration files or logs.

### Infrastructure as code [Recommended]

Manage environment configuration through code. Changes go through review.
