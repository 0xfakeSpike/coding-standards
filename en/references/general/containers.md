# Containers

## Dockerfile

### Keep only runtime files in the runtime image [Recommended]

Use multi-stage builds when there is a compile or packaging step. Leave compilers and build tools in the build stage.

**Source:** Docker Build Best Practices

### Choose small, trusted base images [Recommended]

**Applies to:** production service

Pin the base image version. When strict reproducibility is needed, pin it by digest and update it with an automated tool.

**Source:** Docker Build Best Practices

### Use `.dockerignore` to exclude irrelevant files [Recommended]

Keep local dependencies, build output, and secrets out of the build context.

**Source:** Docker Build Best Practices

### Order steps to use the cache [Recommended]

Put steps that change rarely first. For example, copy the dependency manifest and install dependencies, then copy the source code.

### Install packages in one layer [Recommended]

Put `apt-get update` and `apt-get install` in the same `RUN`. Otherwise a stale package index cache may be used.

**Source:** Docker Build Best Practices

### Run as a non-root user [Required]

**Applies to:** production service

This limits the impact if the container is compromised.

### Keep secrets out of images [Required]

Inject secrets at runtime. When a build needs them, use the build secrets mechanism. Do not write them into image layers.

### Rebuild images regularly [Recommended]

Refresh the base image when rebuilding to pick up security updates.

**Source:** Docker Build Best Practices

## Image tags

### Do not deploy with `latest` [Required]

**Applies to:** production service

Deploy with immutable tags, such as a commit SHA or version number, so you can tell exactly what is running in production.

## Processes

### Keep processes stateless, fast to start, and graceful to shut down [Required]

**Applies to:** production service

On a termination signal, stop accepting new requests, finish in-flight work, then exit. Stateful components such as databases do not need to be stateless.

**Source:** Twelve-Factor App

### Write logs to standard output [Recommended]

**Applies to:** production service

Treat logs as an event stream collected by the platform. Do not manage log files inside the container.

**Source:** Twelve-Factor App

## Kubernetes probes

**Applies to:** production service (all rules in this section)

### Choose probes by purpose [Recommended]

| Probe | On failure | Purpose |
|---|---|---|
| startup | Restarts the container after the threshold is exceeded. Other probes do not run until it succeeds | Protect slow-starting containers |
| liveness | Restarts the container after the threshold is exceeded | States it cannot recover from on its own, such as deadlock |
| readiness | Removed from service endpoints, not restarted | Temporarily unable to accept traffic |

**Source:** Kubernetes Probes

### Do not check external dependencies in liveness [Required]

Restarting the application during a brief database failure does not help, and may cause cascading restarts.

### Keep shared dependencies out of readiness [Recommended]

Include in readiness only dependencies that are exclusive to the instance and required. When a shared dependency fails briefly, all instances are pulled from traffic at once.

## Kubernetes resources

### Set requests and a memory limit [Recommended]

**Applies to:** production service

Requests determine scheduling. A limit is the runtime ceiling. A container whose memory exceeds the limit may be OOM-killed. Decide whether to set a CPU limit based on the workload. A container that reaches the limit is throttled.

**Source:** Kubernetes Resource Management

### Watch memory units [Required]

In Kubernetes, `M` and `m` mean different things. Do not write memory as `400m`.

**Source:** Kubernetes Resource Management

### Separate configuration from images [Recommended]

**Applies to:** production service

Put configuration in environment variables or a ConfigMap. Put sensitive configuration in a Secret or an external secrets service.

**Source:** Twelve-Factor App
