# DevOps-Cloud-Platform-Engineering
DevOps, Cloud &amp; Platform Engineering 
Here are clear answers to all 10 DevOps fundamentals:

---

**1. CPU bottleneck vs I/O bottleneck**

- **CPU bottleneck**: The processor can't keep up with work. You'll see high `%cpu` in `top`/`htop`, long run queues (`uptime` load average > CPU count), and processes spinning.
- **I/O bottleneck**: Disk or network is the limiting factor. CPU is mostly idle but processes are in **D state** (uninterruptible sleep). Tools: `iostat -x` shows high `%util` and `await`, `iotop` shows which process is causing it.

---

**2. Why systems fail even when CPU and memory look fine**

Many failure modes don't touch CPU/RAM:
- **File descriptor exhaustion** (`ulimit -n`)
- **Thread/connection pool saturation**
- **Network latency or packet loss**
- **Disk I/O saturation** (even with free space)
- **Database lock contention**
- **DNS resolution failures**
- **Downstream service timeouts**

Always check: open connections, FD limits, network errors, and dependency health.

---

**3. What actually happens when a Kubernetes Pod restarts**

1. Kubernetes detects failure (via liveness probe or crash)
2. Container is **killed** (SIGTERM → grace period → SIGKILL)
3. A **new container** starts from scratch in the same Pod
4. Pod IP stays the same, but **in-memory state is lost**
5. If `restartPolicy` allows, it restarts with **exponential backoff** (CrashLoopBackOff)
6. Persistent volumes are remounted; ephemeral storage is wiped

---

**4. Readiness vs Liveness probes**

| | Readiness | Liveness |
|---|---|---|
| **Purpose** | Is the app ready to receive traffic? | Is the app still alive? |
| **Failure action** | Removed from Service endpoints | Container is **restarted** |
| **How it breaks prod** | Too strict → pod never gets traffic | Too strict → endless restart loops; too loose → dead app stays up serving errors |

**Rule of thumb**: Liveness should only fail if the process is truly stuck/deadlocked. Readiness can fail during startup or DB connection loss.

---

**5. Infrastructure as Code (IaC) — problem it solves and when it's dangerous**

**Solves**: Manual, inconsistent, undocumented infrastructure. IaC gives you version control, repeatability, and auditability.

**Becomes dangerous when**:
- People make **manual changes** alongside IaC (drift)
- State files are shared without locking (race conditions)
- Secrets are hardcoded in code
- Destroy operations run without proper review (`terraform destroy`)
- No testing/staging — changes go straight to prod

---

**6. How Terraform state works and what happens when it drifts**

Terraform keeps a **state file** (`terraform.tfstate`) mapping your config to real infrastructure. On every `plan`/`apply`, it compares desired config → state → actual cloud resources.

**Drift** happens when real infrastructure changes outside Terraform (manual edits, other tools). Result:
- `terraform plan` shows unexpected changes
- Terraform may **destroy and recreate** resources it thinks are wrong
- Fix with `terraform refresh` or `terraform import`

Always use **remote state** (S3 + DynamoDB lock) in teams to prevent corruption.

---

**7. Horizontal vs Vertical scaling, and when to avoid autoscaling**

- **Vertical** (scale up): Bigger machine. Simple but has limits and causes downtime.
- **Horizontal** (scale out): More instances. Resilient and theoretically unlimited, but your app must be **stateless**.

**Avoid autoscaling when**:
- Your app has **long warmup times** (new instances aren't ready fast enough)
- **Stateful workloads** that can't be distributed (e.g., some DBs)
- Cost is unpredictable and you need a hard cap
- The bottleneck is a **downstream dependency** (scaling your app won't help)

---

**8. Why deployments succeed but users still see errors**

- **CDN/cache** still serving old broken content
- **Database migrations** not run or rolled back
- **Feature flags** not toggled
- **Dependent services** not yet updated (API contract mismatch)
- **DNS TTL** still pointing to old servers
- **Health checks pass** but specific code paths are broken
- **Session/cookie** issues from the old version

A green deployment pipeline only tests *your* artifact — not the whole system.

---

**9. Rollback vs Hotfix during an incident**

| | Rollback | Hotfix |
|---|---|---|
| **When** | Bug is in the new release; old version is stable | Old version also has the issue, or rollback isn't possible |
| **Speed** | Faster — just redeploy previous artifact | Slower — requires coding, review, testing |
| **Risk** | Low if previous version is known-good | Higher — code written under pressure |
| **Use when** | Data migrations haven't run yet (or are reversible) | DB schema changes make rollback unsafe |

**Default to rollback**. Only hotfix when rollback is impossible or makes things worse.

---

**10. Metrics vs Logs vs Traces**

| | What it is | Start here when... |
|---|---|---|
| **Metrics** | Aggregated numbers over time (CPU %, req/s, error rate) | You need to know *that* something is wrong (alerting, dashboards) |
| **Logs** | Timestamped event records | You know *where* it's broken and need to see *what happened* |
| **Traces** | End-to-end request journey across services | You know a request is slow/failing but don't know *which service* is the culprit |

The golden workflow: **Metrics** alert you → **Logs** give context → **Traces** pinpoint the root cause across distributed systems.
