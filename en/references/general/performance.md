# Performance methodology

This document covers general methods. For specific techniques, see the frontend and backend performance files. For general problems, research mature libraries first. See "Prefer mature libraries for general problems" in [dependencies.md](dependencies.md).

### Measure first [Required]

Do not optimize without data. Find the real bottleneck first. Avoiding known anti-patterns such as N+1 queries and duplicate requests does not need measurement first.

**Source:** Team

### Measure before and after the same way [Required]

Use the same environment, data volume, and steps before and after the change. Otherwise the results are not comparable.

**What to record:**

| Item | Example |
|---|---|
| Scenario | Opening a large file |
| Environment | Machine, version, network |
| Method | Tools and steps |
| Runs | Number of repetitions and how they are aggregated |
| Results | Numbers before and after optimization |

**Source:** Team

### Look at the distribution [Recommended]

Look at P95 and P99 as well as the average. The average hides a small number of very slow requests.

### Make optimization worthwhile [Recommended]

The complexity an optimization adds must be proportionate to the actual gain. When the gain is unclear, keep the simple implementation.

### Prefer structural changes [Recommended]

Reducing unnecessary work, such as duplicate requests and redundant computation, is usually more effective than fine-tuning code.

### Report actual numbers [Recommended]

When reporting, state the before and after numbers and the measurement method. Do not just say "it is faster".
