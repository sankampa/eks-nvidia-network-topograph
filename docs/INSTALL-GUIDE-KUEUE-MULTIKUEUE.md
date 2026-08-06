# Installing Kueue + MultiKueue on EKS — Step-by-Step Guide

**Verified 2026-07-28 on the CGK shared EKS cluster** (`ap-southeast-3`, k8s v1.34.9-eks, p5en.48xlarge H200 nodes).
Every command in Parts 1–2 **and Part 5 (Topology-Aware Scheduling)** was run live on CGK and returned the
output shown. Part 4 (multi-cluster MultiKueue wiring) uses the schema proven in our 2026-07-25 3-cluster
E2E (`EVIDENCE-PASS-2026-07-25.txt`), adapted to real EKS.

- **Kueue** = a job queueing layer for Kubernetes: ClusterQueue (cluster-wide capacity) → LocalQueue
  (per-namespace handle) → jobs wait until quota is free, then get admitted.
- **MultiKueue** = one *management* cluster dispatches a whole job to one of several *worker* clusters,
  and mirrors status back. The job never splits across clusters (no cross-cluster interconnect seam).
- **Topology-Aware Scheduling (TAS)** = optional; packs a job onto the tightest network domain
  (leaf → block) using node labels. Covered in Part 5.

---

## Version note (READ THIS FIRST)

| Item | Value | Why |
|------|-------|-----|
| Kueue version | **v0.19.0** (released 2026-07-22) | latest stable; MultiKueue + TAS are **GA / on by default** here |
| API version | **v1beta1** (works) → **v1beta2** (current) | v0.19 prints `v1beta1 is deprecated, use v1beta2`. v1beta1 still applies fine; migrate field-for-field when you can. |
| kube-rbac-proxy sidecar | **GONE** in modern Kueue | The old `gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0` 404 → ImagePullBackOff gotcha (that bit us on v0.9.1) **does not exist on v0.19**. No sidecar to repoint. |
| Feature gates | **none needed** | On v0.9.x you had to pass `--feature-gates=TopologyAwareScheduling=true`. On v0.19 MultiKueue+TAS are GA, so the controller args are just `--config=... --zap-log-level=2`. |

> If you are stuck on Kueue v0.9.x for some reason, you DO need the rbac-proxy fix
> (`set image ... kube-rbac-proxy=quay.io/brancz/kube-rbac-proxy:v0.18.1`) and the TAS feature-gate patch.
> On v0.19 skip both.

---

## Part 1 — Install Kueue (do this on EVERY cluster: the mgmt cluster AND each worker)

```bash
# point at the target cluster first
export KUBECONFIG=$HOME/.kube/config-cgk        # (CGK example; use your own kubeconfig/context)

KUEUE_VER=v0.19.0
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VER}/manifests.yaml

kubectl -n kueue-system rollout status deploy/kueue-controller-manager --timeout=180s
```

**Verified output (CGK):**
```
deployment.apps/kueue-controller-manager serverside-applied
...
deployment "kueue-controller-manager" successfully rolled out
```

Confirm it's healthy + the CRDs you need exist:
```bash
kubectl -n kueue-system get pods
kubectl get crd | grep -iE "multikueue|topolog|admissioncheck|clusterqueue"
```
**Verified output (CGK):**
```
kueue-controller-manager-8d8ccfb68-flxbk   1/1   Running   0   29s
admissionchecks.kueue.x-k8s.io
multikueueclusters.kueue.x-k8s.io
multikueueconfigs.kueue.x-k8s.io
topologies.kueue.x-k8s.io
```
✅ Controller `1/1 Running`, all MultiKueue + TAS CRDs present. **This is the whole install** — single binary, no sidecar.

---

## Part 2 — Base queue objects (single-cluster; also the worker-side objects for MultiKueue)

Creates a namespace, a ResourceFlavor (the "kind of node"), a ClusterQueue (capacity), and a LocalQueue
(the namespace handle jobs point at).

```bash
kubectl create ns scale-ai-demo --dry-run=client -o yaml | kubectl apply -f -

cat <<'YAML' | kubectl apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: default-flavor}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata: {name: cgk-cluster-queue}
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["cpu","memory","nvidia.com/gpu"]   # include nvidia.com/gpu or GPU jobs never admit
    flavors:
    - name: default-flavor
      resources:
      - {name: "cpu",            nominalQuota: "500"}
      - {name: "memory",         nominalQuota: "4000Gi"}
      - {name: "nvidia.com/gpu", nominalQuota: "8"}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata: {name: cgk-local-queue, namespace: scale-ai-demo}
spec: {clusterQueue: cgk-cluster-queue}
YAML

kubectl get clusterqueue cgk-cluster-queue \
  -o jsonpath='{.status.conditions[?(@.type=="Active")].status}={.status.conditions[?(@.type=="Active")].reason}{"\n"}'
```
**Verified output (CGK):** `True=Ready`

> ⚠️ **The #1 GPU gotcha:** a ClusterQueue only admits resources it *covers*. If `coveredResources`
> lists only cpu/memory, a job requesting `nvidia.com/gpu` will sit **Pending forever** (never admitted).
> Always add `nvidia.com/gpu` (and `vpc.amazonaws.com/efa` if you gate on EFA devices).

### Smoke-test the queue (proves admission works)
```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: kueue-smoke
  namespace: scale-ai-demo
  labels: {kueue.x-k8s.io/queue-name: cgk-local-queue}   # THIS label routes the job into Kueue
spec:
  parallelism: 1
  completions: 1
  suspend: true            # REQUIRED: Kueue only manages jobs that start suspended; it un-suspends on admit
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: w
        image: public.ecr.aws/docker/library/busybox:1.36
        command: ["sh","-c","echo kueue-admitted on $(hostname); sleep 5; echo done"]
        resources: {requests: {cpu: "1", memory: "256Mi"}, limits: {cpu: "1", memory: "256Mi"}}
YAML

kubectl -n scale-ai-demo get workloads \
  -o custom-columns='WL:.metadata.name,ADMITTED:.status.conditions[?(@.type=="Admitted")].status,QUEUE:.spec.queueName'
kubectl -n scale-ai-demo get job kueue-smoke
```
**Verified output (CGK):**
```
WL                      ADMITTED   QUEUE
job-kueue-smoke-61307   True       cgk-local-queue
kueue-smoke   Complete   1/1   8s
```
✅ Job routed through the queue, admitted, ran, completed. **Single-cluster Kueue is done.**

Two non-negotiables for any Kueue-managed job:
1. `metadata.labels[kueue.x-k8s.io/queue-name]` = the LocalQueue name.
2. `spec.suspend: true` — Kueue admits by un-suspending. A non-suspended job bypasses Kueue entirely.

---

## Part 3 — MultiKueue: concept + topology

```
              ┌─────────────────────────┐
   submit ───▶│  MANAGEMENT cluster      │   (has Kueue + MultiKueueCluster/Config + AdmissionCheck)
   job here   │   ClusterQueue           │
              │    └─ AdmissionCheck ─────┼──┐  dispatches WHOLE job to ONE worker
              └─────────────────────────┘  │  (by MultiKueueConfig order + live quota)
                        ▲ status mirrors    │
                        │ back              ▼
        ┌───────────────┴──────┐   ┌────────────────────┐
        │ WORKER cluster A     │   │ WORKER cluster B    │   (each: plain Kueue + CQ/LQ from Part 2)
        │  (e.g. CGK p5en)     │   │ (e.g. east-2 P5)    │
        └──────────────────────┘   └────────────────────┘
```
- You submit to the **management** cluster's LocalQueue.
- The AdmissionCheck (`kueue.x-k8s.io/multikueue`) holds admission until a worker accepts.
- MultiKueue picks a worker (first in `MultiKueueConfig.spec.clusters` that has free quota) and creates the
  job there. If the preferred worker is starved, it **fails over** to the next.
- Job/pod status is mirrored back to mgmt, so `kubectl get jobs` on mgmt shows real progress.

**Worker side = just Part 1 + Part 2** (install Kueue, create default-flavor + CQ + LQ). Nothing MultiKueue-specific.

---

## Part 4 — Wire MultiKueue (management side)

> This is the schema **proven** in our 2026-07-25 E2E (`02-wire-multikueue.sh`, `EVIDENCE-PASS-2026-07-25.txt`:
> 2 jobs dispatched + a failover, both `MultiKueueCluster … Active`). The v1alpha1 `apiServer:`/`secretRef:`
> shape some drafts use is **wrong** — use the v1beta1 shape below.

### 4a. The EKS-specific gotcha: the worker kubeconfig must be a static ServiceAccount token

On kind you just do `kind get kubeconfig --internal`. **On EKS that does not work** — a normal EKS kubeconfig
uses `exec:` auth (`aws eks get-token`), and the Kueue controller pod has no `aws` CLI or IAM creds, so it
can't authenticate to the worker. You must hand MultiKueue a **static token** kubeconfig.

Kueue ships a helper that does exactly this. Run it **against each WORKER cluster** (path verified live
2026-07-28 — both URLs return HTTP 200):
```bash
# on/for each WORKER cluster:
export KUBECONFIG=<worker-kubeconfig>          # kubeconfig CURRENTLY pointing at the worker
curl -s https://kueue.sigs.k8s.io/examples/multikueue/create-multikueue-kubeconfig.sh \
  -o create-multikueue-kubeconfig.sh
# (raw-github fallback, same file:)
# curl -s https://raw.githubusercontent.com/kubernetes-sigs/kueue/${KUEUE_VER}/site/static/examples/multikueue/create-multikueue-kubeconfig.sh -o create-multikueue-kubeconfig.sh
chmod +x create-multikueue-kubeconfig.sh
./create-multikueue-kubeconfig.sh worker-A.kubeconfig
# -> creates a 'kueue-manager'/'kueue-batch-admin' SA + RBAC on the worker and writes a TOKEN-based
#    kubeconfig to worker-A.kubeconfig (a static token, NO exec:/aws-eks-get-token block).
```
> Note: MultiKueue validates kubeconfig files (rejects `exec:` blocks) as a code-execution guard — another
> reason the raw EKS kubeconfig won't work and you must use this token-based one.
(If you prefer to do it by hand: create a ServiceAccount + ClusterRoleBinding to `kueue-batch-admin-role`
on the worker, mint a non-expiring token Secret for it, and build a kubeconfig with
`users[].user.token: <token>` — no `exec:` block. The script above just automates that.)

### 4b. Store each worker kubeconfig as a Secret on the MANAGEMENT cluster
```bash
export KUBECONFIG=<mgmt-kubeconfig>
kubectl -n kueue-system create secret generic worker-a-kubeconfig \
  --from-file=kubeconfig=worker-A.kubeconfig
kubectl -n kueue-system create secret generic worker-b-kubeconfig \
  --from-file=kubeconfig=worker-B.kubeconfig
```

### 4c. Apply the MultiKueue objects on the MANAGEMENT cluster
```bash
cat <<'YAML' | kubectl apply -f -
# Register each worker (locationType: Secret + the secret name from 4b)
apiVersion: kueue.x-k8s.io/v1beta1
kind: MultiKueueCluster
metadata: {name: worker-a}
spec: {kubeConfig: {locationType: Secret, location: worker-a-kubeconfig}}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: MultiKueueCluster
metadata: {name: worker-b}
spec: {kubeConfig: {locationType: Secret, location: worker-b-kubeconfig}}
---
# Ordered preference: worker-a tried first, worker-b on failover
apiVersion: kueue.x-k8s.io/v1beta1
kind: MultiKueueConfig
metadata: {name: fleet-config}
spec: {clusters: [worker-a, worker-b]}
---
# The AdmissionCheck that makes a ClusterQueue dispatch via MultiKueue
apiVersion: kueue.x-k8s.io/v1beta1
kind: AdmissionCheck
metadata: {name: fleet-multikueue}
spec:
  controllerName: kueue.x-k8s.io/multikueue
  parameters: {apiGroup: kueue.x-k8s.io, kind: MultiKueueConfig, name: fleet-config}
---
# Management-side ResourceFlavor + ClusterQueue that references the AdmissionCheck + a LocalQueue
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: default-flavor}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata: {name: fleet-queue}
spec:
  namespaceSelector: {}
  admissionChecks: [fleet-multikueue]        # <-- this is what makes it multi-cluster
  resourceGroups:
  - coveredResources: ["cpu","memory","nvidia.com/gpu"]
    flavors:
    - name: default-flavor
      resources:
      - {name: "cpu",            nominalQuota: "1000"}
      - {name: "memory",         nominalQuota: "1000Gi"}
      - {name: "nvidia.com/gpu", nominalQuota: "100"}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata: {name: fleet-lq, namespace: scale-ai-demo}
spec: {clusterQueue: fleet-queue}
YAML
```

### 4d. Verify the workers connected (want Active=True)
```bash
kubectl get multikueuecluster \
  -o custom-columns='NAME:.metadata.name,ACTIVE:.status.conditions[0].status,REASON:.status.conditions[0].reason'
kubectl get admissioncheck fleet-multikueue \
  -o jsonpath='{.status.conditions[0].type}={.status.conditions[0].status}{"\n"}'
```
**Expected (matches 2026-07-25 PASS):**
```
NAME       ACTIVE   REASON
worker-a   True     Active
worker-b   True     Active
```
If `ACTIVE=False`: the token kubeconfig is wrong (exec-auth leaked in, expired token, or RBAC missing) —
re-run 4a. This is the single most common MultiKueue failure on EKS.

### 4e. Submit a job to the MANAGEMENT cluster → watch it dispatch
```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  generateName: multicluster-
  namespace: scale-ai-demo
  labels: {kueue.x-k8s.io/queue-name: fleet-lq}
spec:
  parallelism: 2
  completions: 2
  suspend: true
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: w
        image: public.ecr.aws/docker/library/busybox:1.36
        command: ["sh","-c","echo dispatched to $(hostname); sleep 12; echo done"]
        resources: {requests: {cpu: "1", memory: "256Mi"}, limits: {cpu: "1", memory: "256Mi"}}
YAML

# The reservation line tells you WHICH worker got it:
kubectl -n scale-ai-demo get workloads \
  -o custom-columns='WL:.metadata.name,QUOTA:.status.conditions[?(@.type=="QuotaReserved")].status,MSG:.status.conditions[?(@.type=="QuotaReserved")].message'
kubectl -n scale-ai-demo get jobs
```
**Expected (matches 2026-07-25 PASS):**
```
WL                                    QUOTA   MSG
job-multicluster-...                  True    The workload got reservation on "worker-a"
multicluster-...   Complete   2/2   24s
```
✅ Whole job dispatched to one worker; status mirrored back to mgmt.

---

## Part 5 — Topology-Aware Scheduling (TAS) — **verified live on CGK 2026-07-28**

TAS packs a whole job onto the tightest topology domain. It reads node labels: you declare a `Topology`
(ordered widest→narrowest levels), bind a `ResourceFlavor` to it, then a Job annotation names the level
the job must (or prefers to) fit inside.

### Which labels? Depends on the cluster (check first — don't assume)
```bash
# See what topology-ish labels your nodes actually expose:
kubectl get node <a-gpu-node> -o json | \
  python3 -c "import json,sys; l=json.load(sys.stdin)['metadata']['labels']; [print(k,'=',v) for k,v in sorted(l.items()) if any(t in k.lower() for t in ['topolog','zone','network','hostname','block','rack'])]"
```
- **east-2 P5 cluster:** `ml.p5.48xlarge` nodes expose `topology.k8s.aws/network-node-layer-{1,2,3}`
  (layer-1 = block/spine, layer-3 = leaf/ToR) — use those for a *fabric-domain* story.
- **CGK p5en:** those `network-node-layer-*` labels are **blank** — but the nodes DO expose
  `topology.kubernetes.io/zone` and `kubernetes.io/hostname`. TAS works fine using **zone → hostname**
  (verified below). Narrowest level = one host = tightest colocation you can express on CGK today.

### 5a. Two gotchas that will bite you (both hit + solved live on CGK v0.19.0)
1. **`Topology` is `v1beta2`, NOT `v1alpha1`.** Applying `apiVersion: kueue.x-k8s.io/v1alpha1` fails with
   `no matches for kind "Topology"`. Confirm served versions:
   `kubectl get crd topologies.kueue.x-k8s.io -o jsonpath='{range .spec.versions[*]}{.name}{" "}{end}'`
   → on v0.19 that's `v1beta1 v1beta2` (v1beta2 = storage).
2. **CRD name collision.** If `kai.scheduler` (or another scheduler) is installed, it ALSO owns a
   `topologies` short-name. `kubectl get topology <name>` resolves to the wrong group and returns
   `topologies.kai.scheduler "<name>" not found`. **Always use the fully-qualified name:**
   `kubectl get topologies.kueue.x-k8s.io <name>`.

On v0.19 **no feature gate** is required (TAS is GA). On v0.9.x you must add
`--feature-gates=TopologyAwareScheduling=true` to the controller args.

### 5b. Install the Topology + a TAS-bound flavor + CQ + LQ  [verified ✅ CGK]
```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: kueue.x-k8s.io/v1beta2            # NOT v1alpha1 (gotcha #1)
kind: Topology
metadata: {name: cgk-topology}
spec:
  levels:
  - nodeLabel: topology.kubernetes.io/zone    # AZ (widest)
  - nodeLabel: kubernetes.io/hostname         # node (narrowest = tightest domain on CGK)
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: p5en-tas-flavor}
spec:
  nodeLabels: {node.kubernetes.io/instance-type: p5en.48xlarge}
  topologyName: cgk-topology                  # binds this flavor to the Topology above
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata: {name: cgk-tas-queue}
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["cpu","memory"]
    flavors:
    - name: p5en-tas-flavor
      resources:
      - {name: "cpu",    nominalQuota: "300"}
      - {name: "memory", nominalQuota: "2000Gi"}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata: {name: cgk-tas-local-queue, namespace: scale-ai-demo}
spec: {clusterQueue: cgk-tas-queue}
YAML

kubectl get topologies.kueue.x-k8s.io cgk-topology       # fully-qualified (gotcha #2)
kubectl get clusterqueue cgk-tas-queue \
  -o jsonpath='{.status.conditions[?(@.type=="Active")].status}={.status.conditions[?(@.type=="Active")].reason}{"\n"}'
```
**Verified output (CGK):** `cgk-topology` present; ClusterQueue `True=Ready`.

### 5c. Submit a TAS job + prove it lands topology-aware  [verified ✅ CGK — the money shot]
```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: cgk-tas-demo
  namespace: scale-ai-demo
  labels: {kueue.x-k8s.io/queue-name: cgk-tas-local-queue}
spec:
  parallelism: 3
  completions: 3
  suspend: true
  backoffLimit: 0
  template:
    metadata:
      annotations:
        # hard constraint: all pods must fit within ONE value of this level (here: one host).
        # use ...podset-preferred-topology for best-effort instead of hard.
        kueue.x-k8s.io/podset-required-topology: kubernetes.io/hostname
    spec:
      restartPolicy: Never
      containers:
      - name: w
        image: public.ecr.aws/docker/library/busybox:1.36
        command: ["sh","-c","echo TAS pod on $NODE; sleep 8; echo done"]
        env: [{name: NODE, valueFrom: {fieldRef: {fieldPath: spec.nodeName}}}]
        resources: {requests: {cpu: "1", memory: "256Mi"}, limits: {cpu: "1", memory: "256Mi"}}
YAML

kubectl -n scale-ai-demo get workloads -o custom-columns='WL:.metadata.name,ADMITTED:.status.conditions[?(@.type=="Admitted")].status' | grep tas
# prove all pods landed on ONE node:
kubectl -n scale-ai-demo get pods -l job-name=cgk-tas-demo -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
kubectl -n scale-ai-demo get job cgk-tas-demo
```
**Verified output (CGK):**
```
job-cgk-tas-demo-c68ef   True                     # Admitted
ip-10-240-96-48.ap-southeast-3.compute.internal   # <- all 3 pods, ONE distinct node
cgk-tas-demo   Complete   3/3   13s
```
✅ Required-topology honored: all 3 pods packed onto a single host, job completed. Swap the annotation
to `topology.kubernetes.io/zone` to relax the constraint to "same AZ"; on east-2 P5 use
`topology.k8s.aws/network-node-layer-3` for "same leaf".

---

## Troubleshooting quick-reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| Job stays Pending, never Admitted | ClusterQueue doesn't cover the resource (esp. `nvidia.com/gpu`) | add it to `coveredResources` + a quota |
| Job runs immediately, ignores queue | missing `spec.suspend: true` or the `queue-name` label | add both |
| `MultiKueueCluster ACTIVE=False` on EKS | worker kubeconfig uses `exec:`/`aws eks get-token` | regenerate as a static SA-**token** kubeconfig (Part 4a) |
| ImagePullBackOff on `kube-rbac-proxy:v0.8.0` | you're on **old** Kueue (≤0.9.x) | upgrade to v0.19 (sidecar removed), or `set image ...=quay.io/brancz/kube-rbac-proxy:v0.18.1` |
| `v1beta1 is deprecated` warning | v0.19 prefers v1beta2 | harmless; migrate fields to `v1beta2` when convenient |
| TAS annotation ignored, pods spread | nodes lack `network-node-layer-*` labels (e.g. CGK p5en) | use a cluster that exposes them (east-2 P5), or drop the TAS annotation |

## Teardown
```bash
kubectl -n scale-ai-demo delete jobs --all
# full uninstall of Kueue:
kubectl delete -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.19.0/manifests.yaml
```

## Provenance
- Parts 1–2 + smoke test: **run live on CGK 2026-07-28**, output pasted above verbatim.
- Part 4 schema + expected output: **2026-07-25 3-cluster E2E** (`02-wire-multikueue.sh`,
  `EVIDENCE-PASS-2026-07-25.txt` — 2 dispatches + 1 failover, both workers Active).
- Part 5 label facts: east-2 P5 rehearsal 2026-07-28; CGK label-absence verified same day.
