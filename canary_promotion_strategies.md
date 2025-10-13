# Canary Promotion and Deployment Models

## 1. Introduction

Once a canary deployment has been successfully validated against a subset of production traffic, a decision must be made on how to roll out the new version to 100% of users. This is the "promotion" phase. The chosen promotion model determines the speed, risk, and operational complexity of the final cutover.

This document compares two primary models for canary promotion: **Redeploy Stable** and **Promote Canary (Swap Routing)**.

---

## 2. Model A: Redeploy Stable

This is a conservative, robust, and operationally simple model. The canary is treated as a temporary, disposable environment used purely for validation. The promotion is achieved by performing a standard rolling update on the stable production environment.

### The Process

1.  **Deploy Canary:** A separate, parallel `v_new` canary stack is deployed. It is validated with a small percentage of traffic.
2.  **Make "Go" Decision:** The canary is deemed healthy and the decision is made to proceed with the full release.
3.  **Update Stable Environment:** The CI/CD pipeline initiates a standard `RollingUpdate` on the main, stable deployment. The only change is updating the container image from `v_old` to `v_new`.
4.  **Graceful Rollout:** Kubernetes gradually terminates `v_old` pods and replaces them with `v_new` pods within the stable deployment. During this time, both versions are running and serving traffic.
5.  **Tear Down Canary:** Once the stable deployment's rollout is 100% complete and healthy, the temporary canary stack is destroyed.

### Pros

*   **Operational Predictability:** The promotion process is a standard rolling update, which is a well-understood, native Kubernetes operation. The `stable` deployment is always the single source of truth for production configuration.
*   **Safety and Stability:** The stable deployment is already configured to handle 100% production load (replicas, HPA, etc.). New pods are added to a pre-warmed, correctly scaled environment. There is no risk of overwhelming a "cold" canary.
*   **Clean Separation of Concerns:** The canary's lifecycle is simple: it is created for validation and then destroyed. There is no complex logic to mutate it into a production environment.
*   **Immutable Infrastructure:** This model aligns well with the principles of immutable infrastructure, as the production environment is replaced rather than modified in-place.

### Cons & Challenges

*   **Slower Deployment Time:** The primary drawback is speed. The total time-to-production includes the time to deploy the canary *plus* the full time for the stable rolling update. If each step takes 10 minutes, the total deployment time is ~20 minutes.
*   **Higher Resource Usage (Briefly):** For a short period, you are running three sets of pods: the old stable pods, the new stable pods being rolled out, and the canary pods.

---

## 3. Model B: Promote Canary (Swap Routing)

This is a more aggressive, faster model that treats the canary environment as the next production environment in waiting. Promotion is achieved by gradually shifting all traffic to the canary and then decommissioning the old stable environment. This is often referred to as a Blue/Green deployment with a Canary-style traffic shift.

### The Process

1.  **Deploy Canary (as the "Blue" environment):** A full-scale `v_new` canary stack is deployed.
2.  **Gradual Traffic Shift:** The traffic manager (e.g., Istio) gradually shifts traffic from the stable environment (`v_old`) to the canary (`v_new`), from 5% -> 10% -> ... -> 100%. The health of the canary is monitored at each stage.
3.  **The "Promotion":** Once the canary is receiving 100% of traffic and is stable, it is now the de facto production environment.
4.  **Tear Down Stable:** The old stable (`v_old`) deployment is decommissioned.
5.  **(Optional but critical) Re-labeling:** The Kubernetes objects for the canary (Deployments, Services, etc.) are often relabeled to become the "stable" or "green" deployment for the next release cycle.

### Pros

*   **Faster Time-to-Production:** The deployment is complete as soon as the traffic shift reaches 100%. The total time is roughly the time for the initial canary deployment plus the traffic shifting window.
*   **Instant Rollback:** Rollback is extremely fast, often just a matter of reconfiguring the traffic manager to send 100% of traffic back to the old stable environment (which is kept running until the promotion is deemed successful).

### Cons & Challenges

*   **Operational Complexity:** This model introduces significant complexity at the most critical phase of the release.
    *   **The Scaling Problem:** The canary is likely running a minimal number of replicas. It must be scaled up to handle 100% of traffic *just before or during* the final traffic shift. This can lead to dropped requests or performance degradation if the scaling isn't perfectly timed.
    *   **The Identity & Monitoring Problem:** When the canary becomes the new stable, all monitoring dashboards, alerting rules, and log queries that are hardcoded to look for `deployment="monolith-stable"` will break. This requires a complex and risky process of relabeling live Kubernetes objects.
    *   **Configuration Drift:** The canary may have specific configurations (e.g., special feature flags, different resource limits). These must be mutated on the live system to match the standard production configuration, which is a stateful and potentially brittle operation.

---

## 4. The Impact of Messaging Strategy on Promotion Models

The choice of promotion model is heavily influenced by the chosen strategy for handling asynchronous messaging workers.

*   **With a "Discipline-First" Messaging Strategy (Context-Unaware):** If the application is context-unaware (sharing RabbitMQ queues), the **Promote Canary** model is more feasible. Since the canary is already configured identically to production, the promotion is primarily a networking change (swapping service labels or Istio rules) combined with scaling up the replica count. The operational challenges remain but are less complex.

*   **With an "Infrastructure Isolation" Messaging Strategy (Context-Aware):** If the application is context-aware (using dedicated canary queues via a `DEPLOYMENT_CONTEXT` variable), the **Promote Canary** model becomes significantly more complex. A simple networking swap is insufficient. The promotion would require:
    1.  Scaling up the canary.
    2.  Performing a live reconfiguration of the canary deployment to change `DEPLOYMENT_CONTEXT` from `canary` to `stable`.
    3.  This change would trigger a rolling restart of the canary pods so they connect to the stable queues.
    4.  Only then could the traffic be fully swapped.

    This multi-stage process is complex, risky, and negates much of the speed advantage over the **Redeploy Stable** model. This strong dependency is a primary reason why **Redeploy Stable** is the recommended starting point.

---

## 5. Comparative Summary

| Aspect                   | Redeploy Stable                                    | Promote Canary (Swap Routing)                             |
| ------------------------ | -------------------------------------------------- | --------------------------------------------------------- |
| **Primary Goal**         | Safety & Predictability                            | Speed & Fast Rollback                                     |
| **Deployment Time**        | Slower (~2x a single rollout)                      | Faster (~1x a single rollout)                             |
| **Risk Profile**         | Lower (uses well-understood operations)            | Higher (complex scaling & relabeling at cutover)          |
| **Operational Complexity** | Low (standard, declarative updates)                | High (requires sophisticated, stateful automation)         |
| **Resource Overhead**    | Higher during promotion (3 concurrent versions)    | Lower during promotion (2 concurrent versions)            |
| **Best For**             | Initial canary implementations; mission-critical systems where safety is paramount. | Mature platforms with very sophisticated automation; scenarios where rollback speed is the top priority. |

## 6. Recommendation

For the initial implementation of a canary process for a complex, critical system like the monolith, the **Redeploy Stable** model is strongly recommended.

The additional deployment time is a small and acceptable trade-off for the massive reduction in operational risk and complexity. This approach prioritizes the primary goal of a canary release: increasing safety and confidence. It allows the team to focus on validating the application's behavior without simultaneously having to solve the complex challenges of in-place promotion, live scaling, and object relabeling.

The **Promote Canary** model can be considered as a future optimization once the canary process is fully mature, the automation is robust, and the organization decides that the engineering investment is worth the time saved per deployment.
