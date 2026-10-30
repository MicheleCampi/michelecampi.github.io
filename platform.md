---
layout: page
title: Inference platform
permalink: /platform/
---

**What it is.** An LLM inference platform on Kubernetes, built from public
repositories: a Rust operator that treats a model's cold start as a
lifecycle state, GKE infrastructure in Terraform delivered through GitOps
with Argo CD and Helm, and the load, SLO and cost tooling that judges both.
Designed, built and verified under simulated production load; the evidence
is in CI and in archived GPU runs.

**What it is not.** It has not served production traffic. The GPU scale
measured is one NVIDIA L4 running Qwen2.5-7B-Instruct in bf16 on vLLM
v0.11.0. Fleets, faults and autoscaling are exercised on multi-node kind,
against a simulator calibrated to that GPU, which states what it does not
reproduce.

## Architecture

    Terraform (GCS backend)                      gke-llm-inference-platform
      VPC + subnet, private Google access
      GKE regional cluster, Workload Identity, least-privilege node SA
        system-pool  e2-standard-2: Argo CD, operator, Alloy
        gpu-pool     g2-standard-4 + 1 L4, 0..1 nodes, flex-start, 4 h cap

    Argo CD (installed once with Helm, then managing itself; sync waves)
      wave -1  argo-cd, external-secrets (GCP Secret Manager, no keys)
      wave  0  monitoring, vllm-coldstart-operator (Helm chart, v0.6.0)
      wave  1  alloy  ->  Grafana Cloud: metrics, SLO burn-rate alerts

    vllm-coldstart-operator (Rust, kube-rs)       vllm-coldstart-operator
      VllmService   one model; cold start is a phase, not a timeout
      FleetService  N replicas across nodes; preemption and drain
                    move replicas make-before-break, without cascade
      KEDA (on kind) 1..N on load; 0 -> 1 through the HTTP add-on
        -> vLLM pods on the GPU pool

## Measured

- **Objectives per GPU class, on one L4:** time to first token at most
  1 s, or 2 s for prompts over 2048 tokens; time per output token at most
  100 ms; 99.5% of requests served
  ([ADR-0020](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0020-time-per-output-token-per-gpu-class.md),
  [ADR-0022](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0022-time-to-first-token-of-long-prompts-per-gpu-class.md)).
- **Capacity of one L4** on a load profile derived from a production
  trace: 0.229 to 0.345 requests a second within those objectives
  (ADR-0022).
- **Cost:** 4.38 to 8.07 EUR per million generated tokens within the
  objectives, at a declared 0.793 EUR/h; 1.16 EUR in overload, where the
  objectives no longer hold
  ([ADR-0025](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0025-the-cost-of-a-run-per-million-generated-tokens.md)).
- **Cold start on GKE:** 20 min 40 s from the Argo CD root application to
  a serving model, of which 18 min 46 s from pod to Ready.
- **Faults under load, in CI on every change:** a node drain and a
  preemption notice cut no stream in flight; the same test cuts 3 of 4
  streams without the fix
  ([ADR-0019](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0019-a-pod-leaves-the-service-before-it-stops-serving.md),
  [ADR-0024](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0024-the-simulator-runs-at-the-speed-of-one-l4.md)).
- **Scale from zero** through the KEDA HTTP add-on: first token at 8.2 s,
  worst wait 10.9 s, on kind
  ([ADR-0015](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0015-a-fleet-woken-from-zero-through-the-http-add-on.md)).
- **Alerting:** burn-rate alerts on the time to first token fired by email
  on real series from the L4
  ([ADR-0021](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0021-burn-rate-alerts-on-the-time-to-first-token.md)).
- **GPU budget:** 4.42 EUR spent of a 100 EUR ceiling; every run is
  estimated before it starts and reconciled with the bill after.

## Key decisions

1. **Cold start is a state, not a timeout**
   ([ADR-0002](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0002-coldstart-as-first-class-state.md)).
2. **A preemption or a drain moves replicas make-before-break, without
   cascade**
   ([ADR-0005](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0005-spot-preemption-reschedule-without-cascade.md),
   [ADR-0017](https://github.com/MicheleCampi/vllm-coldstart-operator/blob/main/docs/adr/0017-a-cordon-displaces-placements-like-a-preemption-notice.md)).
3. **Objectives per GPU class, with error budgets and burn-rate alerts**
   (ADR-0013, ADR-0020, ADR-0021, ADR-0022).
4. **The CI simulator runs at the speed of one L4, and says what it does
   not prove** (ADR-0024).
5. **Cost is a range tied to a load and its objectives, never one
   number** (ADR-0025).
6. **No secrets in git and no service-account keys:** Workload Identity
   and external-secrets
   ([gke-llm-inference-platform](https://github.com/MicheleCampi/gke-llm-inference-platform#design-decisions)).

## Status

- **Built and verified:** the operator and its CI, the GKE infrastructure
  with GitOps, objectives and alerts, faults under load, the calibrated
  simulator, the cost model.
- **In progress:** a public results report, a C4 architecture, runbooks,
  production readiness reviews and a threat model.

## Next, in order

1. **One platform definition for GKE and EKS.** The same Argo CD
   applications and Helm values on both clouds, with Terraform per cloud
   behind one contract
   ([eks-llm-inference-platform](https://github.com/MicheleCampi/eks-llm-inference-platform)).
2. **Security and supply chain.** A STRIDE threat model of every
   component, and a declared and verified SLSA level for the operator's
   image and chart.
3. **Production readiness.** A C4 architecture, runbooks and a failure-mode
   analysis for the faults already tested, and a readiness review per
   component.
4. **Model promotion into fleets.** The model-promotion bridge writes
   `FleetService`, not only `VllmService`, so a promoted model lands as a
   fleet.
5. **The simulator's tails.** Fit its dispersion to the L4, so CI can
   judge capacity as well as timing (ADR-0024).

## Evidence

- [vllm-coldstart-operator](https://github.com/MicheleCampi/vllm-coldstart-operator):
  25 architecture decision records, CI on multi-node kind.
- [gke-llm-inference-platform](https://github.com/MicheleCampi/gke-llm-inference-platform):
  Terraform, Argo CD applications, CI that validates and plans.
- [inferscope](https://github.com/MicheleCampi/inferscope): measurement
  and cost attribution for inference runs.
