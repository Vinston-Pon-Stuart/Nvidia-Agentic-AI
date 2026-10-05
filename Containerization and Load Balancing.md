## In one line

**Containerization and load balancing** = the concrete Kubernetes machinery — Docker images, Pods/Deployments/Services/Ingress, an HPA + node autoscaler, and a GPU-specific layer — that turns "a stateless service" (1.8) into N replicas actually absorbing real traffic behind a load balancer.

---

## Why it's needed

- 1.8 established *why* an agent service needs to be stateless/horizontally scalable; 4.1 established the *framing* (package each agent as a deployable service, "mechanics belong to 4.4"). This lesson **is** 4.4 — the concrete machinery.
- Five pieces: Docker images (portable deployment unit), four K8s objects (Pod, Deployment, Service, Ingress), two autoscalers (HPA + node/cluster autoscaler), L4/L7 load balancing (what actually spreads traffic), and a GPU-specific layer (device plugin, GPU Operator, NIM Operator) since GPUs aren't natively understood by Kubernetes.
- Cost/HA trade-offs = 4.5's territory; this lesson stays on scaling mechanics.

---

## Containers: the portable, immutable unit under horizontal scaling

- **What it is**: Docker's own definition — "a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another." An **image** is the built artifact (code, runtime, tools, libraries, settings); a **container** is an image that's running ("images become containers when they run").
- **What it does**: fixes "works on my machine" — ships the environment along with the code, every time.
- **Portability**: the same image runs identically on a laptop, CI runner, or GPU node — an industry-standard format (later the Open Container Initiative spec), "lightweight" because containers "share the machine's OS system kernel," unlike a VM which needs "a full copy of an operating system... tens of GBs" and can be slow to boot; a container image is typically "tens of MBs."
- **Immutability — the key property for scaling**: an image is fixed/versioned — you build a new one and replace the old, never patch in place. Ten replicas of the same image are, by construction, identical and interchangeable. This is the literal substrate under 1.8's claim that a stateless component "can be freely replicated."

---

## Kubernetes core objects: Pod, Deployment, Service, Ingress

NVIDIA's glossary: Kubernetes is "an open-source platform for automating container orchestration... automates service discovery and load balancing... monitors container health, restarts containers that fail." Four objects do the concrete work.

- **Pod** — the smallest deployable unit: "a group of one or more containers, with shared storage and network resources." Containers in a Pod are co-located/co-scheduled, share an IP, and reach each other via `localhost`. Pods are "ephemeral, disposable" — rarely created directly; a dead Pod isn't healed, it's replaced by a new Pod with a new IP.
- **Deployment** — the controller managing replicated Pods; a declarative desired state (image, `.spec.replicas`) the controller drives the actual state toward.
  - **Rolling updates** (default strategy): `maxUnavailable` (25% default — at least 75% of desired Pods stay up) and `maxSurge` (25% default — up to 125% can exist temporarily) replace Pods gradually. `Recreate` kills all old Pods first — faster, but a real availability gap.
  - **Rollback** — every template change creates a new revision; `kubectl rollout undo` reverts to a prior one — the built-in safety net for a bad deploy.
- **Service** — fixes Pod ephemerality: a stable **cluster IP** that tracks which Pod IPs currently match its label selector, so callers never track individual Pod addresses. `ClusterIP` (default, internal only), `NodePort` (+ static port on every node), `LoadBalancer` (external cloud LB). This *is* Kubernetes' **Layer 4 (transport) load balancing**, implemented by `kube-proxy`.
- **Ingress** — the HTTP(S)-specific layer above Service: routes by hostname/URL path, consolidates routing rules into one resource. Needs an **Ingress controller** to do anything ("only creating an Ingress resource has no effect") — NGINX, Traefik, or cloud controllers implement it differently. Supports TLS termination; "more advanced load balancing concepts (e.g. persistent sessions, dynamic weights)" aren't exposed through Ingress itself.

---

## Autoscaling: HPA and node/cluster autoscaling

- **Horizontal Pod Autoscaler (HPA)** — changes `.spec.replicas` automatically. A control loop, not continuous — default sync period **15s**; each tick computes `desiredReplicas = ceil(currentReplicas × currentMetricValue / desiredMetricValue)`.
  - Scales on **resource metrics** (CPU/memory via `metrics.k8s.io`, usually the `metrics-server` add-on) or **custom metrics** (via `custom.metrics.k8s.io`, a vendor-specific adapter) — the path that lets HPA scale on something like GPU queue depth instead of CPU%.
  - Anti-**flapping**: `stabilizationWindowSeconds` (default **5 min** on scale-down, **0** on scale-up) uses the highest recent recommendation; a tolerance band (default **10%**) suppresses action inside it.
  - Horizontal (more Pods) is explicitly distinguished from vertical (more CPU/mem per existing Pod) scaling — HPA does the former.
- **Node (cluster) autoscaling** — needed because HPA alone only helps if the cluster has spare node capacity.
  - **Provisioning**: unschedulable Pods trigger new nodes. **Consolidation**: removes underutilized nodes (the inverse) to control cost.
  - Two named implementations: **Cluster Autoscaler** (adds/removes nodes from pre-configured cloud node groups) and **Karpenter** (auto-provisions individual nodes directly against constraints, also manages node lifecycle/refresh/upgrade).
  - **Composition**: HPA creates Pods that don't fit → the node autoscaler notices unschedulable Pods and adds nodes (reversing on the way down). Two layers operating on different objects (Pods vs. Nodes) and different signals (a metric vs. unschedulable Pods) — exam-relevant.

---

## Load balancing: L4 vs L7 — what makes scaling actually work

- NVIDIA's Triton/TensorRT-LLM Kubernetes walkthrough: "Layer 4 load balancing operates at the transport level and manages traffic based on network information, while Layer 7... operates at the application level and uses protocols to make decisions based on the content of each message."
- **L4** = a Kubernetes Service — `kube-proxy` spreads TCP/UDP connections across matching Pod IPs with no knowledge of request content.
- **L7** = Ingress or a dedicated L7 load balancer (Traefik, NGINX Plus, cloud application LB) — routes on HTTP host/path/headers, can terminate TLS.
- **Why it matters**: closes the loop with 1.8's claim that distributing requests is what makes horizontal scaling actually even out load — N replicas plus an autoscaler accomplish nothing if every request still lands on the same one or two Pods. Service/Ingress is what spreads traffic across whatever replica count currently exists.
- **Key caveat — readiness probes**: a Service only routes to Pods in the `Ready` condition. A newly started model-serving Pod that reports `Running` before its weights finish loading will otherwise receive live traffic it can't yet serve; a correctly configured readiness probe prevents this.

---

## The GPU-specific layer

- **What it is**: Kubernetes has no native concept of "GPU" the way it does CPU or memory. NVIDIA's GPU Operator docs: "configuring and managing nodes with these hardware resources requires configuration of multiple software components such as drivers, container runtimes or other libraries which are difficult and prone to errors."
- **NVIDIA device plugin** — advertises GPUs as an allocatable resource (`nvidia.com/gpu: 1`) so the scheduler places GPU-requesting Pods only on nodes that have GPUs.
- **GPU Operator** — automates the rest (GPU drivers, NVIDIA Container Toolkit, node labeling via GPU Feature Discovery, DCGM-based monitoring) as installable Kubernetes components rather than manual node setup.
- **Caveat**: GPU node autoscaling is materially slower than CPU — a fresh GPU node has to boot, install/validate the driver stack, and only then become schedulable.
- **NVIDIA NIM Operator** — the Kubernetes-native layer for deploying/scaling NIM (and NeMo) microservices:
  - **NIM cache CRD** — downloads model weights (NGC, Hugging Face, local storage) onto a PersistentVolumeClaim so replicas don't each re-download/re-optimize independently.
  - **NIM service CRD** — "manage[s] deployment, scaling, and ingress" — the Operator's own Deployment/Service/HPA equivalents, purpose-built for NIM.
  - **Multi-node deployment** via Kubernetes **LeaderWorkerSets** for models too large for one GPU/node — coordinates multiple Pods across nodes as one logical replica.
  - **Dynamic Resource Allocation (DRA)** — a technology preview; more flexible than the plain device plugin, letting workloads request GPUs by device class and share a single GPU via time-slicing.
  - NIM cache targets the cold-start problem directly: "intelligent caching with NIM cache" reduces "both initial inference time and autoscaling latency."

---

## Concrete reference pattern: Triton + TensorRT-LLM + Kubernetes

- Wires every piece together end to end: a **Deployment** runs Triton server Pods (ports **8000/HTTP, 8001/gRPC, 8002/metrics**) on GPU nodes; a **ClusterIP Service** exposes them; a **PodMonitor** tells Prometheus to scrape port 8002 on every matching Pod; a **PrometheusRule** computes a custom recording-rule metric — `triton:queue_compute:ratio = rate(nv_inference_queue_duration_us[1m]) / rate(nv_inference_compute_infer_duration_us[1m])` — the ratio of time a request waits in queue to time it spends actually being computed.
- An **HPA** (`autoscaling/v2`, metric type `Pods`) scales the Deployment when the average `queue:compute` ratio across replicas exceeds a target (their example: `1000m`, i.e., queue time exceeding compute time), between `minReplicas`/`maxReplicas`.
- **Live demo result**: scaling ten client replicas up drove GPU utilization and the queue:compute ratio up → HPA scaled the Triton Deployment from **1 Pod to 4** → the ratio came back down → reversed cleanly when client load dropped back to one replica.
- This is exactly what "GPU queue depth as a custom autoscaling metric" (1.8's forward reference) looks like in practice — not CPU%, but an inference-specific latency signal exposed by the serving engine itself.
- **Cold-start mitigation**: "the generated TensorRT engine and plan files are stored in the host node and remapped to all the Kubernetes Pods on the same node," so scaling up doesn't require regenerating optimized engine files for every new replica on that node.

---

## Trade-offs and pitfalls

- **Stateful pods can't scale freely** (1.8) — a component that must serialize requests or pin to one instance caps horizontal scaling. Agent state (history, plan, session) has to live outside the Pod for the Deployment/HPA/Service pattern to work at all.
- **GPU cost of over-provisioning replicas** — GPUs are the most expensive line item; a generous `maxReplicas` with no node-autoscaler ceiling can scale a bursty or looping workload into a very large bill before anyone notices. `minReplicas`/`maxReplicas` bounds exist to cap this (deeper cost/HA trade-offs = 4.5).
- **Cold-start / model-load latency on scale-out** — a newly scheduled Pod isn't ready the instant it starts; it must load model weights (and, without host-level engine caching, potentially rebuild TensorRT engines). HPA's scale decision is near-instant relative to how long the Pod takes to become `Ready`; NIM cache and host-node-mapped engine files shrink but don't eliminate that gap.
- **Autoscaling lag vs. bursty agentic load** — HPA's loop runs on a fixed interval (15s default) with a stabilization window smoothing scale-down over minutes; GPU node provisioning is slower still. Agentic fan-out (one request triggering many parallel tool/subagent calls) can spike load faster than this two-layer stack can react — the concrete reason 4.1's "per-agent scaling" framing matters.
- **Readiness/liveness probes are not optional** for model-serving Pods — without one gated on the model actually being loaded, the Service routes live traffic to a Pod that reports `Running` but can't yet serve.
- **Scaling the wrong tier** — an HPA on the model-serving Deployment does nothing for a bottleneck sitting downstream (vector store, tool-execution backend, rate-limited external API); each tier in a multi-agent deployment (4.1) needs its own autoscaler bound to its own metric.

---

## NVIDIA angle (read critically)

- Genuine engineering value: GPU Operator/device plugin turn manual driver-stack setup into a declarative, Kubernetes-native install; NIM Operator's cache/service CRDs plus multi-node LeaderWorkerSet support turn a genuinely hard multi-GPU, multi-node orchestration problem into a couple of custom resources; the queue:compute ratio is a well-chosen metric — it reflects request-level latency pressure on a GPU-bound, batch-serving workload in a way raw CPU utilization structurally cannot.
- **Read the specifics critically**: the Triton/TensorRT-LLM walkthrough is tutorial-scale (a GPT-2 model, 1–4 replicas, a single A10G GPU class) — not a validated production capacity plan; its bounds and target ratio (`1000m`) are illustrative defaults, not universal tuning values. NIM Operator's DRA support is explicitly a "technology preview," not generally available. NVIDIA's own multi-node NIM docs admit a real limitation: deploying without GPUDirect RDMA "may result in frequent restarts of LWS leader and worker pods due to model shard loading timeouts" — the smooth "just scale it across nodes" story has a fast-interconnect prerequisite. On load balancing itself, NVIDIA ships no opinionated product — its walkthrough points to third-party (Traefik, NGINX Plus) or cloud-provider load balancers, i.e., the L7 layer is "bring your own."

---

## How to think about it for the exam

| Concept | Key point |
|---|---|
| Container image | Immutable, dependency-complete, portable unit; fixes "works on my machine"; identical replicas are what make horizontal scaling safe (substrate for 1.8) |
| Pod | Smallest deployable unit; one or more co-located, co-scheduled containers; ephemeral — rarely created directly |
| Deployment | Declarative desired state + controller; manages replica count and rolling updates (`maxUnavailable`/`maxSurge`, default 25%/25%); supports rollback via revision history |
| Service | Stable virtual IP + selector-based routing to a changing set of Pod IPs; this *is* Kubernetes' L4 load balancing (kube-proxy); types: ClusterIP, NodePort, LoadBalancer |
| Ingress | L7 HTTP(S) routing (host/path rules, TLS termination) from outside the cluster to Services; requires an Ingress controller to do anything |
| HPA | Control loop (15s default) scaling replica count on a metric; resource metrics (CPU/mem) or custom metrics (e.g., GPU queue:compute ratio); stabilization window/tolerance prevent flapping |
| Node/cluster autoscaler | Provisions nodes when Pods are unschedulable, consolidates underused nodes; Cluster Autoscaler (pre-configured node groups) vs. Karpenter (direct provisioning); composes with HPA (Pods scale first, nodes follow) |
| L4 vs L7 load balancing | L4 = transport-level, network info only (Service/kube-proxy); L7 = application-level, content-aware (Ingress, third-party/cloud LB) |
| GPU as a scheduled resource | Not native to Kubernetes; NVIDIA device plugin advertises GPUs as allocatable; GPU Operator automates driver/toolkit/monitoring install |
| NIM Operator | Kubernetes-native NIM deployment/scaling: NIM cache CRD (weights on PVC), NIM service CRD (deploy/scale/ingress), multi-node via LeaderWorkerSets, DRA (preview) |
| Concrete GPU autoscaling pattern | Triton pods → Prometheus scrapes metrics port → custom `queue:compute` ratio → HPA scales Deployment on that custom metric |

---

## Quick self-check

1. Why does container image immutability matter specifically for horizontal scaling, not just for portability?
   **A:** Replication only works cleanly if every replica is guaranteed to run the same code — an immutable image means N replicas of it are, by construction, identical and interchangeable. That's the literal substrate under 1.8's claim that a stateless component can be freely replicated.
2. What is the difference in role between a Kubernetes Service and a Kubernetes Ingress, and which one is doing Layer 4 vs. Layer 7 load balancing?
   **A:** A Service gives Pods a stable virtual IP and routes to a changing set of matching Pod IPs based purely on network information — Kubernetes' **L4** load balancing via `kube-proxy`. Ingress sits above it and routes HTTP(S) traffic by hostname/path/headers — **L7** — and needs an Ingress controller to actually function.
3. Walk through what happens end to end when HPA decides more replicas are needed but the cluster has no spare node capacity — which two Kubernetes mechanisms are involved, and in what order do they act?
   **A:** HPA raises `.spec.replicas` on the Deployment, producing Pods that can't be scheduled on existing nodes. The node/cluster autoscaler (Cluster Autoscaler or Karpenter) notices those unschedulable Pods and provisions new nodes to fit them; once ready, the Pods schedule. Consolidation reverses this as load drops.
4. In NVIDIA's Triton/TensorRT-LLM Kubernetes walkthrough, what custom metric drives the HPA, how is it computed, and why is it a better autoscaling signal for this workload than CPU utilization?
   **A:** `triton:queue_compute:ratio = rate(nv_inference_queue_duration_us[1m]) / rate(nv_inference_compute_infer_duration_us[1m])` — the ratio of queue-wait time to compute time, scraped from Triton's metrics port by Prometheus. It's better than CPU% because it directly reflects request-level latency pressure on a GPU-bound, batch-serving workload, which raw CPU utilization can't capture.
5. Name two reasons a newly scaled-out GPU-serving Pod might not be able to handle traffic the instant it's scheduled, and what Kubernetes mechanism exists to keep the load balancer from routing to it prematurely.
   **A:** It may still be loading model weights, or — without host-level engine caching — rebuilding TensorRT engine/plan files; both take real time after the Pod reports `Running`. A **readiness probe** gated on the model actually being loaded is the mechanism: a Service only routes to Pods in the `Ready` condition.

---

## Sources used

- [Pods — Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Deployments — Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service — Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress — Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Horizontal Pod Autoscaling — Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Node Autoscaling — Kubernetes documentation](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [What is a Container? — Docker](https://www.docker.com/resources/what-container/)
- [Scaling LLMs with NVIDIA Triton and NVIDIA TensorRT-LLM Using Kubernetes — NVIDIA Technical Blog](https://developer.nvidia.com/blog/scaling-llms-with-nvidia-triton-and-nvidia-tensorrt-llm-using-kubernetes/)
- [What Is Kubernetes and Why Does It Matter? — NVIDIA Glossary](https://www.nvidia.com/en-us/glossary/kubernetes/)
- [About the NVIDIA GPU Operator — NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
- [Deploy Scalable AI Inference with NVIDIA NIM Operator 3.0.0 — NVIDIA Technical Blog](https://developer.nvidia.com/blog/deploy-scalable-ai-inference-with-nvidia-nim-operator-3-0-0/)
