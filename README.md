# EKS Topology-Aware Scheduling with NVIDIA Topograph + Kueue TAS + MultiKueue

Topology-aware GPU scheduling on Amazon EKS using **NVIDIA Topograph** for network topology discovery, **Kueue Topology-Aware Scheduling (TAS)** for rack-local placement, and **MultiKueue** for multi-cluster job dispatch.

Everything here was validated live on real EKS clusters (Kueue v0.19.0, EKS k8s v1.34+).

---

## What This Solves

Two independent scheduling problems:

| Question | Layer | What it guarantees |
|---|---|---|
| Which **cluster** should this job run on? | MultiKueue | Each job runs entirely on one cluster. Never straddles a no-fast-interconnect seam. |
| Which **racks/nodes** inside that cluster should the job's pods land on? | Kueue TAS + Topograph | A multi-node training job is packed onto the tightest network domain (same rack), so all-reduce traffic rides the shortest path. |

They compose: submit once → MultiKueue picks a cluster → TAS packs pods onto one rack.

---

## Repo Structure

```
├── README.md                              ← You are here
├── docs/
│   ├── INSTALL-TOPOGRAPH-STANDALONE.md    ← NVIDIA Topograph install + region-gate + fallback
│   └── INSTALL-GUIDE-KUEUE-MULTIKUEUE.md ← Kueue + MultiKueue + TAS install guide
├── scripts/
│   └── create-multikueue-kubeconfig.sh    ← Generates static-token kubeconfig for workers
├── kueue-tas/
│   ├── topology-tas-flavor.yaml           ← Topology + ResourceFlavor + ClusterQueue + LocalQueue
│   ├── submit-topology-job.yaml           ← Sample TAS job (pack 3 pods on one node)
│   ├── topology-aware-scheduling-cgk-p5en.yaml   ← Full TAS config for p5en cluster
│   └── topology-aware-scheduling-poc2-p3dn.yaml  ← Full TAS config for p3dn cluster
├── multikueue/
│   ├── management-cluster.yaml            ← Management cluster MultiKueue objects
│   ├── worker-cluster.yaml                ← Worker cluster queue + topology objects
│   ├── multikue-objects.yaml              ← Compact MultiKueue wiring (minimal)
│   └── sample-multicluster-job.yaml       ← Sample multi-cluster job with TAS annotation
└── configs/
    ├── worker-A.kubeconfig                ← Template (tokens redacted, generated at runtime)
    └── worker-B.kubeconfig                ← Template (tokens redacted, generated at runtime)
```

---

## Quick Start

### Prerequisites

- EKS clusters (one management + one or more workers)
- `kubectl`, `helm`, `aws` CLI v2
- IAM: `ec2:DescribeInstanceTopology` + `AmazonEC2ReadOnlyAccess` (for Topograph IRSA)
- Kueue v0.19.0 installed on all clusters

### 1. Install Kueue on all clusters

```bash
KUEUE_VER=v0.19.0
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VER}/manifests.yaml
kubectl -n kueue-system rollout status deploy/kueue-controller-manager --timeout=180s
```

### 2. Deploy Topograph for topology labels

Follow [`docs/INSTALL-TOPOGRAPH-STANDALONE.md`](docs/INSTALL-TOPOGRAPH-STANDALONE.md):
- **Part A** — Check if your region supports the topology API
- **Part B** — Install Topograph (writes `network.topology.nvidia.com/{leaf,spine,core}` labels)
- **Part C** — Fallback using `zone → hostname` labels (works in any region)

### 3. Set up Kueue TAS (single-cluster)

```bash
kubectl apply -f kueue-tas/topology-tas-flavor.yaml
kubectl apply -f kueue-tas/submit-topology-job.yaml
```

See [`docs/INSTALL-GUIDE-KUEUE-MULTIKUEUE.md`](docs/INSTALL-GUIDE-KUEUE-MULTIKUEUE.md) Part 5 for details.

### 4. Set up MultiKueue (multi-cluster)

```bash
# On each worker: generate a static-token kubeconfig
./scripts/create-multikueue-kubeconfig.sh worker-A.kubeconfig

# On management cluster: store as secrets + apply MultiKueue objects
kubectl -n kueue-system create secret generic worker-a-kubeconfig \
  --from-file=kubeconfig=worker-A.kubeconfig
kubectl apply -f multikueue/management-cluster.yaml

# Submit a job
kubectl apply -f multikueue/sample-multicluster-job.yaml
```

See [`docs/INSTALL-GUIDE-KUEUE-MULTIKUEUE.md`](docs/INSTALL-GUIDE-KUEUE-MULTIKUEUE.md) Part 4 for the full walkthrough.

---

## Key Gotchas

| Issue | Fix |
|---|---|
| Topograph produces no labels | Check region support first (Part A). Use zone→hostname fallback in unsupported regions. |
| `no matches for kind "Topology" v1alpha1` | Kueue v0.19 uses `v1beta2` |
| `MultiKueueCluster ACTIVE=False` | Worker kubeconfig has `exec:` block — regenerate with `create-multikueue-kubeconfig.sh` |
| Missing CRDs on workers (JobSet, Ray, MPIJob, etc.) | Kueue v0.19 requires all framework CRDs on workers. Install stubs for unused frameworks. |
| Job runs immediately, ignores queue | Missing `spec.suspend: true` or `kueue.x-k8s.io/queue-name` label |
| TAS annotation ignored | Nodes lack topology labels — verify with `kubectl get nodes -L topology.k8s.aws/network-node-layer-3` |

---

## References

- [Kueue Topology-Aware Scheduling](https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/)
- [Kueue MultiKueue](https://kueue.sigs.k8s.io/docs/concepts/multikueue/)
- [MultiKueue Setup Guide](https://kueue.sigs.k8s.io/docs/tasks/manage/setup_multikueue/)
- [NVIDIA Topograph](https://github.com/NVIDIA/topograph)
- [EC2 Instance Topology Prerequisites](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-topology-prerequisites.html)
- [DescribeInstanceTopology API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstanceTopology.html)
- [EFA on EKS](https://docs.aws.amazon.com/eks/latest/userguide/node-efa.html)

---

## Validated On

- EKS clusters: us-east-1 (p3dn.24xlarge), ap-southeast-3 (p5en.48xlarge)
- Kueue v0.19.0, Kubernetes v1.34+
- NVIDIA Topograph (region-supported clusters)
- MultiKueue E2E: 3-cluster setup (1 management + 2 workers), job dispatch + status mirror confirmed
