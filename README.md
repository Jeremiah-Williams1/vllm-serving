# vLLM Inference Platform on Kubernetes

An end-to-end AI infrastructure platform for serving LLMs on
GPU-backed Kubernetes: a custom operator, a CLI for model lifecycle
management, and this repo — the deployment layer tying them together
and proving they work against real GPU hardware, both locally and on
managed cloud infrastructure.

This is the deployment/validation layer of a three-part project. The
other two pieces:

- **[llmservice-operator](https://github.com/Jeremiah-Williams1/llmservice-operator)**
  — the Go Kubernetes operator: a custom `LLMService` CRD and
  reconciler that deploys vLLM, wires up GPU resource requests, a
  vLLM-aware readiness probe, and KEDA-based autoscaling.
- **llmservice-cli** — a Cobra-based CLI wrapping the operator's API,
  so deploying, checking status on, and rolling back a model doesn't
  require hand-writing CRD YAML. *(link once published)*

## What's proven here

- The same, unmodified manifests reconcile correctly on both a local
  dev cluster (minikube, with GPU passthrough via `--gpus=all`) and a
  real managed cluster (AWS EKS, GPU node group on `g4dn.xlarge`) —
  the operator isn't tied to a dev-only environment.
- Real inference, not a mock: `TinyLlama/TinyLlama-1.1B-Chat-v1.0`
  served through vLLM's OpenAI-compatible API, running on an actual
  NVIDIA T4.
- Real, honestly-documented failure modes and fixes along the way —
  see `docs/validation-notes.md` — rather than a sanitized "it just
  worked" writeup.

## Repo structure

```
.
├── manifests/
│   └── llmservice-sample.yaml     # Sample LLMService CR (TinyLlama, 1 GPU)
├── cluster/
│   └── eks-cluster-config.yaml    # eksctl config: 1x g4dn.xlarge node group
├── docs/
│   └── validation-notes.md        # Debugging log: GPU passthrough, EBS I/O
│                                   # contention, KEDA trigger mismatch
└── README.md
```

## Prerequisites

- An AWS account with EC2 quota for GPU instances (this project used
  `g4dn.xlarge` — confirm your account's "Running On-Demand G and VT
  instances" quota covers at least 4 vCPUs before starting)
- `aws` CLI configured with credentials
- `eksctl`, if deploying to real EKS rather than just validating
  locally

## Setup

### 1. Spin up a GPU-capable EC2 instance

Launch a `g4dn.xlarge` on an **AWS Deep Learning AMI** — this is what
was used here, and it ships the NVIDIA driver preinstalled, so
`nvidia-smi` works immediately with no manual driver install step.

### 2. Install remaining dependencies on the instance

```bash
# Set nvidia as Docker's default runtime
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo systemctl restart docker

# Go (matches this project's go.mod version)
wget https://go.dev/dl/go1.26.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.26.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# minikube (for local validation)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 3. Clone the operator and this repo

```bash
git clone https://github.com/Jeremiah-Williams1/llmservice-operator.git
git clone https://github.com/Jeremiah-Williams1/llm-serving.git
```

### 4. Validate locally on minikube

```bash
minikube start --driver=docker --container-runtime=docker --gpus=all

cd llmservice-operator
make manifests
make install          # installs the CRD

eval $(minikube docker-env)
make docker-build IMG=llmservice-operator:dev
minikube image load llmservice-operator:dev
make deploy IMG=llmservice-operator:dev

kubectl apply -f ../llm-serving/manifests/llmservice-sample.yaml
kubectl get pods -w
```

### 5. Deploy to real EKS

```bash
cd llm-serving
eksctl create cluster -f cluster/eks-cluster-config.yaml

# Apply the NVIDIA device plugin (not automatic on EKS, unlike minikube's --gpus flag)
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml

# Push the operator image to ECR, then:
cd ../llmservice-operator
make deploy IMG=<your-ecr-uri>/llmservice-operator:dev

kubectl apply -f ../llm-serving/manifests/llmservice-sample.yaml
```

### 6. Tear down when done

```bash
eksctl delete cluster -f cluster/eks-cluster-config.yaml
```

EKS control plane and GPU node group both cost money while running —
this was run as a validation exercise, not a persistent deployment.

## Validated behavior

- ✅ CRD reconciles a real vLLM `Deployment`, `Service`, and KEDA
  `ScaledObject` from a single custom resource
- ✅ GPU resource requests (`nvidia.com/gpu`) scheduled correctly on
  both minikube and EKS
- ✅ Readiness probe correctly gates `Ready` status on vLLM actually
  finishing model load, not just container start
- ✅ Real inference confirmed via the OpenAI-compatible
  `/v1/completions` endpoint
- ⚠️ KEDA's default CPU-utilization trigger does **not** scale
  GPU-bound inference workloads — documented in
  `docs/validation-notes.md` as a known limitation. The correct
  trigger for this workload would be GPU utilization (DCGM) or vLLM's
  own queue-depth metric.
