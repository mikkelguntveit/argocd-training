# Canary Deployment Strategies for Asynchronous Messaging

## 1. Introduction

This document outlines and compares two primary strategies for safely managing asynchronous, message-based components (e.g., RabbitMQ workers) within a canary deployment process for the monolith. The core challenge is to validate the new version of a message worker with live production data without impacting the stability of the main production environment.

The choice of strategy has significant implications for infrastructure complexity, developer workflow, and the overall risk profile of a release.

---

## 2. The Core Challenge: Competing Consumers

During a canary release of a message worker, new pods (`v_new`) are introduced alongside existing stable pods (`v_old`). If both versions listen to the same message queue, they will "compete" for messages. This creates two primary risks:

1.  **Poison Pill (Forward Incompatibility):** A `v_new` canary producer publishes a message with a new format that the `v_old` stable workers cannot understand, causing them to crash. This can lead to a widespread outage.
2.  **Processing Failure (Backward Incompatibility):** A `v_new` canary worker consumes a message created by a `v_old` producer. If the new code is not fully backward-compatible, it may fail to process the message, leading to errors and potential message loss or processing delays.

The two strategies below address this challenge in fundamentally different ways.

---

## 3. Strategy 1: The "Discipline-First" Approach

This strategy posits that deployment safety can be achieved through rigorous engineering practices and operational vigilance, without requiring complex infrastructure changes. The canary and stable workers share the same production queues.

### How It Works

The canary workers are deployed and connect to the exact same RabbitMQ queues as their stable counterparts. The system relies on the canary's ability to coexist peacefully with the stable version.

This safety is predicated on a strict set of disciplines:

*   **Strict Backward Compatibility:** All changes to message contracts must be non-breaking. New fields must be optional, and existing fields cannot be renamed or have their types changed in a single deployment.
*   **Idempotent Consumers:** All workers must be designed to safely process the same message multiple times. This is critical because deployment-related failures can often lead to message redelivery.
*   **The Expand/Contract Pattern:** For any unavoidable breaking changes, a formal, two-stage deployment process must be used to ensure the system remains compatible at all stages.
*   **Active Monitoring:** During the canary validation window, the deploying team is responsible for closely monitoring Dead-Letter Queues (DLQs) for "hard" failures (crashes) and application/system metrics (CPU, latency, queue depth) for "soft" failures (performance degradation, freezes).

### Pros

*   **Simplicity:** Requires no changes to the messaging infrastructure. The production environment is the production environment.
*   **Realistic Testing:** The canary is tested under the *exact* conditions it will experience during the full rolling update, providing a high-fidelity validation.
*   **Faster Rollout (Potentially):** Since no infrastructure is being provisioned or torn down, the process can be quicker.

### Cons & Challenges

*   **Higher Risk Profile:** The blast radius of a faulty canary is significantly larger. A single "poison pill" message from a 1% canary can, in theory, take down 100% of the stable worker fleet.
*   **Heavy Reliance on Human Process:** Safety is tightly coupled to developer discipline. A single mistake in ensuring backward compatibility can lead to an incident.
*   **Detecting "Soft" Failures:** It's difficult to distinguish between normal system noise and subtle performance degradation caused by the canary when both versions are writing to the same logs and metrics dashboards.

### Relation to Deployment Models

This approach aligns well with the **Redeploy Stable** promotion model. The canary validates that the new code can coexist with the old. Once validated, the canary is destroyed, and the stable deployment is updated in a standard rolling update, confident that the mix of versions during the rollout will be safe.

Because this strategy results in a context-unaware application, it also makes the **Promote Canary** model more feasible (though still complex) compared to the Infrastructure Isolation strategy.

---

## 4. Strategy 2: The "Infrastructure Isolation" Approach

This strategy posits that safety is best achieved by physically isolating the canary environment from the stable environment. The canary workers connect to a parallel, temporary set of dedicated "canary queues."

### How It Works

This model requires the application to be "context-aware" and the deployment pipeline to be capable of automation.

*   **Context-Aware Application:** The application code is modified once to read an environment variable (e.g., `DEPLOYMENT_CONTEXT`). If the value is `canary`, the application automatically modifies the routing keys it publishes with (e.g., `orders.created.canary`) and the queue names it subscribes to (e.g., `OrdersQueue.canary`).
*   **CI/CD Automation:** The deployment pipeline is responsible for:
    1.  **Provisioning:** Before deploying the canary, it automatically creates the necessary `*.canary` queues and bindings in RabbitMQ.
    2.  **Deploying:** It deploys the canary pods with `DEPLOYMENT_CONTEXT` set to `canary`.
    3.  **Tear Down:** After the canary is terminated (pass or fail), the pipeline automatically deletes the temporary queues and bindings.

### Pros

*   **Maximum Safety:** Complete isolation. It is impossible for a canary-produced message to affect a stable worker, or vice-versa. This provides the smallest possible blast radius.
*   **Clear Signal:** Failures are unambiguous. If a message appears in the canary DLQ, it was 100% caused by the canary. Monitoring the canary's health is much clearer.
*   **Enables "Bake-In" Time:** Allows a canary to run for an extended period to test for memory leaks or other long-term issues without any risk to the stable system.

### Cons & Challenges

*   **Infrastructure Complexity:** This is the most significant drawback. The requirement for "CI/CD automation" is non-trivial and includes:
    *   **Automated Provisioning:** The pipeline needs robust logic to discover all queues used by the monolith and script the creation of their `*.canary` equivalents with perfectly mirrored settings (durability, TTLs, etc.).
    *   **Permissions Management:** The CI/CD system requires elevated administrative permissions on the RabbitMQ broker, which can be a security concern.
    *   **Reliable Teardown:** The pipeline *must* flawlessly clean up all temporary queues and bindings. Failure to do so will pollute the broker with hundreds of orphaned queues over time, creating significant operational debt.
    *   **Error Handling:** The automation must be resilient. It needs to handle cases where queue creation fails, and it must have a strategy for dealing with failures during the teardown phase.
*   **Handling of In-Flight Messages:** The pipeline must implement a "canary drain" procedure. After routing is shut off to the canary, the pipeline must monitor the canary queues and wait for them to be empty before tearing them down. Without this drain period, any messages still in the queue when the promotion happens would be lost.
*   **Code Modification:** Requires a small, one-time modification to the application's message publishing and subscribing logic.
*   **Divergence from Production:** The canary environment is not identical to the final production rollout (which will use shared queues during the rolling update). This could, in rare cases, mask bugs that only appear when versions are mixed.

### Relation to Deployment Models

This approach also works perfectly with the **Redeploy Stable** model. It provides an even higher degree of confidence before starting the final rollout.

Conversely, this strategy makes the **Promote Canary** model significantly more complex. The promotion would require a risky, live reconfiguration of the canary to change its context from `canary` to `stable`, negating much of the speed advantage of that model. This dependency strongly reinforces the recommendation to use the **Redeploy Stable** model.

---

## 5. Comparative Summary

| Aspect                 | Discipline-First Approach                                | Infrastructure Isolation Approach                          |
| ---------------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| **Primary Safety Net**   | Human Process & Vigilance                              | Infrastructure & Automation                              |
| **Blast Radius**         | High (canary can impact stable)                          | Very Low (canary is completely isolated)                   |
| **Setup Complexity**     | Low (no infra changes)                                   | High (requires CI/CD automation)                         |
| **Operational Overhead** | High (requires intense monitoring during canary)         | Low (failures are isolated and clear)                    |
| **Test Fidelity**        | High (mimics rolling update conditions exactly)          | Medium (environment is slightly different from prod)       |
| **Best For**           | Mature teams with strong discipline; less critical systems. | Mission-critical systems; teams desiring maximum safety.   |

## 6. Recommendation

The recommended path is to begin with the **Discipline-First Approach**. It represents the minimum viable change to the current process while still introducing the core concepts of canary validation. This allows the team to gain experience with canary releases without the immediate overhead of building complex infrastructure automation.

This strategy should be adopted with the explicit acknowledgment of its risks. The implementation must include robust, mandatory runbooks for developers on how to monitor DLQs and performance metrics during their validation window.

If this approach proves to be too risky in practice—if canaries are found to be causing instability in the stable environment—the organization will have a strong, evidence-based justification to invest in the **Infrastructure Isolation Approach** as a future evolution of the platform.
