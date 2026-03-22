# KubeCon Tutorial — AI Assistant Guide

You are helping a user work through the KubeCon hands-on tutorial:
**"KV-Cache Wins You Can Feel: Building AI-Aware LLM Routing on Kubernetes"**

The full tutorial is in `tutorial/kubecon-kv-cache-tutorial.md`. Read it first.

## Your role

Walk the user through the tutorial step by step. Run each command for them,
verify it succeeded, and explain what happened before moving to the next step.
If something fails, debug it — don't just show the error.

## Key context

- The tutorial deploys llm-d (a Kubernetes-native inference serving stack) on a local kind cluster using a vLLM simulator (no GPUs needed).
- There are two phases: (1) baseline with a plain k8s Service (round-robin), (2) llm-d with prefix-cache-aware routing.
- Model name is `random` (the simulator's model). The baseline Deployment needs `args: ["--model", "random"]`.
- There is NO pre-work section. Everything — cluster creation, image pulls, monitoring install — happens during Part 0 while the instructor presents.

## Traffic generation — two paths

- **Option A (full benchmark)**: Uses inference-perf at 35 req/s for 120s. Requires the ~3GB benchmark image, an HF token (for the Qwen3-32B tokenizer), and a PVC. Only use if the user has the benchmark image pulled.
- **Option B (lightweight script)**: Uses `tutorial/generate-shared-prefix-traffic.sh` with curl. No extra images needed. Use this if WiFi was bad or the benchmark image didn't pull.

Check which path is available:
```bash
docker images ghcr.io/llm-d/llm-d-benchmark:v0.5.0 -q | grep -q . && echo "Option A available" || echo "Use Option B"
```

## Workflow

1. **Part 0**: Create kind cluster, pull 3 core images (~600MB), load into kind, install monitoring, create namespace. All in parallel.
2. **Part 1**: Deploy 8 simulator pods + Service. Open Grafana/Prometheus. Run traffic (Option A or B). Observe round-robin distribution.
3. **Part 2**: Install Gateway CRDs + kgateway. Delete baseline. Deploy llm-d via helmfile (`-e kgateway`), scale decode to 8. Run same traffic. Compare in dashboards — requests are now prefix-aware.
4. **Part 3**: Recap, show production numbers, cleanup.

## Important details

- Namespace: `llm-d-tutorial`
- The helmfile uses `-e kgateway` environment for kind deployments.
- After `helmfile apply`, scale decode to 8 replicas: `kubectl scale deployment ms-sim-llm-d-modelservice-decode -n llm-d-tutorial --replicas=8`
- The benchmark template uses `sed` for variable substitution (not `envsubst`).
- For the full benchmark: NAMESPACE, GATEWAY_SVC, and BENCHMARK_PVC must be exported before running `sed`.

## Common issues

- **Simulator crashes with "model parameter is empty"**: The standalone baseline Deployment needs `args: ["--model", "random"]`.
- **ImagePullBackOff**: Images weren't loaded into kind. Run `kind load docker-image <image> --name llm-d-tutorial`.
- **Gateway not programmed**: CRDs or kgateway not installed. Check `kubectl api-resources --api-group=gateway.networking.k8s.io`.
- **No metrics in Grafana**: Label the namespace: `kubectl label namespace llm-d-tutorial monitoring-ns=llm-d-monitoring`.
- **Port-forward drops**: Just restart it. They're fragile.
- **Wrong kubectl context**: Run `kubectl config use-context kind-llm-d-tutorial`.
