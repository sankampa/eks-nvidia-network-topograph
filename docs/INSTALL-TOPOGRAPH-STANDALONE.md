# Installing NVIDIA Topograph (network topology labels) on EKS — standalone guide

Topograph queries the cloud provider's topology API and writes `network.topology.nvidia.com/{leaf,spine,core}` labels onto your nodes so a scheduler (Kueue TAS, Volcano, etc.) can pack a job onto the tightest network domain.

> **⚠ REGION GATE — READ FIRST (this is why it "did not work on the other cluster"):** Topograph on AWS depends on the EC2 **`DescribeInstanceTopology`** API. That API is **only available in a subset of regions** (mainly us-east-1, us-east-2, us-west-2, eu-west-1, etc.). On the CGK cluster (`ap-southeast-3`, Jakarta) it returns, **verified live 2026-07-28:**
>
> ```
> An error occurred (UnsupportedOperation) when calling the DescribeInstanceTopology operation:
> The functionality you requested is not available in this region.
> ```
>
> When that happens Topograph produces **no** `network.topology.nvidia.com/*` labels — nothing you set in the script can fix it, because the underlying data source doesn't exist in that region.
> → In an unsupported region, **skip Topograph and use the AWS-native `zone → hostname` fallback** (Part C below — proven green on CGK this session).

## Part A — Check if your region/cluster even supports Topograph (30 seconds, do this FIRST)

```bash
# instance IDs of your GPU nodes:
IDS=$(kubectl get nodes -l node.kubernetes.io/instance-type=p5en.48xlarge \
  -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' | sed 's|.*/||' | tr '\n' ' ')

# does the topology API answer in this region?
aws ec2 describe-instance-topology --region "$AWS_REGION" --instance-ids $IDS \
  --query 'Instances[].{id:InstanceId,net:NetworkNodes}' --output json
```

* **Returns a list with `NetworkNodes`** → supported. Go to Part B (install Topograph).
* **`UnsupportedOperation … not available in this region`** → NOT supported (e.g. ap-southeast-3). Skip to **Part C** (zone→hostname fallback). Do not bother installing Topograph here.
* **Empty `[]` but no error** → API exists but has no data for these instances (the HyperPod `DescribeInstanceTopology-[]` gap). Topograph installs but emits no labels → also use Part C.

## Part B — Install Topograph (only in a SUPPORTED region)

Prereqs: `helm`, `aws` CLI with EC2 read perms, a non-GPU node for the controller, IRSA (OIDC) enabled.

```bash
# 0) clone the chart (NVIDIA publishes it in-repo)
git clone https://github.com/NVIDIA/topograph
export HELM_CHART=$PWD/topograph/charts/topograph

# 1) context
export AWS_REGION=$(aws configure get region)
export CLUSTER_NAME=<your-eks-cluster-name>
export NS=topograph
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
       --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')

# 2) IRSA role: EC2 read-only, trust the cluster OIDC + the topograph SA
cat > trust.json <<JSON
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Federated":"arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC}"},
 "Action":"sts:AssumeRoleWithWebIdentity",
 "Condition":{"StringEquals":{"${OIDC}:sub":"system:serviceaccount:${NS}:topograph"}}}]}
JSON
aws iam create-role --role-name topograph-irsa --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name topograph-irsa \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
ROLE_ARN=arn:aws:iam::${ACCOUNT_ID}:role/topograph-irsa

# 3) install
kubectl create ns "$NS" --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install topograph "$HELM_CHART" -n "$NS" \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$ROLE_ARN"

# 4) the node-data broker needs host network to reach IMDS
kubectl -n "$NS" patch ds topograph-node-data-broker --type strategic \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}' || true
kubectl -n "$NS" rollout status deploy/topograph --timeout=180s

# 5) trigger label generation (provider + engine MUST be in the body)
kubectl -n "$NS" port-forward svc/topograph 49021:49021 >/dev/null 2>&1 & PF=$!; sleep 4
curl -sS -X POST http://127.0.0.1:49021/v1/generate \
  -H 'Content-Type: application/json' \
  -d '{"provider":{"name":"aws"},"engine":{"name":"k8s"}}'
sleep 16; kill "$PF" 2>/dev/null || true

# 6) verify labels landed
kubectl get nodes -L \
  network.topology.nvidia.com/leaf,network.topology.nvidia.com/spine,network.topology.nvidia.com/core
```

If leaf/spine/core are populated → feed them to Kueue TAS as topology levels.
If they are **blank** even here, re-run Part A — you're almost certainly in the unsupported-region or empty-`[]` case → use Part C.

## Part C — Fallback that WORKS in any region (verified green on CGK 2026-07-28)

No Topograph, no `DescribeInstanceTopology`. Use the topology labels EKS puts on every node natively: `topology.kubernetes.io/zone` (AZ) and `kubernetes.io/hostname` (node). This is enough for Kueue TAS to express "same AZ" or "same node" colocation.

```yaml
# Kueue Topology CR — NOTE apiVersion is v1beta2 on Kueue v0.19 (NOT v1alpha1)
apiVersion: kueue.x-k8s.io/v1beta2
kind: Topology
metadata: {name: aws-native-topology}
spec:
  levels:
  - nodeLabel: topology.kubernetes.io/zone      # AZ (widest)
  - nodeLabel: kubernetes.io/hostname           # node (narrowest = tightest domain)
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: p5en-tas-flavor}
spec:
  nodeLabels: {node.kubernetes.io/instance-type: p5en.48xlarge}
  topologyName: aws-native-topology
```

```bash
# gotcha: if kai.scheduler (or another scheduler) is installed it also owns a 'topologies' CRD.
# ALWAYS use the fully-qualified name so kubectl resolves the Kueue one:
kubectl get topologies.kueue.x-k8s.io aws-native-topology
```

Then on the Job template add the annotation naming the level to pack into:

```yaml
      annotations:
        kueue.x-k8s.io/podset-required-topology: kubernetes.io/hostname   # hard: one node
        # or: kueue.x-k8s.io/podset-preferred-topology: topology.kubernetes.io/zone   # best-effort: one AZ
```

**Verified on CGK:** a 3-pod job with `required-topology=kubernetes.io/hostname` → all 3 pods landed on one node (`ip-10-240-96-48`), job Complete 3/3.

## Cheat sheet: which labels exist where

| Cluster / region | `network.topology.nvidia.com/*` (Topograph) | `topology.k8s.aws/network-node-layer-{1,2,3}` | `topology.kubernetes.io/zone` + hostname |
|---|---|---|---|
| east-2 P5 (`ml.p5.48xlarge`) | via Topograph (region supported) | **present** (use directly) | always |
| **CGK p5en (`ap-southeast-3`)** | **NONE** — API unsupported in region | **blank** | **always → use this (Part C)** |
| any EKS, any region | only if `DescribeInstanceTopology` supported + non-empty | only where AWS populates them | **always** |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `network.topology.nvidia.com/*` labels after Topograph install | `DescribeInstanceTopology` `UnsupportedOperation` in this region | can't fix — use Part C zone→hostname |
| Labels blank, API returns `[]` (no error) | HyperPod/instance not in AWS topology dataset | use Part C |
| `topograph` pod CrashLoop / AccessDenied | IRSA role missing EC2 read or wrong OIDC sub | recheck trust.json `sub` = `system:serviceaccount:$NS:topograph` |
| `no matches for kind "Topology" v1alpha1` | Kueue v0.19 uses v1beta2 | change apiVersion to `kueue.x-k8s.io/v1beta2` |
| `topologies "x" not found` but you just made it | `kai.scheduler` CRD name collision | `kubectl get topologies.kueue.x-k8s.io x` |

## Provenance

* Region-gate + fallback: verified live on CGK (`ap-southeast-3`) 2026-07-28 — `DescribeInstanceTopology` → `UnsupportedOperation`; zone→hostname TAS job Complete 3/3 on one node.
* Part B install flow: adapted from the proven mixed p4d/HyperPod-p5/trn1 topograph deploy (2026-05-20).
