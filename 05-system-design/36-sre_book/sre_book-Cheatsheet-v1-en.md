# sre_book High-Density Core Card System (English Version)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Sentence Essence**: SRE (Site Reliability Engineering) is Google's software-engineered solution to operations, with the essence of using software engineering practices to manage, automate, and optimize large-scale distributed systems under the constraint of error budgets, balancing the velocity of feature releases against the stability of service delivery.
*   **L1 Four-Sentence Logic**:
    1.  **Metric Definition & Error Budget**: Defining clear Service Level Indicators (SLIs) and Service Level Objectives (SLOs) to derive an Error Budget, which acts as the quantitative governance tool to determine release speed and alert priorities.
    2.  **Multi-Burn-Rate Alerting & Response**: Constructing multi-window multi-burn-rate alerting algorithms to capture critical failures while ignoring transient fluctuations, and designing structured on-call runbooks to minimize MTTR.
    3.  **Cascading Failure Prevention & Throttling**: Protecting backends from cascading outages through active server-side load shedding and graceful degradation, paired with client-side adaptive throttling formulas.
    4.  **Idempotent Retries & Circuit Breaking**: Mitigating transient faults within service boundaries through randomized exponential backoff and jitter retries, supplemented by three-state circuit breakers.
*   **L2 Core Data Flow Topology**:
    *   `User Request` ➜ `Edge Load Balancer` ➜ `Gateway Router` ➜ `SRE Prometheus SLI Monitoring` ➜ `Multi-Burn-Rate Rule Match` ➜ `Error Budget Burn Alert` ➜ `Alertmanager Alert routing` ➜ `On-call SRE Paged` ➜ `Runbook mitigation: Dynamic load shedding` ➜ `Drop low-priority tasks` ➜ `Healthy payment transactions complete` ➜ `Post-incident Blameless Postmortem`.

---

## ⚔️ High Concurrency & Availability Architecture Trade-offs Matrix (Page 1 Bottom)

| Design Dimension | Traditional Monolithic Intuition (⚠) | Underlying System Physical Reality (★) | Core Architectural Trade-off & Evolution |
| :--- | :--- | :--- | :--- |
| **Availability Metrics** | Setting a 100% perfect uptime goal as the only target for operational stability. | Physical systems cannot achieve 100% reliability; pursuing it yields infinite costs and halts feature velocity. | **Error Budget Governance**: Acknowledge and quantify acceptable failure rates (SLO), releasing budget for rapid releases. |
| **Monitoring & Alerts** | Sending SMS or email alerts to the on-call engineer immediately whenever any error occurs. | Transient spikes and secondary errors trigger alert fatigue (storms), causing engineers to miss critical outages. | **Burn Rate Alerts**: Alerting only when the rate of error budget consumption threatens the monthly SLO. |
| **Retries & Backoff** | Launching immediate retries on the client to boost transaction success rate when downstream fails. | Under heavy load, thousands of clients retrying simultaneously trigger a thundering herd, collapsing downstreams. | **Exponential Backoff with Jitter**: Stretching retry intervals exponentially, injecting random jitter, and limiting budget. |
| **Overload Mitigation** | Spawning threads and request queues infinitely to buffer all incoming traffic under high load. | Memory bloat and CPU context switching consume host performance, exhausting resources and triggering cascade failure. | **Load Shedding**: Grid/service layer actively drops low-priority requests, backed by client-side local throttling. |

---

## 📂 M1: SLI, SLO & Error Budget (Cards 1-4)

#### Card 1. Service Level Indicator (SLI) Principles: Latency, Error Rate, Throughput, and Saturation
*   **Technical Mechanism**: SLI is a quantitative measure of service performance (e.g., `Successful Requests / Total Requests`). The four golden signals of monitoring are: **Latency** (time taken to serve a request), **Traffic** (demand rate), **Errors** (rate of failing requests), and **Saturation** (fullness of the most constrained system resource).
*   **Application Strategy**: SLIs must reflect user experience (e.g., client-side perceived latency) rather than bare host CPU utilization, ensuring metrics correlate directly with user satisfaction.

#### Card 2. Service Level Objective (SLO) Formulation and User Experience Thresholds
*   **Technical Mechanism**: SLO is a target value for an SLI (e.g., 99.9% of requests have latency < 200ms). Setting SLOs should target the user happiness boundary: below this threshold, users complain. Pursuing excessively high SLOs exponentially restricts development velocity and raises server cost.
*   **Application Strategy**: Drive consensus among Product, Dev, and SRE that SLO is not a pursuit of perfection, but a balance of user happiness and speed.

#### Card 3. Error Budget Mathematical Derivation, Burn Speed, and Release Velocity Governance
*   **Technical Mechanism**: $\text{Error Budget} = 1 - \text{SLO}$ (e.g., 99.9% SLO yields a 0.1% monthly error budget, equivalent to 43.8 minutes of downtime/month). The error budget balances dev (velocity) and ops (reliability): as long as budget remains, dev releases freely; if budget burns out, releases freeze to focus on stability.
*   **Application Strategy**: Implement error budget as a hard organizational rule, turning it into an automated release gatekeeper to eliminate inter-departmental conflict.

#### Card 4. White-box vs. Black-box Monitoring: Scopes, Use Cases, and Blind Spots
*   **Technical Mechanism**: **White-box monitoring** queries internal metrics (e.g., heap memory, GC pauses, pool sizes) to diagnose root causes and predict capacity. **Black-box monitoring** tests external behavior (e.g., polling ping requests) to detect ongoing outages.
*   **Application Strategy**: Use white-box for debugging and capacity alerts, and reserve black-box for waking up on-call engineers to minimize false alerts from transient single-node spikes.

---

## 📂 M2: Alerting & Incident Management (Cards 5-9)

#### Card 5. Traditional Threshold Alerting Limitations and Multi-window Multi-burn-rate Alerting
*   **Technical Mechanism**: Naive threshold alerts (e.g., "Error rate > 1%") generate excessive false alarms (on transient drops) or miss slow-bleed leaks. SRE burn-rate alerting monitors budget depletion rates. A 14.4x burn rate means 2% of the monthly budget is consumed in 1 hour. By analyzing 1-hour and 6-hour windows concurrently, SRE captures both fast disasters and slow drifts.
*   **Application Strategy**: Rewrite Prometheus alerting rules to base alerts on SLO burn rates, drastically reducing alarm fatigue and pager noise.

#### Card 6. Alert Noise Reduction: Separating Pager Alerts from Non-Urgent Tickets
*   **Technical Mechanism**: Restrict **Pager** alerts (which wake up engineers via phone/SMS) only to incidents that require immediate human intervention and directly threaten SLOs. Downclass secondary issues, covered by self-healing redundancy, to automated **Tickets** processed during business hours.
*   **Application Strategy**: Auditing alert rules periodically; if an alert doesn't require immediate action, it must be removed from the pager system.

#### Card 7. Structured On-call Incident Response and MTTR Compression
*   **Technical Mechanism**: During incidents, the on-call team adopts structured roles: **Incident Commander** (IC, directs response and delegates), **Communications Lead** (updates stakeholders and public), and **Operations Lead** (isolates faults). This prevents engineers from duplicate debugging or leaving communications ignored.
*   **Application Strategy**: Compile detailed Runbooks for all critical alerts, guiding on-call actions step-by-step to compress troubleshooting time.

#### Card 8. Blameless Postmortem and 5 Whys Root Cause Analysis
*   **Technical Mechanism**: Postmortems assume engineers made good-faith decisions based on available information. It focuses on "why the system failed" rather than "who broke it." The 5 Whys recursively trace the root cause (e.g., why DB crashed -> why query bloated -> why index was missing -> why migration was skipped -> why review policy was weak).
*   **Application Strategy**: Build a blameless culture to encourage team honesty in reporting defects, converting outages into structural stability improvements.

---

## 📂 M3: Overload Control & Cascading Failures (Cards 9-13)

#### Card 9. Cascading Failure Mechanisms: Queue Bloat, Memory Leaks, and Deadlocks
*   **Technical Mechanism**: Cascading failure is a positive feedback loop. When a subset of nodes crashes due to overload, traffic automatically redirects to the remaining healthy nodes, exceeding their capacity and triggering OOMs, queue delays, and lock deadlocks, collapsing the entire system.
*   **Application Strategy**: Design defensive gateways that proactively drop excess requests to prevent the cluster from entering a cascading collapse.

#### Card 10. Google's Adaptive Client Throttling Formula and SDK Integration
*   **Technical Mechanism**: When backends overload, client retries can saturate network cards even if the gateway drops requests. Adaptive throttling calculates rejection probability locally: $P_{\text{reject}} = \max\left(0, \frac{\text{requests} - K \times \text{accepts}}{\text{requests} + 1}\right)$, where $K$ is the multiplier (typically 2.0). Rejections happen on the client before hitting the network.
*   **Application Strategy**: Embed this adaptive throttling algorithm in gRPC client interceptors and client libraries to insulate overloaded backends from probe storms.

#### Card 11. Server-side Graceful Degradation (Load Shedding) by Traffic Prioritization
*   **Technical Mechanism**: Under high CPU utilization or queue congestion, servers drop incoming requests based on priority tags (e.g., `X-Priority`). Dropping non-essential requests (e.g., recommendation feeds) frees up CPU cycles to process core transactions (e.g., payment processing).
*   **Application Strategy**: Configure dynamic load-shedding policies at the API gateway layer to drop non-critical traffic during peak workloads.

#### Card 12. Distributed Timeout Budgets and Deadline Propagation
*   **Technical Mechanism**: In long call chains (A -> B -> C), if A has a 5-second timeout, but B takes 4.9 seconds to call C, A has already timed out. Continuing computation on C wastes CPU resources. Deadline propagation sends the remaining timeout budget inside the request context, allowing downstream nodes to abort early if budget is exhausted.
*   **Application Strategy**: Enable Deadline Propagation in RPC frameworks to eliminate zombie computations on overloaded backends.

#### Card 13. Slow Call Mitigation and Self-healing Warm-up Algorithms
*   **Technical Mechanism**: Newly launched instances (lacking JIT compilation or cache warming) can freeze if they receive full production traffic immediately. A Warm-up algorithm scales incoming traffic weight linearly or exponentially (e.g., from 10% to 100% over 5 minutes) to protect cold instances.
*   **Application Strategy**: Configure warm-up properties in Kubernetes Service routing or ingress load balancers to eliminate rolling update timeout spikes.

---

# Page 2 (Distributed, Async, Rate Limiting & Resiliency)

## 📂 M4: Transient Failures & Retries (Cards 14-18)

#### Card 14. Retry Storm Mechanisms and Client Retry Budget Limits
*   **Technical Mechanism**: When a downstream service degrades, multiple layers retrying calls amplify traffic exponentially (e.g., $3^4 = 81$ times for a 4-layer call stack), creating a retry storm. A Retry Budget restricts retry requests to a fixed ratio (e.g., max 10% of total outbound requests) locally.
*   **Application Strategy**: Enforce that retries only occur at the top-level client or use retry budgets in RPC client interceptors.

#### Card 15. Exponential Backoff with Jitter for Anti-Thundering Herd Spikes
*   **Technical Mechanism**: Retries without delays cause clients to bombard backends simultaneously. Exponential backoff stretches retry intervals ($2^{\text{attempt}}$). Adding randomized "Jitter" (e.g., scattering a 2-second delay randomly between 0.2s and 2s) spreads client attempts evenly over time.
*   **Application Strategy**: Enforce exponential backoff with jitter on all distributed client-server communication channels to protect recovering instances.

#### Card 16. Idempotency Design and Distributed Locking in Retry Flows
*   **Technical Mechanism**: Retrying a write operation requires that the operation is idempotent. Downstream handlers verify the request's unique ID (e.g., `message_id`) using Redis `SETNX` or database unique indexes to block duplicate processing.
*   **Application Strategy**: Ensure all mutable RPC interfaces exposed to retries enforce strict idempotency contracts at the API layer.

#### Card 17. Feature Toggles for Dynamic Degraded Runtime Configurations
*   **Technical Mechanism**: Wrap complex or resource-heavy features inside Feature Toggles. During extreme load spikes, operators toggle configs via central configuration servers (e.g., Apollo) without code changes or restarts to disable heavy services.
*   **Application Strategy**: Maintain pre-configured degradation playbooks, disabling non-essential UI sections dynamically during peak workloads.

---

## 📂 M5: Capacity Planning & Load Balancing (Cards 18-21)

#### Card 18. Global Server Load Balancing (GSLB) DNS, Anycast, and Multi-Center Failover
*   **Technical Mechanism**: GSLB manages top-level ingress traffic. Anycast IP routing forwards requests to the nearest edge router; GeoDNS resolves domain queries to physically close data centers. SREs dynamically adjust GSLB traffic weights (e.g., 40:60) to balance regional capacities.
*   **Application Strategy**: Shift GSLB routing targets instantly during fiber cuts or power failures, routing traffic around impacted regions.

#### Card 19. Capacity Planning and Limit Testing: Watermarks and Safe Scaling Thresholds
*   **Technical Mechanism**: SREs run load tests (e.g., replaying shadow traffic or scaling single-instance loads to failure) to find real inflection watermarks. Safe thresholds (e.g., autoscale at 70% CPU or 65% bandwidth) are set based on grow forecasts.
*   **Application Strategy**: Drive infrastructure provisioning through limit-test data rather than guesswork, optimizing server budgets.

#### Card 20. Client-side Load Balancing and Connection Subsetting
*   **Technical Mechanism**: In huge clusters (e.g., 10000 clients and 1000 servers), full-mesh connections force each server to maintain thousands of TCP connections. Subsetting limits each client to connect to a random subset (e.g., 20) of backend instances.
*   **Application Strategy**: Eliminate TCP connection storms in massive clusters, reducing discovery catalog overhead on service mesh control planes.

#### Card 21. Graceful Shutdown and Connection Draining for Zero-Downtime Updates
*   **Technical Mechanism**: Force-killing processes during updates breaks active requests. Graceful shutdowns set instance weight to 0, unregister from discovery, transition to Draining (rejecting new connections while finishing active requests), and kill the process only when empty.
*   **Application Strategy**: Enable graceful shutdowns in application frameworks and container specs to eliminate 502/504 errors during rolling updates.

---

## 📂 M6: SRE Automation & Toil Elimination (Cards 22-28)

#### Card 22. Defining Toil and SRE's 50% Engineering Time Budget Rule
*   **Technical Mechanism**: Toil is operational work that is manual, repetitive, automatable, and lacks long-term value (e.g., manual host restarts, provisioning credentials). Google SRE rules cap toil at 50%, forcing engineers to spend at least 50% of their time on engineering to automate toil away.
*   **Application Strategy**: Track team time allocation; if toil exceeds 50%, reassign dev resources to reconstruct automation pipelines.

#### Card 23. Infrastructure as Code (IaC) and GitOps in Auto-Remediation Environments
*   **Technical Mechanism**: Define configs, infrastructure, and access rules in code (e.g., Terraform, YAML). Changes occur only via Git PRs. GitOps controllers (e.g., ArgoCD) watch the Git repo and reconcile cluster states to match Git definitions.
*   **Application Strategy**: Eliminate configuration drift on production hosts, enabling reproducible, version-controlled disaster recovery.

#### Card 24. Progressive Canary Deployments and Automated SLI Rollbacks
*   **Technical Mechanism**: Releases deploy to a small subset (e.g., 1%) of instances (Canary nodes). Metric monitors compare Canary SLIs (errors, latency, OOMs) against Baseline nodes. If canary SLIs degrade, automatic rollback is triggered.
*   **Application Strategy**: Implement progressive delivery to restrict the blast radius of software regressions.

#### Card 25. Auto-Remediation Script Limits and Positive Feedback Loop Blockers
*   **Technical Mechanism**: Remediation systems (e.g., StackStorm) automate recovery actions (e.g., restarting containers on memory leaks). To prevent loops (e.g., cascading restarts during database downtime), remediation scripts must include execution rate limits and circuit breakers.
*   **Application Strategy**: Define clear exit criteria and handover points to human operators in all auto-healing pipelines.

#### Card 26. Disaster Recovery Testing (Google DiRT) and Ingress Failover Scenarios
*   **Technical Mechanism**: Google DiRT simulates catastrophic failures (e.g., fiber cuts, data center power outages) without warning. This verifies multi-region failovers under pressure and test SRE response coordination.
*   **Application Strategy**: Conduct regular fire drills; disaster recovery plans not tested in production drills are assumed to be non-functional.

#### Card 27. Sublinear Operational Scaling Math and SRE Efficiency Metrics
*   **Technical Mechanism**: SRE targets sublinear scaling, ensuring operational overhead grows sublinearly with system size: $\text{SRE Staff} \propto \log(\text{Scale})$. By constant engineering and automation, a 10x scale growth should not require 10x staff.
*   **Application Strategy**: Use sublinear scaling metrics to evaluate SRE team efficiency and automation platform success.

#### Card 28. Change Management Principles: Small Changes, High Observability, and Fast Rollback
*   **Technical Mechanism**: Over 70% of outages stem from changes. Change management rests on three pillars: **Small changes** (reduces complexity), **High observability** (early detection via SLI sampling), and **Fast rollback** (idempotent rollback scripts).
*   **Application Strategy**: White-box all change deployment pipelines, ensuring changes are incrementally rolled out.

---

## ⚔️ google / sre-book Core Parameters & Troubleshooting Dictionary (Page 2 Bottom)

### T1: google / sre-book Core Parameters
*   **`google-client-throttle-k: 2.0`**: Adaptive client throttling factor. $K=2.0$ permits client requests to double the server accepts; smaller $K$ tightens throttling.
*   **`burn-rate-thresholds: 14.4 (1h) / 6.0 (6h)`**: Multi-burn-rate alert configuration. Triggers if 2% of budget is lost in 1h, or 5% in 6h.
*   **`subsetting-backend-size: 20`**: Client connection subset size, limiting client TCP sockets to 20 random backends.
*   **`graceful-shutdown-drain-timeout: 30s`**: Timeout to drain active connections on termination before hard kill.

### T2: System-level Troubleshooting & Diagnostics
*   Querying edge CDN cache status and TTFB headers using curl formatting parameters:
    `curl -o /dev/null -s -D - -H "Pragma: no-cache" https://example.com/api/v1/health`
*   Monitoring host context switches and CPU sy/us split to diagnose thread storm congestion:
    `vmstat 1 10 | awk '{print "US:" $13 " SY:" $14 " INTR:" $17 " CS:" $18}'`
*   Calculating monthly error budget consumption rate via Prometheus (assuming 99.9% SLO):
    `1 - (sum(increase(http_requests_total{status=~"5.."}[30d])) / sum(increase(http_requests_total[30d]))) / 0.001`
*   Counting active and established TCP sockets on host to verify subsetting distribution:
    `ss -s | grep -E "TCP|ESTAB"`
