## In one line

**Cost and high availability are the same dial, not two checklists**: every lever that makes a deployment cheaper (fewer replicas, scale-to-zero, spot capacity, a smaller model) also makes it less resilient, and every lever that makes it more available (redundancy, multi-zone spread, headroom) costs money that sits idle most of the time — the right setting is defined by your SLO and its error budget, not "cheapest" or "most available."

---

## Why it's needed

- 4.4 built the scaling mechanics (Pods, Deployments, Services, HPA, node autoscaler) and explicitly deferred cost/HA trade-offs to this lesson.
- The exam-relevant skill isn't a separate cost-tricks list and HA-tricks list — it's recognizing both are the same dial, tuned by the **SLO and error budget**.

---

## Cost lever 1: right-sizing and autoscaling to demand

- **Right-sizing** — the cheapest compute is what you never provision. AWS's example: "a reporting process might take five hours to run on a smaller server but one hour to run on a larger server that is twice as expensive... the smaller server incurs more cost over time." Same logic applies to GPU/precision/model-size choice (mechanics = 3.4).
- **Autoscale to demand** — AWS: "you can supply resources to match the workload demand at the time they're needed, this decreases the need for costly and wasteful over provisioning"; a skewed utilization metric hurts either way (degraded performance if under-provisioned, wasted spend if over-provisioned). HPA/node-autoscaler mechanics = 4.4; what matters here is cost: a `minReplicas` floor above zero means paying for idle GPU capacity around the clock.
- **Scale-to-zero** — the extreme case, a floor of zero. NVIDIA NIM Operator, via KServe + Knative: a NIM service can "scale to zero when unused" (`autoscaling.knative.dev/min-scale`, `max-scale`, concurrency `target`). The KEDA-based path uses "Horizontal/Vertical Pod Autoscalers with custom NIM metrics" but "cannot scale to zero by default." **The trade-off in miniature**: Knative eliminates idle-GPU spend for spiky/low-traffic services; KEDA's warm floor costs money at idle but has no cold start to pay for.

---

## Cost lever 2: throughput, caching, cheaper capacity

- **Throughput maximization** — at higher concurrency, "the system can use the batching effect to serve more requests efficiently, leading to increased throughput. However, this comes at the cost of increased latency." A Pareto front exists where "no other option provides a strictly higher throughput at the same or lower latency." NVIDIA's TCO method: first rule out configs that violate the latency constraint (example: TTFT ≤ 250 ms), then pick the highest-throughput survivor — "the most economical option." Cost then rolls up mechanically: yearly server cost / requests served per year = cost-per-1,000-prompts → cost-per-million-tokens via the input/output token ratio. Deep batch-size/quantization mechanics = 3.4.
- **Caching** — NIM Operator: "caching a model improves microservice startup time... a single cached model can serve multiple pods" — weights pulled once onto a persistent volume instead of re-downloaded/re-optimized per replica. Matters because it's what makes scale-to-zero and aggressive scale-out survivable, by shrinking the gap between a new replica scheduling and it actually serving traffic.
- **Cheaper capacity** — the lever with the sharpest availability cost. AWS's pricing spectrum: On-Demand (pay by the hour), Savings Plans/Reserved Instances ("savings of up to 75% off On-Demand"), Spot Instances ("savings of up to 90% off On-Demand," reclaimable "with a two-minute notice when AWS needs the capacity back"). AWS's own qualifier: Spot belongs on workloads "where the system can tolerate using a fleet of servers where individual servers can come and go dynamically, such as stateless web servers, batch processing" — an admission that the discount is bought with exactly the property an availability-sensitive service can't tolerate alone.

---

## Availability, defined: SLI, SLO, SLA, error budget

- Google's SRE book is the canonical framework:
  - **SLI** — "a carefully defined quantitative measure of some aspect of the level of service" (latency, error rate, throughput, or availability/yield — "the fraction of well-formed requests that succeed").
  - **SLO** — a target on that indicator, e.g. "99% of `Get` RPC calls will complete in less than 100 ms."
  - **SLA** — turns a missed SLO into a consequence: "an explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLOs."
  - Availability is expressed in "nines" — 99% = "2 nines," 99.999% = "5 nines"; Google's own Compute Engine target is "three and a half nines" (99.95%) — a deliberately chosen, not maximal, number.
- **Error budget**: "it's both unrealistic and undesirable to insist that SLOs will be met 100% of the time... an error budget is just an SLO for meeting other SLOs!" This makes availability a tractable engineering target — a 99.9% SLO is roughly **43 minutes/month** of budgeted downtime the team can spend on deploys, experiments, or absorbing real failures.
- **Caveat**: the SRE book also warns against over-chasing availability — "avoid absolutes... a system that is 'always' available... will probably turn out to be unnecessarily better than what users would be happy... to have." An unjustified 99.99% target is a design mistake, not a virtue.

---

## Availability levers: redundancy, health checks, failover

- **Redundancy** — the structural answer to single points of failure. AWS's Reliability Pillar: multi-AZ (minimum) or multi-Region (extreme resilience) — "distribute workload data and resources across multiple Availability Zones... this arrangement limits faults... to just the impacted Availability Zone" — run active/active or active/standby, with load balancers plus DNS health checks (short TTLs, "60 seconds or less") rerouting traffic automatically.
- **Health checks / automated failover** — what makes redundancy actually trigger rather than sit unused. Kubernetes' probe model:
  - **Readiness probe** — governs traffic: "if the readiness probe returns a failed state, the EndpointSlice controller removes the Pod's IP address... from all Services that match the Pod" (the same mechanic 4.4 leaned on for routing around not-yet-ready Pods).
  - **Liveness probe** — governs recovery: "if a container fails its liveness probe more times than the configured tolerance, the kubelet restarts that container" — catches a deadlocked-but-technically-running process.
  - **Startup probe** — protects slow-initializing containers (a large model still loading weights) from being killed prematurely by an impatient liveness check.
- **Static stability** (AWS) — redundancy sized to survive a *full zone* loss, not just one instance: "provision enough instances in each Availability Zone... to handle the workload capacity if one Availability Zone... were removed," rather than dynamically launching replacement capacity mid-failure — N+1 headroom that sits unused until it's the only thing standing between a zone failure and an outage.
- **Graceful degradation** — the last line of defense (a cached or last-known-good response, a cheaper model, a canned safe reply instead of a hard error), covered in depth by 2.4's retry/circuit-breaker/fallback pattern.

---

## The tension: redundancy costs money, cost-cutting threatens availability

- AWS says it directly: N+1 capacity across zones "must be weighed against the cost for this model and the business value of maintaining the workload under all resilience cases... it's less expensive to... rely on launching new instances in the case of a failure... but for large-scale failures... this approach is less effective." Every reserved replica is billed every hour it isn't needed.
- **Scale-to-zero** is the same trade-off from the cost side: zero idle spend, but the first post-idle request pays a cold start (Pod scheduled, image pulled, weights loaded). NIM cache narrows but doesn't eliminate this — a hard 250ms TTFT SLO means a multi-second cold start is a guaranteed violation, however rare.
- **Spot capacity**: up to 90% off, reclaimable "with a two-minute notice" precisely when demand (and reclamation risk) is highest — the discount and the availability risk are the same event viewed from two sides.
- **Throughput maximization** hits the same wall from the latency side — the highest-throughput, lowest-cost-per-token point is often *past* what the latency SLO allows, which is why the TCO method rules out SLO-violating configs before picking the cheapest survivor.
- **Resolution** (SRE book and cost frameworks converge independently): **size capacity, redundancy, and autoscaling floors to the SLO and its error budget** — not to cheapest-possible or most-available-theoretically-achievable. The SLO is the input; cost and redundancy are outputs.

---

## Trade-offs and pitfalls

- **Scale-to-zero cold starts breaking a latency SLO** — only safe if the workload's SLO tolerates the first-request-after-idle penalty; NIM cache shrinks the penalty, doesn't remove it.
- **Spot reclamation during a real traffic spike** — the moment demand spikes cluster-wide is exactly when cloud providers are most likely to reclaim Spot capacity.
- **Under-provisioning redundancy to save cost, then blowing the error budget on an ordinary zone failure** — a single-AZ deployment is cheap every day it doesn't fail and catastrophic the one day it does; static stability exists to make that day survivable.
- **Over-provisioning "just in case"** — chasing 99.99% or multi-region redundancy for a workload whose SLO doesn't demand it is waste, per the SRE book's own caution.
- **Cost-per-token looking cheapest at high batch/concurrency, but the latency SLO forcing a lower one** — the economically "optimal" point on a throughput curve is only valid if it also clears the SLO.
- **Misconfigured health checks compounding an incident** — Kubernetes' own docs warn that incorrect liveness-probe tuning "can lead to cascading failures... restarting of container under high load... increased workload on remaining pods" — the opposite of what a health check is supposed to buy.

---

## NVIDIA angle (read critically)

- NIM Operator gives real, usable knobs, not a fixed answer: KServe/Knative is a genuine scale-to-zero implementation (configurable concurrency targets, min/max-scale); KEDA is a genuine warm-floor alternative; NIM cache is a genuine, working cold-start mitigation. None choose the trade-off for you — SLO-driven sizing is still the operator's job.
- **Data Flywheel Blueprint distillation**: a fine-tuned Llama-3.2-1B model "achieved 98% of the tool-calling accuracy of the original 70B model," requiring "only one GPU to serve, compared to two GPUs needed for the original Llama-3.3-70B" — a real right-sizing-as-cost-lever result. **Caveat**: it's NVIDIA's own blueprint, measured on NVIDIA's own demo customer-service dataset with synthetic traffic, reported by NVIDIA — evidence the *distillation mechanism* works, not a transferable "98% accuracy at far fewer GPUs" guarantee for an arbitrary workload (the same caution 3.4 gives quantization accuracy claims).
- **TCO formula caveat**: NVIDIA's cost-per-1,000-prompts figure assumes "100% uptime," explicitly noting it "can be adjusted for the actual uptime fraction" — easy to skip past, but exactly where an availability target has to feed back into a cost estimate. A TCO number quoted without that adjustment implicitly assumes zero downtime, zero redundancy overhead, and zero cost for the HA levers above.

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Core framing | Cost and availability are one dial, not two checklists — every cost cut (fewer replicas, scale-to-zero, spot, smaller model) reduces resilience; every HA measure (redundancy, headroom, multi-zone) costs money |
| SLI / SLO / SLA / error budget | SLI = measured indicator (latency, errors, throughput, availability/yield); SLO = target on it; SLA = contract with consequences; error budget = the allowed rate of missing the SLO — "an SLO for meeting other SLOs" |
| Cost levers | Right-sizing (model/GPU/precision — 3.4); autoscale to demand incl. scale-to-zero (control-loop mechanics — 4.4); maximize throughput/batching to cut cost-per-token, bounded by the latency SLO; caching (KV-cache, NIM cache narrowing cold start); spot/reserved/committed-use capacity |
| Availability levers | Redundancy (N+1, multi-AZ/region, static stability); health checks (liveness/readiness/startup probes) driving automated failover; graceful degradation as the last resort (2.4); avoiding single points of failure |
| The tension, concretely | Scale-to-zero: zero idle cost vs. a cold start on the next request; Spot: up to 90% off vs. reclaimable exactly when demand spikes; static stability: N+1 headroom sits idle vs. survives a zone loss; high batch: cheapest cost-per-token vs. blown latency SLO |
| Resolution principle | Size capacity, redundancy, and autoscaling floors to the SLO and its error budget — not to "cheapest possible" or "maximally available"; NVIDIA's own TCO method (rule out SLO violations, then pick cheapest survivor) is the same pattern from the cost side |
| NVIDIA angle | NIM Operator gives real scale-to-zero (Knative) vs. warm-floor (KEDA) and caching knobs but doesn't set them for you; Data Flywheel's distillation savings and TCO's "100% uptime" assumption are vendor-reported / self-selected defaults — read critically before transferring |

---

## Quick self-check

1. Define SLI, SLO, SLA, and error budget in one sentence each, and explain why "an error budget is just an SLO for meeting other SLOs" is a useful way to think about it.
   **A:** SLI = a measured indicator of service quality (latency, errors, throughput, availability). SLO = a target value on that indicator. SLA = a contract with consequences attached to meeting/missing the SLO. Error budget = the allowed rate of missing the SLO. It's useful because it turns "be available" from a vague aspiration into a spendable budget the team can allocate to deploys, experiments, or absorbing real failures.
2. Using NVIDIA NIM Operator's KServe integration, contrast the cost and availability consequences of Knative-based scale-to-zero versus a KEDA-based deployment that "cannot scale to zero by default."
   **A:** Knative can scale a NIM service to zero replicas when unused, eliminating idle-GPU spend entirely, but pays a cold start (Pod scheduling, image pull, weight loading) on the first request after idle. KEDA keeps a warm floor of Pods via HPA/VPA on custom NIM metrics, so it costs money at idle but has no cold start to pay for on the next request.
3. What is "static stability" (per AWS's Reliability Pillar), and why does the guidance explicitly say it "must be weighed against the cost" rather than treating it as an unconditional best practice?
   **A:** Static stability means provisioning enough instances in each Availability Zone to absorb the full workload if one zone is removed, rather than dynamically launching replacement capacity during a failure. The cost trade-off is explicit because that N+1 headroom is billed continuously whether or not a zone failure ever happens, while the cheaper "launch replacements on failure" approach works for smaller failures but is less effective at large scale.
4. Walk through NVIDIA's TCO method for picking "the most economical option" from a latency-throughput curve. Why does the latency SLO have to be applied *before* selecting on cost, not after?
   **A:** First rule out every configuration that violates the latency constraint (e.g., TTFT > 250 ms), then among the survivors pick the one with the highest throughput — the most economical option. The SLO must come first because the highest-throughput, lowest-cost-per-token point on the curve is often past what the latency SLO allows; selecting on cost alone would silently pick a config that breaks the SLO.
5. Name two concrete pitfalls where optimizing purely for cost breaks an availability target, and one pitfall where optimizing purely for availability wastes money the workload's SLO didn't require.
   **A:** Cost-driven: scale-to-zero cold starts breaking a hard latency SLO on the first post-idle request, and Spot capacity being reclaimed exactly during a real demand spike. Availability-driven waste: over-provisioning multi-region redundancy or a 99.99% target for a workload whose actual SLO doesn't require it.

---

## Sources used

- [LLM Inference Benchmarking: How Much Does Your LLM Inference Cost? — NVIDIA Technical Blog](https://developer.nvidia.com/blog/llm-inference-benchmarking-how-much-does-your-llm-inference-cost/)
- [Service Level Objectives (Chapter 4) — Google SRE Book](https://sre.google/sre-book/service-level-objectives/)
- [Cost-effective resources — AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/cost-cereso.html)
- [Manage demand and supply resources — AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/cost-mandem.html)
- [REL10-BP01 Deploy the workload to multiple locations — AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_fault_isolation_multiaz_region_system.html)
- [REL11-BP05 Use static stability to prevent bimodal behavior — AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_static_stability.html)
- [Liveness, Readiness, and Startup Probes — Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Caching NIM Models — NVIDIA NIM Operator documentation](https://docs.nvidia.com/nim-operator/latest/cache.html)
- [KServe Support on NIM Operator — NVIDIA NIM Operator documentation](https://docs.nvidia.com/nim-operator/latest/kserve.html)
- [Build Efficient AI Agents Through Model Distillation With the NVIDIA Data Flywheel Blueprint — NVIDIA Technical Blog](https://developer.nvidia.com/blog/build-efficient-ai-agents-through-model-distillation-with-nvidias-data-flywheel-blueprint/)
