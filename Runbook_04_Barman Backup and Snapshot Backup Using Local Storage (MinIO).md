
# Barman Backup and Snapshot Backup Using Local Storage (MinIO)

This runbook documents the step-by-step procedure to set up a **CloudNativePG** PostgreSQL cluster with **MinIO** object storage for backups using both **Volume Snapshots** and **Barman Object Store**.  

---

## Step 1: Create a Kind Cluster

Use the following syntax to create a cluster with a specific name.

**Command Syntax:**

```
kind create cluster --name <cluster_name>
````

**Example Execution:**

```
% kind create cluster --name minio-cnpg-1.25.0

using docker due to KIND_EXPERIMENTAL_PROVIDER
Creating cluster “minio-cnpg-1.25.0" ...
✓ Ensuring node image (kindest/node:v1.32.0) 🖼
✓ Preparing nodes 📦
✓ Writing configuration 📜
✓ Starting control-plane ️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
Set kubectl context to "kind-cnpg-1.25.0"
You can now use your cluster with:
kubectl cluster-info --context kind-minio-cnpg-1.25.0
```

---

## Step 2: Check the Cluster Details

```
kubectl cluster-info --context kind-minio-cnpg-1.25.0
```

---

## Step 3: Deploy the CNPG Operator

```
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.0.yaml
```

---

## Step 4: Verify Deployment Status

```
kubectl get deployment -n cnpg-system cnpg-controller-manager
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
cnpg-controller-manager   1/1     1            1           47s
```

---

## Step 5: Install the CloudNativePG Plugin

```
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | sudo sh -s -- -b /usr/local/bin
```

Verify installation:

```
kubectl cnpg version
```

---

## Step 6: Install MinIO Object Storage

```
% vi minio.yaml

apiVersion: v1
kind: Secret
metadata:
  name: minio-creds
data:
  ACCESS_KEY_ID: bWluaW8=
  ACCESS_SECRET_KEY: bWluaW8xMjM=
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  ports:
    - port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    app: minio
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-pv-claim
      containers:
        - name: minio
          image: minio/minio:RELEASE.2025-01-20T14-49-07Z
          args:
            - server
            - /data
          env:
            - name: MINIO_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: minio-creds
                  key: ACCESS_KEY_ID
            - name: MINIO_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: minio-creds
                  key: ACCESS_SECRET_KEY
          ports:
            - containerPort: 9000
          readinessProbe:
            httpGet:
              path: /minio/health/ready
              port: 9000
            initialDelaySeconds: 30
          livenessProbe:
            httpGet:
              path: /minio/health/live
              port: 9000
            initialDelaySeconds: 30
```

---

## Step 7: Apply and Verify MinIO Deployment

```
kubectl apply -f minio.yaml
kubectl get deployment minio
```

---

## Step 8: Create and Deploy HostPath CSI for Volume Snapshots and Barman Backups

### A. Create Deployment Script

```
% vi deploy-hostpath-csi.sh

#!/bin/env bash
CSI_BASE_URL=https://raw.githubusercontent.com/kubernetes-csi
CSI_DRIVER_HOST_PATH_VERSION=v1.11.0
SNAPSHOTTER_VERSION="v6.3.1"
PROVISIONER_VERSION="v3.6.1"
RESIZER_VERSION="v1.9.1"
ATTACHER_VERSION="v4.4.1"

## Install external snapshotter CRD
kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/client/config/crd/snapshot.storage.k8s.io_volumesnapshot.yaml
kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontent.yaml
kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclass.yaml

kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
kubectl apply -f "${CSI_BASE_URL}"/external-snapshotter/"${SNAPSHOTTER_VERSION}"/deploy/kubernetes/csi-snapshotter/rbac-csi-snapshotter.yaml

## Install external provisioner
kubectl apply -f "${CSI_BASE_URL}"/external-provisioner/"${PROVISIONER_VERSION}"/deploy/kubernetes/rbac.yaml

## Install external attacher
kubectl apply -f "${CSI_BASE_URL}"/external-attacher/"${ATTACHER_VERSION}"/deploy/kubernetes/rbac.yaml

## Install external resizer
kubectl apply -f "${CSI_BASE_URL}"/external-resizer/"${RESIZER_VERSION}"/deploy/kubernetes/rbac.yaml

## Install driver and plugin
kubectl apply -f "${CSI_BASE_URL}"/csi-driver-host-path/"${CSI_DRIVER_HOST_PATH_VERSION}"/deploy/kubernetes-1.24/hostpath/csi-hostpath-driverinfo.yaml
kubectl apply -f "${CSI_BASE_URL}"/csi-driver-host-path/"${CSI_DRIVER_HOST_PATH_VERSION}"/deploy/kubernetes-1.24/hostpath/csi-hostpath-plugin.yaml

## create volumesnapshotclass
kubectl apply -f "${CSI_BASE_URL}"/csi-driver-host-path/"${CSI_DRIVER_HOST_PATH_VERSION}"/deploy/kubernetes-1.24/hostpath/csi-hostpath-snapshotclass.yaml

## create storage class
kubectl apply -f "${CSI_BASE_URL}"/csi-driver-host-path/"${CSI_DRIVER_HOST_PATH_VERSION}"/examples/csi-storageclass.yaml
```

### B. Deploy

```
% bash deploy-hostpath-csi.sh
```

### C. Verify

```
% kubectl get deployment -A
NAMESPACE            NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
default              minio                        1/1     1            1           4m4s
kube-system          coredns                      2/2     2            2           23m
kube-system          snapshot-controller          2/2     2            2           36s
local-path-storage   local-path-provisioner       1/1     1            1           22m
```

---

## Step 9: Restart PostgreSQL Operator Controller Manager

```
kubectl rollout restart deployment -n postgresql-operator-system postgresql-operator-controller-manager
```

---

## Step 10: Deploy PostgreSQL Cluster with Minio Storage

### A. Create

```
% vi cluster-minio.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-snapshot
spec:
  instances: 2
  storage:
    storageClass: csi-hostpath-sc
    size: 1Gi
  backup:
    volumeSnapshot:
      className: csi-hostpath-snapclass
    barmanObjectStore:
      destinationPath: s3://swapnil-backup/
      endpointURL: http://minio-service:9000
      s3Credentials:
        accessKeyId:
          name: minio-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: minio-creds
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
    data:
      immediateCheckpoint
```

### B. Apply

```
kubectl apply -f cluster-minio.yaml
```

### C. Verify

```
% kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
cluster-snapshot-1           1/1     Running   0          72s
cluster-snapshot-2           1/1     Running   0          33s
csi-hostpathplugin-0         8/8     Running   0          8m35s
minio-74bb68884d-dp4wn        1/1     Running   0          12m
```

---

## Step 11: Verify PostgreSQL Cluster Status

```
% kubectl get cluster
NAME               AGE    INSTANCES   READY   STATUS                   PRIMARY
cluster-snapshot   3m1s   2           2       Cluster in healthy state cluster-snapshot-1
```

---

## Step 12: Check Detailed Cluster Status

```
% kubectl cnpg status cluster-snapshot

Cluster Summary
Name: default/cluster-snapshot
System ID: 7464614539111931932
PostgreSQL Image: quay.io/enterprisedb/postgresql:17.2
Primary instance: cluster-snapshot-1
Primary start time: 2025-01-27 15:17:05 +0000 UTC (uptime 1m42s)
Status: Cluster in healthy state
Instances: 2
Ready instances: 2
Size: 94M

Current Write LSN: 0/4000060 (Timeline: 1 - WAL File: 000000010000000000000004)
Continuous Backup status
First Point of Recoverability: Not Available
Working WAL archiving: OK
WALs waiting to be archived: 0
Last Archived WAL: 000000010000000000000003.00000028.backup @ 2025-01-27T15:17:29.518622Z
Last Failed WAL: -

Streaming Replication status

Replication Slots Enabled
Name Sent LSN Write LSN Flush LSN Replay LSN Write Lag Flush Lag Replay Lag State Sync State Sync Priority Replication Slot
cluster-snapshot-2 0/4000060 0/4000060 0/4000060 0/4000060 00:00:00 00:00:00 00:00:00 streaming async 0 active

Instances status
Name Current LSN Replication role Status QoS Manager Version Node
cluster-snapshot-1 0/4000060 Primary OK BestEffort 1.25.0 miniio-cnp-1.25.0-control-plane
cluster-snapshot-2 0/4000060 Standby (async) OK BestEffort 1.25.0 miniio-cnp-1.25.0-control-plane
```

---

## Method 1: Backup using Plugin

### Step 13: Perform Volume Snapshot Backup

```
% kubectl cnpg backup cluster-snapshot -m volumeSnapshot
backup/cluster-snapshot-20250127205000 created
```

### Step 14: Verify Backup Completion

```
% kubectl get backup

NAME                                 AGE   CLUSTER           METHOD          PHASE       ERROR
cluster-snapshot-20250127205000      8s    cluster-snapshot  volumeSnapshot  completed

% kubectl get volumeSnapshot

NAME                                 READYTOUSE   SOURCEPVC          SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS             SNAPSHOTCONTENT                                           CREATIONTIME   AGE
cluster-snapshot-20250127205000      true         cluster-snapshot-2                          1Gi           csi-hostpath-snapclass   snapcontent-40172d26-8468-4ddc-a074-5f3aa281384e          17s            17s
```

---

### Step 15: Perform Barman Backup

```
% kubectl cnpg backup cluster-snapshot
backup/cluster-snapshot-20250127205100 created
```

### Step 16: Verify Backup Completion

```
% kubectl get backup
NAME                                 AGE   CLUSTER           METHOD            PHASE       ERROR
cluster-snapshot-20250127205000      64s   cluster-snapshot  volumeSnapshot    completed
cluster-snapshot-20250127205100      4s    cluster-snapshot  barmanObjectStore completed
```

---

## Method 2: Backup using YAML File

```
% vi volume-snapshot-backup.yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Backup
metadata:
  name: volume-snapshot-backup
spec:
  cluster:
    name: cluster-snapshot
  method: volumeSnapshot

% kubectl apply -f volume-snapshot-backup.yaml
backup.postgresql.k8s.enterprisedb.io/volume-snapshot-backup created

% kubectl get volumeSnapshot
NAME                     READYTOUSE   SOURCEPVC          SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS             SNAPSHOTCONTENT                                           CREATIONTIME   AGE
volume-snapshot-backup   true         cluster-snapshot-2                          1Gi           csi-hostpath-snapclass    snapcontent-d26061ef-9611-42bc-bb33-2974b8f691e2          36s            36s
```

---

## MinIO Client Setup and Backup Verification

### Step 1: Download MinIO Client (mc)

For macOS:

```
curl -LO https://dl.min.io/client/mc/release/darwin-amd64/mc
```

For Linux:

```
curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
```

### Step 2: Make Binary Executable

```
chmod +x mc
```

### Step 3: Move Binary to `/usr/local/bin`

```
sudo mv -f mc /usr/local/bin/mc
```

### Step 4: Verify Installation

```
mc --help
```

### Step 5: Set Up Port Forwarding for MinIO

```
kubectl port-forward svc/minio-service 9000:9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
```

### Step 6: Configure MinIO Client (mc)

```
mc alias list
```

### Step 7: Remove Any Existing MinIO Alias

```
mc alias remove minio
```

### Step 8: Reconfirm Alias Removal

```
mc alias list
```

### Step 9: Set Up MinIO Alias

```
ACCESS_KEY_ID=$(echo -n minio | base64)
ACCESS_SECRET_KEY=$(echo -n minio123 | base64)

mc alias set 'minio' 'http://127.0.0.1:9000' $(echo -n $ACCESS_KEY_ID | base64 -d) $(echo -n $ACCESS_SECRET_KEY | base64 -d)
```

### Step 10: Confirm Alias Setup

```
mc alias list
```

### Step 11: Verify MinIO Connection

**1. Retrieve MinIO Server Information**

```
mc admin info minio
```

**2. List Buckets in MinIO**

```
mc ls minio
[2025-01-29 16:14:37 IST] 0B swapnil-backup/
```

**3. Deleting Data and Buckets in MinIO**

```
mc rm --recursive --force minio/swapnil-backup

Removed `minio/swapnil-backup/test/cluster-backup/wals/0000000100000000/000000010000000000000017.gz`.
Removed `minio/swapnil-backup/test/cluster-backup/wals/0000000100000000/000000010000000000000018.gz`.
```

**4. Delete the MinIO Bucket**

```
mc rb --force minio/swapnil-backup

Removed `minio/swapnil-backup` successfully.
```

**5. Confirm Bucket Deletion**

```
mc ls minio
# (No output if no buckets exist)
```

---

## Reference

- [Backup on object stores](https://cloudnative-pg.io/documentation/1.25/backup_barmanobjectstore/)
