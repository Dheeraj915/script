# Radar HQ — Installation & Multi-Cluster Setup

| Component | Detail |
|-----------|--------|
| Radar runs on | OKE cluster (`context-cdcojlpzgja`) |
| Bastion | `vm-oci-prod-jenkins-master` |
| UI URL | `http://161.118.180.16:4013` |
| Namespace | `radar` |

---

## Step 1 — AWS Profiles (one-time)

```bash
aws configure --profile aws-sbx
aws configure --profile aws-prd
```

---

## Step 2 — Install Radar (one-time)

```bash
kubectl config use-context context-cdcojlpzgja
helm repo add skyhook https://skyhook-io.github.io/helm-charts
helm repo update
helm install radar skyhook/radar \
  --namespace radar --create-namespace \
  --set service.type=LoadBalancer \
  --set service.port=4013 \
  --set service.targetPort=9280
kubectl -n radar get svc radar --watch   # wait for EXTERNAL-IP
```

---

## Step 3 — Setup Script

Script location: `/root/radar/setup-eks-clusters.sh`

**Edit clusters here:**

```bash
SBX_CLUSTERS=(
  "sbx-euw1-k8s-eks-apps eu-west-1 aws-sbx"
)
PRD_CLUSTERS=(
  "prd-use1-k8s-eks-apps us-east-1 aws-prd"
)
OKE_CONTEXT="context-cdcojlpzgja"
```

Format: `"CLUSTER_NAME REGION AWS_PROFILE"`

**Run:**

```bash
/root/radar/setup-eks-clusters.sh
```

**What it does:**
1. Creates `radar-viewer` ServiceAccount + token on each EKS cluster
2. Grants `view` + ArgoCD/KEDA read permissions
3. Builds merged kubeconfig with static tokens
4. Uploads it as Secret `radar-kubeconfig` to OKE

---

## Step 4 — Patch Radar Deployment (first-time only)

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

## Step 5 — Verify

```bash
kubectl -n radar get pods -w
kubectl -n radar logs deployment/radar | head -50
echo "http://$(kubectl -n radar get svc radar -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):4013"
```

---

## Adding New Clusters

1. Edit: `vi /root/radar/setup-eks-clusters.sh` — add a line
2. Run: `/root/radar/setup-eks-clusters.sh`
3. Restart: `kubectl -n radar rollout restart deployment radar`

---

## Adding a New AWS Profile

```bash
aws configure --profile aws-newteam
```

Then use `"cluster-name region aws-newteam"` in the script.

---

## File Locations

| Path | Purpose |
|------|---------|
| `/root/radar/setup-eks-clusters.sh` | Main setup script |
| `/root/radar/kubeconfigs/` | Individual kubeconfig per cluster |
| `/root/radar/merged-kubeconfig.yaml` | Merged kubeconfig (mounted in pod) |

---

## Troubleshooting

```bash
kubectl -n radar describe pod -l app.kubernetes.io/name=radar     # pod issues
kubectl -n radar logs deployment/radar                             # logs
KUBECONFIG=/root/radar/merged-kubeconfig.yaml kubectl config get-contexts  # verify contexts
KUBECONFIG=/root/radar/kubeconfigs/<cluster>.yaml kubectl get ns   # test a token
```
