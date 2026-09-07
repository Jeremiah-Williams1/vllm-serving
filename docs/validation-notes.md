# Validation Notes

Real issues hit while validating the LLMService operator against a
real GPU workload, and how they were diagnosed — kept here rather than
smoothed over, since the debugging is as much a part of this project
as the working end state.

## 1. `nvidia-smi` succeeding ≠ CUDA actually works in a container

Early on, a container running `nvidia-smi` printed the GPU table
correctly but also emitted:

```
ERROR: driverInitFileInfo 578 result=11
ERROR: init 664 result=11
ERROR: init 250 result=11
```

This matched a known open NVIDIA container-toolkit issue where
`nvidia-smi`'s query path succeeds while actual CUDA context
initialization can still fail. Confirmed real GPU compute (not just
visibility) with a PyTorch tensor op inside a container before trusting
the result:

```bash
kubectl run gpu-compute-test --image=pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime \
  --restart=Never --overrides='{"spec":{"containers":[{"name":"gpu-compute-test",
  "image":"pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime","command":["python","-c",
  "import torch; print(torch.cuda.is_available()); x = torch.rand(1000,1000).cuda(); print(x.sum())"],
  "resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
```

Returned `True` and a real tensor sum — confirmed genuinely functional,
not just visible.

**Takeaway:** verify GPU access in containers with real compute, not
just a query command.

## 2. `kind` doesn't have native GPU passthrough

Initial plan was to validate on `kind`. Turns out `kind` has no
first-party GPU support — a proposal to add it was rejected upstream.
The community workaround (`nvkind` + CDI) is maintained but adds a
non-trivial extra dependency. Switched to **minikube**, which has
stable, first-party `--gpus=all` support on the Docker driver (marked
experimental in minikube's own docs, but far less fragile in practice
than the kind workaround).

**Takeaway:** confirm a tool's GPU support is first-party before
building a plan around it — "should work in Docker" doesn't imply
"works in every Docker-based Kubernetes tool."

## 3. Image built into minikube's Docker daemon still failed to pull

After `eval $(minikube docker-env)` + `docker build`, the image showed
up correctly in `docker images` inside minikube's daemon — but the
operator pod still failed with `ImagePullBackOff`, trying to pull from
a public registry:

```
Failed to pull image "llmservice-operator:dev": pull access denied for
llmservice-operator, repository does not exist or may require 'docker login'
```

Root cause: minikube's Kubernetes runtime uses containerd via CRI, which
doesn't automatically share images with the separate Docker daemon that
`docker-env` points at, even though both run inside the same minikube
VM. Fixed with an explicit load into the runtime kubelet actually pulls
from:

```bash
minikube image load llmservice-operator:dev
```

**Takeaway:** "the image exists in *a* Docker daemon on this host"
isn't the same as "the image exists in the runtime Kubernetes is
actually configured to use."

## 4. Severe EBS-adjacent I/O contention (root cause: concurrent cold starts, not disk type)

Mid-validation, `kubectl` began intermittently failing with
`TLS handshake timeout`. Diagnosis:

```bash
uptime
# load average: 66.10, 61.70, 35.49   (on a 4-vCPU instance)

vmstat 1 5
# wa (I/O wait) 34–63%, b (blocked on I/O) 50–82
```

Initially suspected an EBS `gp2` IOPS ceiling — but `aws ec2
describe-volumes` confirmed the volume was already `gp3` at 3000 IOPS /
125 MiB/s, ruling that out. The actual cause was concurrent cold-start
I/O load: minikube's control plane restarting, containerd
pulling/unpacking layers, and vLLM loading model weights, all
competing for disk at once on a single instance. Resolved with a clean
reboot and letting things start up one at a time rather than in
parallel.

**Takeaway:** high load average with low CPU usage per-process (as
seen in `top`) points to I/O wait, not compute load — check `vmstat`'s
`wa`/`b` columns before assuming a CPU or memory problem.

## 5. KEDA's CPU-based trigger doesn't scale GPU-bound inference

Load-testing the vLLM endpoint with concurrent completion requests
produced correct inference results, but the `ScaledObject` never
scaled beyond 1 replica. The reconciler's default KEDA trigger is:

```yaml
type: cpu
metadata:
  type: Utilization
  value: "50"
```

vLLM's inference work runs almost entirely on the GPU — CPU usage
barely moves under load, so the trigger metric never crosses its
threshold. This isn't a bug in the reconciler so much as a mismatch
between a CPU-shaped default and a GPU-bound workload.

**Correct fix (not yet implemented):** a GPU-utilization trigger
(via a Prometheus scaler reading DCGM metrics) or, better, a trigger
on vLLM's own exposed queue-depth metric (`num_requests_waiting`),
which reflects actual inference backpressure more directly than GPU
utilization alone.

**Takeaway:** default autoscaling metrics tuned for typical web
workloads (CPU) don't transfer to GPU-bound inference without
deliberate adjustment.
