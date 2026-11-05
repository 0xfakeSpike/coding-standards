# Dependencies and supply chain

## Adding dependencies

### First decide whether you can go without it [Recommended]

Every new dependency expands the attack surface and the maintenance cost.

**Source:** OpenSSF Guide to Evaluating Open Source Software

### Prefer mature libraries for general problems [Recommended]

Research mature libraries first. Do not reimplement what is already solved. Weigh this rule together with the previous one: do not add a dependency for what a few lines of code can solve, and do not build your own solution to a complex general problem.

**Source:** Team

### Confirm the package source [Required]

Confirm it is the official package. Guard against lookalike packages with similar names.

**Source:** OpenSSF Guide to Evaluating Open Source Software

### Evaluate before adding [Recommended]

| Check | Requirement |
|---|---|
| Activity | Review commits and releases from the last 12 months |
| License | Open source dependencies use an OSI-approved license compatible with the project. For commercial dependencies, confirm the license terms |
| Vulnerabilities | Review known vulnerabilities and how fast they get fixed |
| Size | Assess size and transitive dependencies |

**Source:** OpenSSF Guide to Evaluating Open Source Software

## Versions

### Declare dependencies explicitly [Required]

List every dependency in the manifest. Do not rely on packages that happen to exist in the environment.

**Source:** Twelve-Factor App

### Make builds reproducible [Required]

Commit the language's lockfile, or pin versions in an equivalent way.

**Source:** Google SRE: Release Engineering

### Do not pin old versions to avoid code changes [Required]

When a tool upgrade leaves the code incompatible, change the code to fit. Do not avoid it by pinning an old version.

**Exceptions:** You may pin temporarily when an upstream defect blocks the upgrade. Record the reason and the condition for removing the pin.

**Source:** Team

### Upgrade regularly [Recommended]

Use Dependabot or Renovate. Upgrade in small steps so changes do not pile up into major version jumps.

## Vulnerabilities and provenance

### Run vulnerability scans in CI [Recommended]

**Applies to:** production service

For high-severity vulnerabilities, upgrade promptly or record the disposition.

## Cleanup

### Delete unused dependencies [Required]

**Source:** Team
