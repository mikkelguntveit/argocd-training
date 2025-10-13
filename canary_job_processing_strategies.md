# Canary Deployment Strategies for Background Job Processing

## 1. Introduction

This document outlines and compares two primary strategies for safely managing background job processing with Hangfire during a canary deployment. The core challenge is analogous to that of messaging: validating a new version of the `jobservice` worker with live production data without impacting the stability of the main production job processing system.

The central problem stems from the fact that all workers—stable and canary—poll a single, shared database backend where jobs are enqueued. This creates a "competing consumers" scenario between different versions of the code.

---

## 2. The Core Challenges with Hangfire

During a canary release, `v_old` and `v_new` workers are simultaneously polling the same Hangfire database. This creates three primary risks:

1.  **Incompatible Job Payloads:** A `v_new` API instance enqueues a job with a method signature that `v_old` workers do not recognize. This causes immediate job failures.
2.  **Incompatible Business Logic:** A `v_old` worker attempts to process a job in a context where the underlying application data has been changed by `v_new` code into a state it doesn't understand, leading to incorrect behavior or data corruption.
3.  **Long-Running Job Interruption:** A long-running job started by a `v_old` worker is interrupted by a deployment. The re-queued job is then picked up by a `v_new` worker, which may cause duplicated work if the job is not designed to be idempotent and resumable.

---

## 3. The Central Trade-Off: Developer Discipline vs. Infrastructure Isolation

As with messaging, the choice between the two strategies is a decision about where to invest engineering effort to guarantee safety.

*   **Strategy 1 ("Discipline-First")** invests in **people and process**. It relies on developers consistently following best practices for backward compatibility, idempotency, and resumability. Its safety net is human vigilance.
*   **Strategy 2 ("Infrastructure Isolation")** invests in the **platform and configuration**. It relies on Hangfire's built-in queueing system to create a dedicated, isolated lane for canary jobs. Its safety net is infrastructure that prevents version mismatches from occurring.

---

## 4. Strategy 1: The "Discipline-First" Approach (Shared Queue)

This strategy makes no changes to the Hangfire configuration. All workers, stable and canary, poll the default job queue in the shared database.

### How It Works

Safety is entirely predicated on a strict set of engineering disciplines:

*   **Strict Backward Compatibility:** Changes to job method signatures are treated as breaking API changes. The Expand/Contract pattern must be used over multiple deployments.
*   **Idempotent & Resumable Jobs:** All long-running jobs **must** be designed to be safely interrupted and restarted. They should use checkpointing to save progress and avoid duplicating work.
*   **Active Monitoring:** During the canary's validation window, the deploying team is responsible for closely monitoring the Hangfire Dashboard, specifically the "Failed" jobs list and the "Retries" count, for any anomalies.

### Pros

*   **Simplicity:** Requires no changes to Hangfire configuration or the deployment process.
*   **Realistic Testing:** The canary is tested under the exact conditions of a standard rolling update.

### Cons & Challenges

*   **High Risk of Human Error:** Safety is entirely dependent on developer discipline. A single non-backward-compatible change can cause job failures.
*   **Data Corruption Risk:** The most significant risk is from non-idempotent long-running jobs, where a standard deployment can cause data duplication.

---

## 5. Strategy 2: The "Infrastructure Isolation" Approach (Canary Job Queues)

This strategy uses Hangfire's built-in queueing mechanism to create an isolated channel for canary-specific jobs.

### How It Works

This model requires the application to be "context-aware."

1.  **Context-Aware Enqueuing:** The monolith's code is modified to read a `DEPLOYMENT_CONTEXT` environment variable. When running in a canary context, it explicitly enqueues jobs to a dedicated `"canary"` queue: `BackgroundJob.Enqueue<MyJob>(j => j.Execute(), "canary");`
2.  **Context-Aware Workers:** The canary `jobservice` deployment is configured at startup to **only** process jobs from the `"canary"` queue. The stable workers continue to only process jobs from the `default` queue.
3.  **Static Queue Management:** Unlike with RabbitMQ, Hangfire queues are not typically created dynamically. The `"canary"` queue would be a permanent fixture. The CI/CD pipeline **must** include a mandatory pre-deployment step to purge the `"canary"` queue to prevent a new canary from processing stale, failed jobs from a previous run.

### Pros

*   **Maximum Safety:** Provides perfect isolation. It is impossible for a canary-enqueued job to be picked up by a stable worker, eliminating version mismatch errors.
*   **Clear Signal:** Any job failures in the `"canary"` queue are unambiguously caused by the new code.

### Cons & Challenges

*   **Code & Config Modification:** Requires a one-time change to the job enqueuing logic and the `jobservice` startup configuration.
*   **Risk of Stale Jobs:** The process is dependent on a reliable CI/CD step to purge the static canary queue. If this step fails or is forgotten, the canary could process old jobs.
*   **Configuration Drift:** If a developer adds a new, specialized queue (e.g., `"high-priority"`), they must remember to also create a corresponding `"high-priority-canary"` queue and update the logic accordingly.

---

## 6. Recommendation

The recommendation for job processing aligns with the messaging strategy.

Begin with the **Discipline-First Approach**, as it requires no initial infrastructure or code changes. This choice must be paired with a significant investment in education and code reviews to ensure developers are rigorously following the principles of backward compatibility and, most importantly, building idempotent, resumable long-running jobs.

If this approach leads to unacceptable job failures or data corruption incidents, the organization will have a strong justification to invest the effort in implementing the **Infrastructure Isolation (Canary Job Queue)** model.
