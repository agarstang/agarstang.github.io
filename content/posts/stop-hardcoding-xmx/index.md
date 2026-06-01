---
title: "Stop Hardcoding -Xmx: Moving to Container-Aware JVM Flags"
date: 2025-09-09
summary: "If you are still using -Xms and -Xmx inside your Docker containers, you could be wasting cloud resources or risking OOMKilled errors when resizing deployments."
tags: ["java", "kubernetes", "docker", "devops"]
---

If you are still using `-Xms` and `-Xmx` inside your Docker containers/Kubernetes (K8s) Pods, you could be wasting cloud resources or risking accidentally causing `OOMKilled` errors when resizing deployments.

Historically, the JVM was "container-blind," seeing the host's total RAM instead of the container's limits. While we used to fix this by setting the above flags, modern Java (10+) provides a stable, percentage-based approach.

## The Problem with Fixed Values

Hardcoding `-Xmx2g` creates a rigid configuration. If an Ops engineer changes the K8s memory limit to 4GB, your Java app stays throttled at 2GB. Even worse, if they lower the limit to 2GB, your app may try to claim it all, leading to a container crash.

Historically different conventions have been used for passing flags to the JVM and it can be confusing or easy to miss these when making changes.

## RAM Percentage Flags

Since Java 10+ the JVM has been container (cgroup) aware allowing us to set percentage based flags. These flags allow the JVM to calculate its heap size dynamically based on the limits of your container.

- **`-XX:InitialRAMPercentage=n`**: Replaces `-Xms`. Sets the starting heap size.

- **`-XX:MaxRAMPercentage=n`**: Replaces `-Xmx`. Sets the ceiling.

- **`-XX:MinRAMPercentage=n`**: ⚠️ This is rarely used. It only applies if your total container RAM is extremely small (under ~250MB).

| Scenario | Recommended Flags | Logic |
|----------|-------------------|-------|
| Standard Microservice | `-XX:MaxRAMPercentage=80.0` | Leaves 20% for Metaspace, Stack, and OS. |
| High-Performance | `-XX:InitialRAMPercentage=75.0` `-XX:MaxRAMPercentage=75.0` | Prevents "Stop-the-World" pauses during heap resizing. |
| Sidecars/Tiny App | `-XX:MaxRAMPercentage=50.0` | Small containers need more relative non-heap space. |

## Conclusion

By using percentages, the configuration becomes "Set and Forget." Your app will automatically scale its memory usage if/when your infrastructure limits change. Ops engineers can change a container size without having to debug what convention was used for passing flags in the entrypoint.
