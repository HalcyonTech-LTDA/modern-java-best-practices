---
name: modern-java-concurrency
description: >-
  Use this skill when designing, implementing, or optimizing multi-threaded, concurrent, or asynchronous Java applications:
  Virtual Threads per-task model, task orchestration with CompletableFuture and Virtual Thread executors,
  workload separation (I/O vs CPU), and diagnosing/preventing carrier thread pinning.
---

# Stable Concurrency with Virtual Threads in Modern Java

This skill provides guidelines and patterns for writing high-throughput, maintainable, and production-stable concurrent Java code using standardized Project Loom virtual threads and `CompletableFuture`.

## Core Tenets

1. **Threads are Cheap**: Do not pool virtual threads. Spawn a new virtual thread for every concurrent I/O-bound task via `Executors.newVirtualThreadPerTaskExecutor()`.
2. **Stable Subtask Orchestration**: Coordinate parallel asynchronous subtasks using `CompletableFuture` combined with virtual thread per-task executors.
3. **Workload Separation**: Confine I/O-bound tasks to virtual threads; reserve platform threads (`ForkJoinPool.commonPool()`) for heavy CPU computation.
4. **Unpinned Execution**: Ensure blocking I/O does not occur inside `synchronized` monitors. Use `ReentrantLock` or non-blocking concurrent collections.

## Architectural Guides

- **Virtual Threads Execution Model**: [Virtual Threads Guide](./references/virtual_threads.md)
  - Thread-per-task model, executor configuration, Spring Boot integration, and CPU vs. I/O boundaries.
- **CompletableFuture Orchestration**: [CompletableFuture Orchestration](./references/completable_future_orchestration.md)
  - Production-stable subtask coordination, `allOf` aggregation, fail-fast mechanics, and timeout handling without preview APIs.
- **Pinning Prevention & Locks**: [Pinning and Locks](./references/pinning_and_locks.md)
  - Diagnosing carrier thread pinning via JVM flags and migrating blocking `synchronized` sections to `ReentrantLock`.

## Concurrency Checklist

- [ ] Are virtual threads used for I/O operations and HTTP/messaging endpoints?
- [ ] Are platform threads reserved strictly for compute-heavy workloads (e.g. cryptography, image processing)?
- [ ] Are thread pools avoided for virtual threads (`newVirtualThreadPerTaskExecutor` instead of fixed pools)?
- [ ] Are subtasks orchestrated via `CompletableFuture` powered by explicit virtual thread executors?
- [ ] Are explicit timeouts (`.orTimeout()`) attached to all asynchronous futures?
- [ ] Are `synchronized` blocks around network or database calls replaced with `ReentrantLock`?
