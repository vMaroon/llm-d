# KV-Cache Wins You Can Feel: Building AI-Aware LLM Routing on Kubernetes

> **KubeCon Hands-On Tutorial** | 75 minutes
>
> By the end of this tutorial you will have a working llm-d environment with
> intelligent inference scheduling, observability via Prometheus and Grafana,
> and a clear before/after comparison of cache-blind vs. cache-aware LLM routing.

---

## Agenda

| Time | Section | What You'll Do |
|------|---------|---------------|
| 0:00 | [Part 0: Presentation](#part-0-presentation-15-min) | Context — why KV-cache locality matters |
| 0:15 | [Part 1: Verify Setup](#part-1-verify-setup-5-min) | Confirm pre-work, check cluster and monitoring |
| 0:20 | [Part 2: Baseline — Vanilla Kubernetes](#part-2-baseline--inference-with-vanilla-kubernetes-15-min) | Deploy vLLM behind a plain Service, send traffic, observe |
| 0:35 | [Part 3: Deploy llm-d](#part-3-deploy-llm-d-20-min) | Install Gateway API, deploy inference scheduler, route traffic |
| 0:55 | [Part 4: See the Difference](#part-4-see-the-difference-15-min) | Compare dashboards, explore metrics, PromQL |
| 1:10 | [Part 5: Wrap-Up](#part-5-wrap-up--next-steps-5-min) | Cleanup, resources, next steps |

---

## Pre-Work — Complete Before the Tutorial

> **This section is mandatory.** The hands-on portion has 75 minutes and zero slack
> for downloading multi-hundred-MB container images over conference WiFi. Complete
> everything below before the session starts.

### Install tools

| Tool | Version | Install |
|------|---------|---------|
| `docker` | 20.10+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| `kind` | v0.20+ | `brew install kind` / [kind.sigs.k8s.io](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| `kubectl` | v1.28+ | [kubernetes.io/docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/) |
| `helm` | v3.12+ | [helm.sh/docs](https://helm.sh/docs/intro/install/) |
| `helmfile` | v1.1+ | [github.com/helmfile](https://github.com/helmfile/helmfile?tab=readme-ov-file#installation) |
| `yq` | v4+ | `brew install yq` / [github.com/mikefarah/yq](https://github.com/mikefarah/yq) |
| `jq` | any | `brew install jq` / [jqlang.github.io](https://jqlang.github.io/jq/) |
| `curl` | any | Pre-installed on most systems |
| `git` | v2.30+ | Pre-installed on most systems |

> **Shortcut**: The llm-d repo provides an install script that covers kubectl, helm, helmfile, and yq:
>
> ```bash
> git clone https://github.com/llm-d/llm-d.git && cd llm-d
> ./guides/prereq/client-setup/install-deps.sh
> ```

### Create the kind cluster

```bash
cat <<'EOF' > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --name llm-d-tutorial --config kind-config.yaml
```

This pulls the `kindest/node` image (~900 MB) and creates 4 nodes. Takes 2-5 minutes
on a reasonable connection.

### Clone the repo

```bash
git clone https://github.com/llm-d/llm-d.git && cd llm-d
```

> **Tip**: To pin to the latest release:
> ```bash
> git checkout $(curl -s https://api.github.com/repos/llm-d/llm-d/releases/latest | jq -r '.tag_name')
> ```

### Pre-pull container images

This is the step that saves you during the tutorial. Every image below will be
needed; pulling them now avoids 10+ minutes of dead time during the session.

```bash
# Simulator image (8 replicas will use this)
docker pull ghcr.io/llm-d/llm-d-inference-sim:v0.7.1

# Routing sidecar
docker pull ghcr.io/llm-d/llm-d-routing-sidecar:v0.6.0

# Inference scheduler (EPP)
docker pull ghcr.io/llm-d/llm-d-inference-scheduler:v0.6.0

# Benchmark harness (~3 GB — start this pull early)
docker pull ghcr.io/llm-d/llm-d-benchmark:v0.5.0

# Load all into kind (so the cluster nodes have them locally)
kind load docker-image ghcr.io/llm-d/llm-d-inference-sim:v0.7.1 --name llm-d-tutorial
kind load docker-image ghcr.io/llm-d/llm-d-routing-sidecar:v0.6.0 --name llm-d-tutorial
kind load docker-image ghcr.io/llm-d/llm-d-inference-scheduler:v0.6.0 --name llm-d-tutorial
kind load docker-image ghcr.io/llm-d/llm-d-benchmark:v0.5.0 --name llm-d-tutorial
```

### Download the benchmark script

```bash
curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh
chmod u+x run_only.sh
```

### Install the monitoring stack

The kube-prometheus-stack Helm chart pulls ~6 container images (Prometheus, Grafana,
node-exporter, kube-state-metrics, alertmanager, config-reloader). Installing it now
means Part 2 of the tutorial just works.

```bash
./docs/monitoring/scripts/install-prometheus-grafana.sh
./docs/monitoring/scripts/load-llm-d-dashboards.sh llm-d-monitoring
```

Wait until all monitoring pods are running:

```bash
kubectl get pods -n llm-d-monitoring
# All pods should show Running/Ready
```

### Create the tutorial namespace, HF token, and benchmark PVC

The benchmark harness needs a HuggingFace token (to download the Qwen3-32B tokenizer
for generating synthetic prompts) and a PVC to store results.

```bash
export NAMESPACE=llm-d-tutorial
kubectl create namespace ${NAMESPACE}

# HuggingFace token — needed for tokenizer download
export HF_TOKEN=<your-huggingface-token>
kubectl create secret generic llm-d-hf-token \
    --from-literal="HF_TOKEN=${HF_TOKEN}" \
    --namespace "${NAMESPACE}"

# Benchmark results PVC (1Gi is plenty for tutorial)
cat <<'EOF' | kubectl apply -n ${NAMESPACE} -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tutorial-bench-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

### Verify everything

```bash
# Cluster is up
kubectl get nodes
# Should show 4 nodes, all Ready

# Monitoring is running
kubectl get pods -n llm-d-monitoring
# All pods Running

# Images are loaded
docker images | grep llm-d
# Should show inference-sim, routing-sidecar, inference-scheduler, llm-d-benchmark

# Benchmark script is available
./run_only.sh --help 2>&1 | head -1

# HF token and PVC exist
kubectl get secret llm-d-hf-token -n ${NAMESPACE}
kubectl get pvc tutorial-bench-pvc -n ${NAMESPACE}
```

> **Path B (real GPUs)**: Skip the kind and image pre-pull steps. Ensure your cluster
> meets the [infrastructure prerequisites](./guides/prereq/infrastructure/README.md).
> You still need the HF token, PVC, benchmark image, and `run_only.sh` above.
> Change the model name in `tutorial/benchmark-template.yaml` from `random` to your
> deployed model (e.g., `Qwen/Qwen3-32B`).

---

## Part 0: Presentation (15 min)

*Instructor-led. Slides + live discussion.*

### Key Concepts to Cover

**The problem**: Kubernetes default load balancing (round-robin / random) is cache-blind.
LLM inference workloads reuse KV-cache entries heavily — multi-turn conversations,
shared system prompts, retrieval-augmented generation all produce overlapping prefixes.
Scattering these requests across pods destroys locality, forces recomputation, wastes GPU
cycles, and inflates time-to-first-token (TTFT).

**Why it matters**: Hitting the KV-cache can make prefill 10x cheaper and up to 50x faster.
A cache miss means the GPU must recompute every token in the prompt from scratch.
At scale, this is the difference between viable and prohibitively expensive.

**What llm-d does**: llm-d is a Kubernetes-native distributed inference serving stack.
Its core idea: replace the default Service load balancing with an intelligent inference
scheduler that understands KV-cache state, prefix locality, and per-instance load.

Architecture overview:
- **vLLM** as the model server (or our simulator for this tutorial)
- **Kubernetes Gateway API** as the traffic management layer
- **Inference Scheduler (EPP)** — an endpoint picker that scores instances based on
  prefix-cache affinity, queue depth, and KV-cache utilization
- **Envoy proxy** handles the actual traffic, delegating routing decisions to the EPP

**What we'll build today**:
1. A baseline: vLLM behind a plain k8s Service (cache-blind round-robin)
2. Observability: Prometheus + Grafana to see what's happening
3. llm-d: replace the Service with intelligent routing
4. Compare: same traffic, dramatically different behavior

---

## Part 1: Verify Setup (5 min)

Confirm that the pre-work is complete. If anyone hasn't finished, pair up with
a neighbor or follow along on screen — there is no buffer time to install tools
or pull images during the session.

```bash
# 1. Cluster is up and has 4 nodes
kubectl get nodes
# Expected: 1 control-plane + 3 workers, all Ready

# 2. You're in the llm-d repo
ls guides/simulated-accelerators/
# Expected: helmfile.yaml.gotmpl, httproute.yaml, ms-sim/, gaie-sim/

# 3. Monitoring stack is running
kubectl get pods -n llm-d-monitoring --no-headers | wc -l
# Expected: 5+ pods

# 4. Key images are loaded into kind
docker exec llm-d-tutorial-worker crictl images | grep llm-d
# Expected: inference-sim, routing-sidecar, inference-scheduler
```

> If any of these fail, refer back to the [Pre-Work](#pre-work--complete-before-the-tutorial) section.

---

## Part 2: Baseline — Inference with Vanilla Kubernetes (15 min)

Goal: deploy model server replicas behind a plain Kubernetes Service (round-robin)
and observe how requests are distributed without any cache awareness.

### 2.1 Set environment variables

```bash
export NAMESPACE=llm-d-tutorial
export BENCHMARK_PVC=tutorial-bench-pvc
```

The namespace, HF token, and PVC were created during pre-work.

### 2.2 Deploy the simulator pods

We deploy **8 simulator replicas** behind a standard Kubernetes Service — matching
the 8-pod Qwen3-32B TP=2 setup from the inference-scheduling guide.
Images were pre-pulled, so pods should start in seconds.

```bash
cat <<'EOF' | kubectl apply -n ${NAMESPACE} -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-baseline
  labels:
    app: vllm-baseline
spec:
  replicas: 8
  selector:
    matchLabels:
      app: vllm-baseline
  template:
    metadata:
      labels:
        app: vllm-baseline
    spec:
      containers:
        - name: vllm-sim
          image: ghcr.io/llm-d/llm-d-inference-sim:v0.7.1
          ports:
            - containerPort: 8000
              name: http
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-baseline
spec:
  selector:
    app: vllm-baseline
  ports:
    - name: http
      port: 80
      targetPort: 8000
  type: ClusterIP
EOF
```

Wait for all pods to be ready:

```bash
kubectl rollout status deployment/vllm-baseline -n ${NAMESPACE} --timeout=120s
kubectl get pods -n ${NAMESPACE}
# Should show 8 Running pods
```

### 2.3 Open the observability dashboards

Prometheus and Grafana were installed during pre-work. Start port-forwards:

```bash
# Grafana
kubectl port-forward -n llm-d-monitoring svc/llmd-grafana 3000:80 &

# Prometheus
kubectl port-forward -n llm-d-monitoring svc/llmd-kube-prometheus-stack-prometheus 9090:9090 &
```

- **Grafana**: [http://localhost:3000](http://localhost:3000) (login: `admin` / `admin`)
- **Prometheus**: [http://localhost:9090](http://localhost:9090)

### 2.4 Run the benchmark against the baseline Service

We use the same `inference-perf` benchmark harness from the inference-scheduling guide,
but with a single stage. The workload generates shared-prefix synthetic data: 150
prefix groups, 6000-token system prompts, 1200-token questions, 1000-token outputs.

This is the same workload profile used to produce the llm-d vs k8s comparison numbers
in the guide — just 1 stage instead of the full rate sweep.

```bash
# Point the benchmark at the plain k8s Service (round-robin)
export GATEWAY_SVC=vllm-baseline
sed -e "s|\${NAMESPACE}|${NAMESPACE}|g" \
    -e "s|\${GATEWAY_SVC}|${GATEWAY_SVC}|g" \
    -e "s|\${BENCHMARK_PVC}|${BENCHMARK_PVC}|g" \
    tutorial/benchmark-template.yaml > config-baseline.yaml
./run_only.sh -c config-baseline.yaml
```

The benchmark creates a launcher pod in-cluster, generates synthetic prompts using
the Qwen3-32B tokenizer, and sends them to the baseline Service at 35 req/s for
120 seconds (~4200 requests).

> While the benchmark runs (~2.5 min), watch the Grafana **llm-d vLLM Overview**
> dashboard — you'll see requests distributed roughly evenly across all 8 pods.

### 2.5 Observe the baseline behavior

**In Prometheus**, check per-pod request distribution:

```promql
sum by (pod) (increase(vllm:request_success_total[3m]))
```

**What you should see**: Requests are spread roughly evenly across all 8 pods.
Each pod handles ~525 of the 4200 requests, regardless of which prefix group
the request belongs to. On real GPUs, this means every pod independently computes
KV-cache for the same 6000-token system prompts — **8x the GPU work for identical
prefixes**.

> **Key takeaway**: Kubernetes round-robin treats all requests as equal.
> It has no concept of "these requests share a prefix and should go to the same backend."

---

## Part 3: Deploy llm-d (20 min)

Now we replace the plain Service with the llm-d inference scheduling stack.
This adds the Gateway API, the inference scheduler (EPP), and prefix-cache-aware routing.

### 3.1 Install Gateway API CRDs

The inference scheduler integrates with the Kubernetes Gateway API.
First, install the required CRDs:

```bash
./guides/prereq/gateway-provider/install-gateway-provider-dependencies.sh
```

This installs:
- Gateway API v1.4.0 CRDs
- Gateway API Inference Extension v1.3.1 CRDs (including `InferencePool`)

Verify:

```bash
kubectl api-resources --api-group=inference.networking.k8s.io
```

You should see `InferencePool` listed.

### 3.2 Install the gateway provider (kgateway)

We use **kgateway** — a lightweight Envoy-based Gateway implementation
that works well on kind:

```bash
cd guides/prereq/gateway-provider
helmfile apply -f kgateway.helmfile.yaml
cd ../../..
```

### 3.3 Clean up the baseline deployment

Remove the vanilla Service and Deployment — we'll redeploy through llm-d's
Helm charts which manage everything consistently:

```bash
kubectl delete deployment vllm-baseline -n ${NAMESPACE}
kubectl delete service vllm-baseline -n ${NAMESPACE}
```

### 3.4 Deploy the full llm-d simulator stack

The simulated-accelerators guide deploys the complete stack: infrastructure (Gateway),
inference scheduler (EPP), and model service (simulator pods).

```bash
cd guides/simulated-accelerators
helmfile apply -e kgateway -n ${NAMESPACE}
```

This creates three Helm releases:

| Release | Chart | What it deploys |
|---------|-------|----------------|
| `infra-sim` | `llm-d-infra` | Gateway + infrastructure |
| `gaie-sim` | `inferencepool` | InferencePool + EPP (inference scheduler) |
| `ms-sim` | `llm-d-modelservice` | Simulator pods (3 decode + 1 prefill by default) |

Now scale the decode deployment to **8 replicas** to match the guide's 8-pod setup:

```bash
kubectl scale deployment ms-sim-llm-d-modelservice-decode -n ${NAMESPACE} --replicas=8
```

Wait for all pods to be ready:

```bash
kubectl get pods -n ${NAMESPACE} -w
```

You should see:
- 1 gateway pod (`infra-sim-inference-gateway-kgateway-*`)
- 1 EPP pod (`gaie-sim-epp-*`)
- **8 decode simulator pods** (`ms-sim-llm-d-modelservice-decode-*`)
- 1 prefill simulator pod (`ms-sim-llm-d-modelservice-prefill-*`)

### 3.5 Install the HTTPRoute

The HTTPRoute tells the Gateway how to route traffic to the InferencePool:

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

### 3.6 Verify the deployment

```bash
# Check all resources
kubectl get all -n ${NAMESPACE}

# Check the Gateway is programmed
kubectl get gateway -n ${NAMESPACE}

# Check the InferencePool is ready
kubectl get inferencepool -n ${NAMESPACE}
```

### 3.7 Send traffic through llm-d

Quick smoke test to verify the stack is working:

```bash
# Discover the gateway service name
export GATEWAY_SVC=$(kubectl get svc -n "${NAMESPACE}" -o yaml | \
  yq '.items[] | select(.metadata.name | test(".*-inference-gateway(-.*)?$")).metadata.name' | head -n1)

# Smoke test via port-forward
kubectl port-forward -n ${NAMESPACE} svc/${GATEWAY_SVC} 8000:80 &
curl -s http://localhost:8000/v1/models | jq
kill %% 2>/dev/null  # stop port-forward; benchmark connects in-cluster
```

Now run the **exact same benchmark** — same workload, same rate, but routed through
the llm-d inference scheduler instead of round-robin:

```bash
cd ../..
sed -e "s|\${NAMESPACE}|${NAMESPACE}|g" \
    -e "s|\${GATEWAY_SVC}|${GATEWAY_SVC}|g" \
    -e "s|\${BENCHMARK_PVC}|${BENCHMARK_PVC}|g" \
    tutorial/benchmark-template.yaml > config-llmd.yaml
./run_only.sh -c config-llmd.yaml
```

> While the benchmark runs (~2.5 min), watch the Grafana dashboard — notice how
> the request distribution is no longer uniform across pods.

---

## Part 4: See the Difference (15 min)

### 4.1 Check request distribution

```bash
for pod in $(kubectl get pods -n ${NAMESPACE} -l llm-d.ai/role=decode -o name); do
  echo "=== ${pod} ==="
  kubectl logs ${pod} -n ${NAMESPACE} -c vllm --tail=100 | grep -c "completions" || echo "0 requests"
done
```

**What you should see now**: Requests are no longer evenly distributed. The inference
scheduler routes requests that share a prefix to the **same instance** — maximizing
KV-cache reuse. A few decode pods handle the majority of traffic while others are
relatively idle.

Compare this to Part 2, where each of the 8 pods handled ~525 of 4200 requests.

### 4.2 Compare in Grafana

Make sure port-forwards are still active:

```bash
kubectl port-forward -n llm-d-monitoring svc/llmd-grafana 3000:80 &
kubectl port-forward -n llm-d-monitoring svc/llmd-kube-prometheus-stack-prometheus 9090:9090 &
```

Open [http://localhost:3000](http://localhost:3000) and explore:

| Dashboard | What to Look For |
|-----------|-----------------|
| **llm-d vLLM Overview** | Request distribution is now skewed — concentrated on fewer pods |
| **llm-d Performance Dashboard** | KV cache utilization differs across pods (hot vs. cold) |
| **llm-d Failure & Saturation** | Error rates, queue depth, saturation indicators |
| **llm-d Diagnostic Drill-Down** | Per-instance latency breakdown |

### 4.3 Key PromQL queries

Open [http://localhost:9090](http://localhost:9090) and run these queries:

**Per-pod request count (the main before/after signal):**

```promql
sum by (pod) (increase(vllm:request_success_total[3m]))
```

In the baseline this was ~equal across 8 pods. With llm-d, you should see
concentration — pods that serve shared-prefix groups get more traffic.

**Request distribution via EPP:**

```promql
sum by (pod) (rate(inference_extension_picked_endpoint_total[3m]))
```

**EPP scheduling latency (P99):**

```promql
histogram_quantile(0.99, sum by (le) (rate(inference_extension_scheduler_e2e_duration_seconds_bucket[3m])))
```

**KV cache utilization per pod:**

```promql
avg by (pod) (vllm:kv_cache_usage_perc)
```

### 4.4 (Optional) Inspect benchmark results

The benchmark harness stored detailed per-request metrics on the PVC. You can
inspect them from the launcher pod:

```bash
HARNESS_POD=$(kubectl get pods -n ${NAMESPACE} -l app --show-labels | \
  awk -v p='lmdbench-.*-launcher' '$0~p {print $1; exit}')
kubectl exec ${HARNESS_POD} -n ${NAMESPACE} -- ls /requests
```

To copy results locally:

```bash
kubectl cp ${NAMESPACE}/${HARNESS_POD}:/requests ./benchmark-results
```

### 4.5 The before/after story

| Metric | Vanilla k8s Service | llm-d |
|--------|-------------------|-------|
| Request distribution | Even (round-robin) | Prefix-aware (concentrated) |
| Prefix cache hits | Low — each pod computes independently | High — shared prefixes routed to same pod |
| TTFT | High — full prefill on cache miss | Low — cached prefill on cache hit |
| GPU utilization | Wasted on redundant computation | Efficient — cache reuse reduces prefill |

In production benchmarks on 8× Qwen3-32B (TP=2) with this same shared-prefix
workload, llm-d shows:

| Metric | k8s Service | llm-d | Change |
|--------|-------------|-------|--------|
| Requests/sec | 5.1 | 7.1 | **+38.9%** |
| Output tokens/sec | 4,787 | 6,644 | **+38.8%** |
| Mean TTFT | 72.9s | 2.1s | **-97.1%** |
| Mean request latency | 123.5s | 85.0s | **-31.2%** |

*(Source: [inference-scheduling guide](./guides/inference-scheduling/README.md#comparing-llm-d-scheduling-to-a-simple-kubernetes-service)
— same workload, same hardware, llm-d vs plain k8s Service)*

---

## Part 5: Wrap-Up & Next Steps (5 min)

### What you built

You now have a working llm-d environment with:
- An inference simulator (or real vLLM) running on Kubernetes
- The llm-d inference scheduler providing prefix-cache-aware routing
- Prometheus + Grafana observability with llm-d dashboards
- A clear understanding of why cache-blind load balancing is wasteful

### Cleanup

```bash
# Remove the llm-d stack
cd guides/simulated-accelerators
helmfile destroy -e kgateway -n ${NAMESPACE}
cd ../..

# Remove the gateway provider
cd guides/prereq/gateway-provider
helmfile destroy -f kgateway.helmfile.yaml
./install-gateway-provider-dependencies.sh delete
cd ../../..

# Remove monitoring
./docs/monitoring/scripts/install-prometheus-grafana.sh -u

# Delete the namespace
kubectl delete namespace ${NAMESPACE}

# Delete the kind cluster
kind delete cluster --name llm-d-tutorial
```

### Where to go next

| Path | Guide | What it adds |
|------|-------|-------------|
| **Production deployment** | [Inference Scheduling](./guides/inference-scheduling/README.md) | Real GPU deployment with vLLM + benchmarking |
| **Lower TTFT** | [Prefill/Decode Disaggregation](./guides/pd-disaggregation/README.md) | Split prefill and decode onto separate servers |
| **Large MoE models** | [Wide Expert-Parallelism](./guides/wide-ep-lws/README.md) | Deploy DeepSeek-R1 class models |
| **Better cache reuse** | [Tiered Prefix Cache](./guides/tiered-prefix-cache/README.md) | Offload KV-cache to CPU/SSD/storage |
| **Precise routing** | [Precise Prefix Cache Aware](./guides/precise-prefix-cache-aware/README.md) | Introspect actual vLLM cache state |
| **Autoscaling** | [Workload Autoscaling](./guides/workload-autoscaling/README.md) | Dynamic scaling based on traffic |

### Resources

- **llm-d repository**: [github.com/llm-d/llm-d](https://github.com/llm-d/llm-d)
- **Documentation**: [llm-d.ai](https://www.llm-d.ai)
- **Slack**: [llm-d.ai/slack](https://llm-d.ai/slack)
- **Inference Gateway (upstream)**: [gateway-api-inference-extension.sigs.k8s.io](https://gateway-api-inference-extension.sigs.k8s.io/)

---

## Appendix A: Troubleshooting

### Pods stuck in Pending

```bash
kubectl describe pod <pod-name> -n ${NAMESPACE}
```

Common causes on kind: insufficient resources. Reduce simulator replicas or add
more kind worker nodes.

### Gateway not programmed

```bash
kubectl describe gateway -n ${NAMESPACE}
```

Ensure the gateway provider (kgateway) is installed and the CRDs are present:

```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
```

### No metrics in Grafana

1. Verify PodMonitors exist: `kubectl get podmonitors -n ${NAMESPACE}`
2. Check Prometheus targets: visit `http://localhost:9090/targets`
3. Ensure the monitoring namespace label is set:
   ```bash
   kubectl label namespace ${NAMESPACE} monitoring-ns=llm-d-monitoring
   ```

### Port-forward drops

Port-forwards are fragile. If they stop working, restart them:

```bash
kubectl port-forward -n ${NAMESPACE} svc/${GATEWAY_SVC} 8000:80 &
kubectl port-forward -n llm-d-monitoring svc/llmd-grafana 3000:80 &
kubectl port-forward -n llm-d-monitoring svc/llmd-kube-prometheus-stack-prometheus 9090:9090 &
```

### Simulator returns errors

Check simulator logs:

```bash
kubectl logs -n ${NAMESPACE} -l app.kubernetes.io/component=decode -c vllm --tail=20
```

The simulator mimics the vLLM API. If you get 404s, verify the model name
matches what the simulator reports at `/v1/models`.

---

## Appendix B: Architecture Reference

```
                                    ┌─────────────────────────┐
                                    │     Grafana + Prom      │
                                    │    (observability)       │
                                    └────────────┬────────────┘
                                                 │ scrapes metrics
                                                 │
  Client                                         │
    │                                            │
    │  curl /v1/completions                      │
    ▼                                            │
┌─────────┐      ┌───────────┐      ┌───────────▼──────────┐
│ Gateway  │─────▶│    EPP    │─────▶│   vLLM Simulator     │
│ (Envoy)  │      │ (Inference│      │   Pod 1 (decode)     │
│          │      │ Scheduler)│      ├──────────────────────┤
│          │      │           │      │   Pod 2 (decode)     │
│          │      │  Scores:  │      ├──────────────────────┤
│          │      │  - prefix │      │   Pod 3 (decode)     │
│          │      │    match  │      ├──────────────────────┤
│          │      │  - load   │      │   Pod 4 (prefill)    │
│          │      │  - kv $   │      └──────────────────────┘
└─────────┘      └───────────┘

Without llm-d: Client → k8s Service → random pod (cache-blind)
With    llm-d: Client → Gateway → EPP picks best pod → cache-aware
```
