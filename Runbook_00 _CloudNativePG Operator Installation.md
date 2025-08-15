# CloudNativePG Operator Installation on Kind

This runbook outlines the steps to install the CloudNativePG (CNP) Operator on a **Kind** Kubernetes cluster.

---

## Prerequisites

Ensure the following tools are installed and operational:

1. **kubectl** (v1.30 or later) — for managing Kubernetes clusters.
2. **Docker Desktop** — for container runtime.
3. **Kind** — for creating local Kubernetes clusters.

---

## Step 1: Create a Kind Cluster

Create a cluster with a specific name.

```
kind create cluster --name cnpg-1.25.0

# Example output
using docker due to KIND_EXPERIMENTAL_PROVIDER
Creating cluster "cnpg-1.25.0" ...
✓ Ensuring node image (kindest/node:v1.32.0) 🖼
✓ Preparing nodes 📦
✓ Writing configuration 📜
✓ Starting control-plane ️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
Set kubectl context to "kind-cnpg-1.25.0"
You can now use your cluster with:
kubectl cluster-info --context kind-cnpg-1.25.0
```

---

## Step 2: Check Cluster Details

Verify cluster information.

```
kubectl cluster-info --context kind-cnpg-1.25.0

# Example output
Kubernetes control plane is running at https://127.0.0.1:50807
CoreDNS is running at https://127.0.0.1:50807/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## Step 3: Deploy the CNP Operator

Install the CNPG Operator using the official release YAML.

```
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.0.yaml

# Example output
namespace/cnpg-system serverside-applied
customresourcedefinition.apiextensions.k8s.io/backups.postgres serverside-applied
...
deployment.apps/cnpg-controller-manager serverside-applied
mutatingwebhookconfiguration.admissionregistration.k8s.io/cnpg-mutating-webhook-configuration serverside-applied
validatingwebhookconfiguration.admissionregistration.k8s.io/cnpg-validating-webhook-configuration serverside-applied
```

---

## Step 4: Verify Deployment Status

Check if the CNP controller is running.

```
kubectl get deployment -n cnpg-system cnpg-controller-manager

# Example output
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
cnpg-controller-manager  1/1     1            1           47s
```

---

## Step 5: Verify Additional Details

Check Pods, Nodes, and Services in the `cnpg-system` namespace.

```
# Check Pods
kubectl get pod -n cnpg-system

NAME                                          READY   STATUS    RESTARTS   AGE
cnpg-controller-manager-7db9c889df-95km2      1/1     Running   0          41s

# Check Nodes
kubectl get node -n cnpg-system

NAME                          STATUS   ROLES           AGE   VERSION
cnpg-1.25.0-control-plane     Ready    control-plane   18m   v1.32.0

# Check Services
kubectl get svc -n cnpg-system

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
cnpg-webhook-service    ClusterIP   10.96.96.197    <none>        443/TCP   114s
```

---

## Step 6: List All Available Kind Clusters

Check all available contexts and clusters.

```
kubectl config get-contexts

CURRENT   NAME                         CLUSTER                      AUTHINFO                     NAMESPACE
*         kind-cnp-1.24.0              kind-cnp-1.24.0              kind-cnp-1.25.0
          kind-cnpg-1.25.0             kind-cnpg-1.25.0             kind-cnpg-1.25.0
```

---

## Reference

- [CloudNativePG Quickstart Guide](https://cloudnative-pg.io/documentation/current/quickstart/)
