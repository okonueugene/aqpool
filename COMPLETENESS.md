# Project Aquifer — completeness checklist

This document records what the **Aquifer Pool** benchmark package contains, how the pieces fit together, and what was verified or intentionally left for you to finish.

## Layout

| Path | Role |
|------|------|
| `task.toml` | Task metadata, verifier/agent timeouts, environment sizing, explanations |
| `instruction.md` | Outcome-only brief for the agent (constraints, API, verification summary) |
| `environment/Dockerfile` | Java 21 + Maven image, Python for setup, dependency pre-resolve |
| `environment/setup_challenge.py` | Generates broken `AquiferPool.java`, `MockConnection.java`, and `pom.xml` under `/app` |
| `solution/solve.sh` | Oracle: patches `AquiferPool.java` with synchronized list access, permit release on create failure, fair semaphore |
| `tests/test.sh` | Copies stress test into `/app`, compiles with stderr kept, runs suite 30×, always writes `/logs/verifier/reward.txt` |
| `tests/AquiferPoolStressTest.java` | JUnit 5 stress tests (concurrency + permit recovery on same pool instance) |

## What was achieved

### Broken baseline (`setup_challenge.py`)

- **Non-synchronized `ArrayList`**: `getConnection()` checks `isEmpty()` and `remove()` without holding a lock, so two threads can observe a non-empty list and remove the same connection — a silent double-issue bug.
- **`Semaphore(maxSize)` non-fair**: allows starvation under extreme contention relative to the stated starvation requirement.
- **No permit release on `createNewConnection()` failure**: if `getConnection()` acquires the semaphore then creation throws, capacity can be lost until the process restarts.
- **`MockConnection.java` generated**: referenced by `createNewConnection()` and written alongside `AquiferPool.java` so the project compiles out of the box.

### Instructions (`instruction.md`)

- Requirements are **outcome-only** (safety, integrity, starvation, reliability) without prescribing `synchronized` vs locks.
- Explicit **edit surface**: only `AquiferPool.java`; tests and `MockConnection` are off limits.
- **Public API** is frozen.

### Stress tests (`AquiferPoolStressTest.java`)

- **`testConcurrentAccess`**: fixed thread pool, latch-released thundering herd, `ConcurrentHashMap.newKeySet()` detects the same `Connection` instance handed to two threads (`add` returns `false`), bounded wait for deadlocks, `remove` then `releaseConnection` ordering preserved.
- **`testPermitLeakRecovery`**: subclass flips an `AtomicBoolean` to simulate DB down then recovery; failures and recovery exercise the **same** `AquiferPool` instance; then acquires `MAX_POOL_SIZE` live connections and releases them in `finally`.

### Verifier (`tests/test.sh`)

- **`mvn clean compile test-compile`**: stdout suppressed to reduce noise; **stderr not redirected to `/dev/null`** so Maven or compiler errors remain visible on failure.
- **No `set -e` before the stress loop**: build failure writes `reward.txt` with `0` and exits cleanly; the 30-run loop does not skip reward emission because of an early `set -e`.
- **Copies** `/tests/AquiferPoolStressTest.java` into the Maven tree before running (expects harness to mount tests at `/tests`).

### Oracle (`solution/solve.sh`)

- Canary / benchmark banner comments and `set -euo pipefail`.
- **`synchronized (available)`** around borrow and return list operations.
- **`createNewConnection()`** outside the lock to avoid holding the monitor during work.
- **`try` / `catch`**: `semaphore.release()` on any exception after `acquire` and before a connection is successfully returned.
- **`new Semaphore(maxSize, true)`** for fair permit queuing.
- Closing `echo` confirms completion.

### Image (`environment/Dockerfile`)

- **`COPY setup_challenge.py` only** into `/app` — solution and tests are not baked into the image from the environment layer (aligns with “do not copy solution/tests into the challenge image” guidance; tests are supplied at verify time).
- **`mvn dependency:resolve`** (with `|| true`) warms the local repo for faster agent runs when network is available.

### Metadata (`task.toml`)

- Version `2.0`, canary GUID comment, tags, task subtypes, milestone count, difficulty/solution/verification explanations, junior/expert time estimates.
- **`[verifier]`**, **`[agent]`**, **`[environment]`** blocks with timeouts, CPU, memory, storage, GPU count, build timeout, `allow_internet`.

## Fixes applied during this completeness pass

1. **`tests/AquiferPoolStressTest.java`**: Removed invalid pseudo–text-block `""" ... """;` fragments (not valid Java expression statements). Replaced with ordinary `//` comments so the suite **compiles**.
2. **`task.toml`**: Added **`name`** and **`difficulty`** under `[metadata]` so the package identifies the task consistently in tooling.

## Gaps / follow-ups before shipping

| Item | Status |
|------|--------|
| `author_name`, `author_email`, `author_organization` in `task.toml` | Still **empty** — fill with real maintainer data before publication. |
| Harness contract | Confirm the runtime mounts **`tests/` at `/tests`** and runs **`test.sh`** with `/app` as the Maven project root after `setup_challenge.py` has run (Docker `RUN` or equivalent). |
| Stray directory | If present, a folder literally named `{environment,solution,tests}` is non-standard; remove or ignore when zipping. |
| End-to-end build | Run `docker build` + verifier locally once Java/Maven paths match production to confirm `dependency:resolve` and tests behave with your network policy. |

## Intended bugs vs oracle behavior (summary)

| Bug | Symptom | Oracle fix |
|-----|---------|------------|
| Check-then-remove on shared `ArrayList` | Same connection to two threads | `synchronized (available)` around list peek/remove and add |
| Exception after `acquire` | Permits leaked; pool cannot reach max after recovery | `catch` releases permit and rethrows |
| Non-fair semaphore | Starvation possible under load | `Semaphore(maxSize, true)` |

---

*Generated for the Aquifer Pool benchmark package; update this file when the task or harness changes.*
