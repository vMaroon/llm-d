# KubeCon Tutorial — Claude Code Guide

Read `tutorial/AGENTS.md` for full context, then walk the user through
`tutorial/kubecon-kv-cache-tutorial.md` step by step.

Run each command, verify it succeeded, explain what happened. If something
fails, debug it. The user may have no Kubernetes background.

Key things Claude Code should know:
- The simulator needs `args: ["--model", "random"]` in standalone Deployments
- Use `sed` not `envsubst` for benchmark template substitution (cross-platform)
- Always `kubectl config use-context kind-llm-d-tutorial` before running kubectl commands
- Check `docker images ghcr.io/llm-d/llm-d-benchmark:v0.5.0 -q` to decide between Option A (full benchmark) and Option B (traffic script)
