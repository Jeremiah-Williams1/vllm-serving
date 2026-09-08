# vLLM Inference Platform on Kubernetes

An end-to-end AI infrastructure platform for serving LLMs on
GPU-backed Kubernetes: a custom operator, a CLI for model lifecycle
management, and this repo — the deployment layer tying them together,
validated both locally (minikube) and on real managed cloud
infrastructure (AWS EKS).

- **[llmservice-operator](https://github.com/Jeremiah-Williams1/llmservice-operator)**
  — Go Kubernetes operator: a custom `LLMService` CRD and reconciler
  that deploys vLLM, wires up GPU resource requests, a vLLM-aware
  readiness probe, and KEDA-based autoscaling.
- **llmservice-cli** — Cobra-based CLI wrapping the operator's API,
  so deploying, checking status on, and rolling back a model doesn't
  require hand-writing CRD YAML. *(add real link)*

## What's proven here

- The same manifests reconcile correctly on both a local dev cluster
  (minikube, GPU passthrough via `--gpus=all`) and real managed
  Kubernetes (EKS, GPU node group on `g4dn.xlarge`).
- Real inference: `TinyLlama/TinyLlama-1.1B-Chat-v1.0` served through
  vLLM's OpenAI-compatible API on an actual NVIDIA T4, on both
  clusters.
- Real, documented failure modes and fixes — see
  `docs/validation-notes.md` — including one still-open gap: KEDA's
  default CPU trigger doesn't scale GPU-bound inference.

## Repo structure

```
.
├── manifests/
│   └── llmservice-sample.yaml     # Sample LLMService CR (TinyLlama, 1 GPU)
├── cluster/
│   └── eks-cluster-config.yaml    # eksctl config: 1x g4dn.xlarge node group
├── docs/
│   └── validation-notes.md        # Real debugging log
└── README.md
```

## Prerequisites

- AWS account with EC2 quota for GPU instances (`g4dn.xlarge` needs
  4 vCPUs under the "Running On-Demand G and VT instances" quota —
  confirm this before creating a cluster, and remember EKS node
  groups draw from the *same* pool as any GPU EC2 instance you're
  running for local dev/build work; you cannot run both
  simultaneously without a higher quota)
- `aws` CLI configured, with IAM permissions covering EKS, EC2,
  CloudFormation, and IAM role creation (`eksctl` needs all of these)
- `eksctl` and `kubectl` installed
- Docker with `nvidia-container-runtime` set as default (for local
  GPU passthrough via minikube)

## Setup — local validation (minikube)

```bash
minikube start --driver=docker --container-runtime=docker --gpus=all

git clone https://github.com/Jeremiah-Williams1/llmservice-operator.git
cd llmservice-operator
make manifests
make install                    # installs the CRD

eval $(minikube docker-env)
make docker-build IMG=llmservice-operator:dev
minikube image load llmservice-operator:dev   # required even after
                                               # docker-build — minikube's
                                               # cluster runtime (containerd)
                                               # doesn't automatically see
                                               # images from the separate
                                               # Docker daemon docker-env
                                               # points at
make deploy IMG=llmservice-operator:dev

kubectl apply --server-side -f https://github.com/kedacore/keda/releases/download/v2.16.0/keda-2.16.0.yaml
kubectl apply -f ../llm-serving/manifests/llmservice-sample.yaml
kubectl get pods -w
```

## Setup — real EKS

```bash
cd llm-serving
eksctl create cluster -f cluster/eks-cluster-config.yaml
# newer eksctl versions auto-install the NVIDIA device plugin for
# GPU node types — confirm with:
kubectl get pods -n kube-system | grep nvidia

# Build + push the operator image from wherever Docker/the repo live
# (this project used a separate g4dn.xlarge EC2 instance for the
# build, since local dev and EKS validation shared the same GPU
# quota and couldn't run concurrently — see docs/validation-notes.md)
aws ecr create-repository --repository-name llmservice-operator --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
docker tag llmservice-operator:dev <account-id>.dkr.ecr.us-east-1.amazonaws.com/llmservice-operator:dev
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/llmservice-operator:dev

# Deploy — this can run from any machine with kubectl pointed at the
# cluster (doesn't need GPU access itself); edit
# config/manager/kustomization.yaml to point the image at your ECR
# URI before running:
kubectl apply -k config/crd
kubectl apply -k config/default

kubectl apply --server-side -f https://github.com/kedacore/keda/releases/download/v2.16.0/keda-2.16.0.yaml
kubectl apply -f manifests/llmservice-sample.yaml
```

## Tear down

```bash
eksctl delete cluster -f cluster/eks-cluster-config.yaml
```

Note: `eksctl scale nodegroup` does **not** merge with the existing
node group spec — it only applies the flags you explicitly pass, so
partial calls (e.g. just `--nodes=0`) can silently reset
`--nodes-min`/`--nodes-max` too. Specify all three every time, or
just delete/recreate the cluster for a clean state, which is what
this project ultimately did.

## Validated behavior

- ✅ CRD reconciles a real vLLM `Deployment`, `Service`, and KEDA
  `ScaledObject` from a single custom resource
- ✅ GPU resource requests (`nvidia.com/gpu`) scheduled correctly on
  both minikube and EKS
- ✅ Readiness probe correctly gates `Ready` status on vLLM actually
  finishing model load
- ✅ Real inference confirmed via the OpenAI-compatible
  `/v1/completions` endpoint on both clusters
- ⚠️ KEDA's default CPU-utilization trigger does not scale GPU-bound
  inference (see `docs/validation-notes.md`) — the correct trigger
  would be GPU utilization (DCGM) or vLLM's own queue-depth metric
- ⚠️ KEDA scaling pods to zero does **not** stop EC2 billing for the
  underlying node — that requires a node-level autoscaler (Cluster
  Autoscaler or Karpenter) or a scale-to-zero framework like Knative,
  neither of which this project implements yet