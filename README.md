# Radar HQ — Installation & Multi-Cluster Setup

## Overview

Radar HQ is an open-source Kubernetes UI by Skyhook, installed on OKE (Oracle Cloud) and connected to multiple EKS clusters via static ServiceAccount tokens.

| Component | Detail |
|-----------|--------|
| Radar runs on | OKE cluster (`context-cdcojlpzgja`) |
| Bastion | `vm-oci-prod-jenkins-master` |
| UI URL | `http://161.118.180.16:4013` |
| Helm chart | `skyhook/radar` |
| Namespace | `radar` |

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  OKE Cluster (Radar installed here)                      │
│                                                          │
│  ┌─────────────────────────────────────────────┐         │
│  │  Radar Pod                                  │         │
│  │                                             │         │
│  │  /home/nonroot/.kube/config  (mounted)      │         │
│  │  KUBECONFIG env var set                     │         │
│  │                                             │         │
│  │  Reads static tokens from kubeconfig        │         │
│  │  Connects to all clusters directly          │         │
│  └──────────┬──────────┬──────────┬────────────┘         │
│             │          │          │                       │
└─────────────┼──────────┼──────────┼───────────────────────┘
              │          │          │
         ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
         │ EKS 1  │ │ EKS 2  │ │ EKS N  │
         │(token) │ │(token) │ │(token) │
         └────────┘ └────────┘ └────────┘
```

---

## Prerequisites

- `kubectl`, `helm`, `aws` CLI installed on the bastion
- AWS profiles configured in `~/.aws/credentials`:

```ini
[aws-sbx]
aws_access_key_id = AKIA...
aws_secret_access_key = ...

[aws-prd]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

Set up with:

```bash
aws configure --profile aws-sbx
aws configure --profile aws-prd
```

---

## Installation Steps

### Step 1 — Install Radar via Helm (one-time)

```bash
kubectl config use-context context-cdcojlpzgja

helm repo add skyhook https://skyhook-io.github.io/helm-charts
helm repo update

helm install radar skyhook/radar \
  --namespace radar \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set service.port=4013 \
  --set service.targetPort=9280

# Wait for external IP
kubectl -n radar get svc radar --watch
```

### Step 2 — Configure clusters in the setup script

Edit `/root/radar/setup-eks-clusters.sh` and add clusters:

```bash
SBX_CLUSTERS=(
  "sbx-euw1-k8s-eks-apps eu-west-1 aws-sbx"
  # Add more sbx clusters here...
)

PRD_CLUSTERS=(
  "prd-cluster-name us-east-1 aws-prd"
  # Add more prd clusters here...
)

OKE_CONTEXT="context-cdcojlpzgja"
```

Format: `"CLUSTER_NAME REGION AWS_PROFILE"`

### Step 3 — Run the setup script

```bash
chmod +x /root/radar/setup-eks-clusters.sh
/root/radar/setup-eks-clusters.sh
```

This script:
1. Creates a `radar-viewer` ServiceAccount on each EKS cluster
2. Binds it to the `view` ClusterRole (read-only)
3. Creates a permanent token Secret
4. Builds individual kubeconfigs with static tokens (no `aws` exec plugin)
5. Merges all kubeconfigs into `/root/radar/merged-kubeconfig.yaml`
6. Creates Secret `radar-kubeconfig` in the `radar` namespace on OKE

### Step 4 — Patch Radar deployment (first-time only)

```bash
kubectl config use-context context-cdcojlpzgja

# Add volume
kubectl -n radar patch deployment radar --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "kubeconfig",
      "secret": { "secretName": "radar-kubeconfig" }
    }
  }
]'

# Add volumeMount
kubectl -n radar patch deployment radar --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "kubeconfig",
      "mountPath": "/home/nonroot/.kube",
      "readOnly": true
    }
  }
]'

# Add KUBECONFIG env var
kubectl -n radar patch deployment radar --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env",
    "value": [{ "name": "KUBECONFIG", "value": "/home/nonroot/.kube/config" }]
  }
]'
```

### Step 5 — Verify

```bash
kubectl -n radar get pods -w
kubectl -n radar logs deployment/radar | head -50
echo "http://$(kubectl -n radar get svc radar -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):4013"
```

---

## Adding New Clusters

### Add a new EKS cluster

1. Edit the script:

```bash
vi /root/radar/setup-eks-clusters.sh
```

2. Add a line to the appropriate array:

```bash
SBX_CLUSTERS=(
  "sbx-euw1-k8s-eks-apps eu-west-1 aws-sbx"
  "new-cluster-name ap-south-1 aws-sbx"        # ← new
)
```

3. Re-run the script:

```bash
/root/radar/setup-eks-clusters.sh
```

4. Restart Radar to pick up the updated secret:

```bash
kubectl config use-context context-cdcojlpzgja
kubectl -n radar rollout restart deployment radar
```

### Add a new AWS profile (new set of credentials)

1. Configure the profile:

```bash
aws configure --profile aws-newteam
```

2. Use it in the script:

```bash
"cluster-name region aws-newteam"
```

---


## File Locations (on bastion)

| Path | Purpose |
|------|---------|
| `/root/radar/setup-eks-clusters.sh` | Main setup script |
| `/root/radar/kubeconfigs/` | Individual kubeconfig per cluster |
| `/root/radar/merged-kubeconfig.yaml` | Merged kubeconfig (mounted in pod) |
| `~/.aws/credentials` | AWS profiles (aws-sbx, aws-prd) |

---

## Key Constraints

- Radar container (`ghcr.io/skyhook-io/radar`) is a minimal Go binary — no `sh`, `bash`, `aws`, `cat`
- Helm chart does NOT support `extraEnv`, `extraVolumes`, `extraVolumeMounts` — use `kubectl patch`
- EKS exec-based auth doesn't work inside the pod — use static ServiceAccount tokens instead
- ServiceAccount tokens (`kubernetes.io/service-account-token` type) do not expire

---

## Troubleshooting

**Pod not starting:**
```bash
kubectl -n radar describe pod -l app.kubernetes.io/name=radar
kubectl -n radar logs deployment/radar
```

**Cluster not showing in UI:**
```bash
# Verify the merged kubeconfig has the context
KUBECONFIG=/root/radar/merged-kubeconfig.yaml kubectl config get-contexts

# Verify the secret was updated
kubectl -n radar get secret radar-kubeconfig -o jsonpath='{.data.config}' | base64 -d | grep -c "name:"
```

**Token not working:**
```bash
# Test the token from bastion
KUBECONFIG=/root/radar/kubeconfigs/<cluster>.yaml kubectl get namespaces
```
