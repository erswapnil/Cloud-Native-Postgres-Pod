# Full Backup Recovery

## Prerequisites

- Existing PostgreSQL cluster: `cluster-backup` (created in runbook-3)
- Backup name: `pg-full` (created in runbook-3)
- AWS S3 bucket containing the backup (created in runbook-3)

---

## Step 1: Verify Backup Status
```
kubectl get backup -n backup

NAME      AGE      CLUSTER         METHOD
PHASE     ERROR
pg-full   2m4s     cluster-backup  barmanObjectStore
completed
```
---

## Step 2: Verify Backup in S3 Bucket

```
aws s3 ls s3://swapnil-backup/test/cluster-backup/ --human-readable --summarize --recursive

2025-01-23 19:34:18 1.4 KiB test/cluster-backup/base/20250123T140331/backup.info
2025-01-23 19:33:39 31.2 MiB test/cluster-backup/base/20250123T140331/data.tar
...
Total Objects: 13
Total Size: 174.3 MiB
```
---

## Step 3: Create the Restore Manifest

```
vi cluster-restore-full.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-restore-full
  namespace: backup
spec:
  logLevel: info
  startDelay: 30
  stopDelay: 30
  nodeMaintenanceWindow:
    inProgress: false
  reusePVC: true
  backup:
    target: primary
    enableSuperuserAccess: true
  monitoring:
    disableDefaultQueries: false
    enablePodMonitor: false
  minSyncReplicas: 0
  postgresGID: 26
  replicationSlots: []
  highAvailability:
    slotPrefix: _cnp_
    updateInterval: 30
    primaryUpdateMethod: switchover
  bootstrap:
    recovery:
      backup:
        name: pg-full
      failoverDelay: 0
  postgresUID: 26
  walStorage:
    resizeInUseVolumes: true
    size: 1G
  maxSyncReplicas: 0
  switchoverDelay: 40000000
  storage:
    resizeInUseVolumes: true
    size: 2Gi
  primaryUpdateStrategy: unsupervised
  instances: 1
```

*This manifest defines a new cluster `cluster-restore-full` restored from the `pg-full` backup.*

---

## Step 4: Apply the Restore Manifest

```
kubectl apply -f cluster-restore-full.yaml -n backup
```
---

## Step 5: Verify Pod Status

```
kubectl get pod -n backup -L role

NAME                       READY  STATUS   RESTARTS  AGE      ROLE
cluster-backup-1           1/1    Running  0         32m      primary
cluster-restore-full-1     1/1    Running  0         5m3s     primary
```

---

## Step 6: Verify Cluster Status

```
kubectl cnpg status cluster-restore-full -n backup

Cluster Summary
Name backup/cluster-restore-full
System ID: 7463110619807920157

PostgreSQL Image: ghcr.io/cloudnative-pg/postgresql:17.2

Primary instance: cluster-restore-full-1
Primary start time: 2025-01-23 14:27:53 +0000 UTC (uptime 56s)
Status: Cluster in healthy state
Instances: 1
Ready instances: 1
Size: 126M
Current Write LSN: 0/B0084F0 (Timeline: 2 - WAL File: 00000002000000000000000B)

Continuous Backup status
First Point of Recoverability: Not Available

Working WAL archiving: OK
WALs waiting to be archived: 0
Last Archived WAL: 00000002000000000000000A @ 2025-01-23T14:27:53.624223Z
Last Failed WAL: 00000002.history @ 2025-01-23T14:27:45.234394Z

Streaming Replication status
Not configured

Instances status
Name                     Current LSN   Replication role  Status  QoS   Manager Version  Node
------------------------ ------------ ----------------  ------ ----  ----------------  ----
cluster-restore-full-1   0/B0084F0     Primary           OK      BestEffort 1.25.0  cnpg-1.25.0-control-plane
```

---

## Reference

- [Recovery](https://cloudnative-pg.io/documentation/1.25/recovery/)



