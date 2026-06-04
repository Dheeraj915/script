# Radar HQ — Installation & Multi-Cluster Setup

| Component | Detail |
|-----------|--------|
| Radar runs on | OKE cluster (`context-cdcojlpzgja`) |
| Bastion | `vm-oci-prod-jenkins-master` |
| UI URL | `[http://radar.sirionlabs.tech:4013/` |
| Namespace | `radar` |

---

## Install Radar (one-time)

```bash
kubectl config use-context context-cdcojlpzgja
helm repo add skyhook https://skyhook-io.github.io/helm-charts
helm repo update
helm install radar skyhook/radar \
  --namespace radar --create-namespace \
  --set service.type=LoadBalancer \
  --set service.port=4013 \
  --set service.targetPort=9280
```

---

## Patch Deployment (first-time only)

```bash
kubectl config use-context context-cdcojlpzgja

kubectl -n radar patch deployment radar --type=json -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-",
   "value":{"name":"kubeconfig","secret":{"secretName":"radar-kubeconfig"}}}]'

kubectl -n radar patch deployment radar --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-",
   "value":{"name":"kubeconfig","mountPath":"/home/nonroot/.kube","readOnly":true}}]'

kubectl -n radar patch deployment radar --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/env",
   "value":[{"name":"KUBECONFIG","value":"/home/nonroot/.kube/config"}]}]'
```

---

## Setup Script

Script: `/root/radar/setup-eks-clusters.sh`

Creates `radar-admin` SA with `cluster-admin` role on EKS/OKE clusters. Uses existing static admin creds for AKS. Merges all into one kubeconfig and uploads to OKE as Secret.

Edit the cluster arrays at the top of the script:

```bash
# EKS — Format: "CLUSTER_NAME REGION AWS_PROFILE"
EKS_CLUSTERS=(
  "sbx-euw1-k8s-eks-apps eu-west-1 aws-sbx"
  ...
)

# OKE — Format: "DISPLAY_NAME CLUSTER_OCID REGION"
OKE_CLUSTERS=(
  "oke-jeddah-prod ocid1.cluster.oc1... me-jeddah-1"
)

# AKS — Format: "DISPLAY_NAME SOURCE_CONTEXT KUBECONFIG_PATH"
# SOURCE_CONTEXT must be the -admin context (static creds)
AKS_CLUSTERS=(
  "prd-uks-aks-main prd-uks-aks-main-admin /root/radar/kubeconfigs/prd-uks-aks-main.yaml"
)

# Bare metal / on-prem — Format: "DISPLAY_NAME SOURCE_CONTEXT KUBECONFIG_PATH"
# Kubeconfig must have static credentials (token or client cert)
BAREMETAL_CLUSTERS=(
  "bm-dc1-prod kubernetes-admin /root/radar/kubeconfigs/bm-dc1-prod.yaml"
)
```

Run:

```bash
/root/radar/setup-eks-clusters.sh
kubectl -n radar rollout restart deployment radar
```

---

## Adding New Clusters

1. Edit `/root/radar/setup-eks-clusters.sh` — add a line to the appropriate array
2. Run the script
3. `kubectl -n radar rollout restart deployment radar`

**Prerequisites by provider:**

| Provider | Requirement |
|----------|------------|
| EKS | AWS profile configured (`aws configure --profile <name>`) |
| OKE | OCI CLI configured on bastion |
| AKS | Copy admin kubeconfig to `/root/radar/kubeconfigs/` — must use `-admin` context with static creds, not exec-based |
| Bare metal | Copy kubeconfig with static credentials to `/root/radar/kubeconfigs/` |

---

## File Locations

| Path | Purpose |
|------|---------|
| `/root/radar/setup-eks-clusters.sh` | Setup script |
| `/root/radar/kubeconfigs/` | Individual kubeconfigs |
| `/root/radar/merged-kubeconfig.yaml` | Merged kubeconfig (mounted in pod) |

---

## Troubleshooting

```bash
kubectl -n radar logs deployment/radar
kubectl -n radar top pod
KUBECONFIG=/root/radar/merged-kubeconfig.yaml kubectl config get-contexts
grep -c "exec:" /root/radar/merged-kubeconfig.yaml   # must be 0
```
