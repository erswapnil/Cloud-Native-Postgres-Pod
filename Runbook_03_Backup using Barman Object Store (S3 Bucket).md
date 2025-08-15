# Backup using Barman Object Store (S3 Bucket)

## Prerequisites Before Backup

To take a backup of any cluster, first, you have to set up the storage for the cluster so that we can archive the backup files and WAL files.

1. Request access to Amazon AWS.
2. Create an IAM role, and generate **ACCESS_KEY_ID** and **ACCESS_SECRET_KEY**.  
   - **ACCESS_KEY_ID**: the ID of the access key that will be used to upload files into S3  
   - **ACCESS_SECRET_KEY**: the secret part of the access key mentioned above
3. Create a bucket and provide it access to external apps. You need the below complete permissions:  
   - `s3:AbortMultipartUpload`  
   - `s3:DeleteObject`  
   - `s3:GetObject`  
   - `s3:ListBucket`  
   - `s3:PutObject`  
   - `s3:PutObjectTagging`
4. Verify access from external sources.

> **Note:** The steps above will vary according to the storage you are using. Here, we are showing an example of an AWS S3 bucket to store the WAL files and backups.

---

## Step 1: Create a Namespace for the Backup Cluster

```
kubectl create namespace backup
namespace/backup created
```

## Step 2: Create an AWS Secret in the Backup Namespace

```
kubectl create secret generic aws-creds --from-literal=ACCESS_KEY_ID=xxxxxxxxx7BO57CN3Gxxxxx --from-literal=ACCESS_SECRET_KEY=xxxxxxxxhMMBTW7sF234utAjVGrS+xxxxxxxx -n backup
secret/aws-creds created
```

## Step 3: Verify the Secret Creation

```
kubectl get secret -n backup
NAME        TYPE     DATA   AGE
aws-creds   Opaque   2      10s
```

## Step 4: Create Cluster Manifest for Backup

```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-backup
  namespace: backup
spec:
  logLevel: info
  startDelay: 30
  stopDelay: 30
  nodeMaintenanceWindow:
    inProgress: false
  reusePVC: true
  backup:
    barmanObjectStore:
      destinationPath: 's3://swapnil-backup/test/'
      s3Credentials:
        accessKeyId:
          key: ACCESS_KEY_ID
          name: aws-creds
        inheritFromIAMRole: false
        secretAccessKey:
          key: ACCESS_SECRET_KEY
          name: aws-creds
  enableSuperuserAccess: true
  monitoring:
    disableDefaultQueries: false
    enablePodMonitor: false
  minSyncReplicas: 0
  postgresGID: 26
  replicationSlots:
    highAvailability:
      slotPrefix: _cnp_
      updateInterval: 30
  primaryUpdateMethod: switchover
  postgresUID: 26
  walStorage:
    resizeInUseVolumes: true
    size: 1Gi
  maxSyncReplicas: 0
  switchoverDelay: 40000000
  storage:
    resizeInUseVolumes: true
    size: 2Gi
  primaryUpdateStrategy: unsupervised
  instances: 2
  imagePullPolicy: Always
```

## Step 5: Apply the Cluster Manifest

```
kubectl apply -f backupcluster.yaml
cluster.postgresql.cnpg.io/cluster-backup created
```

## Step 6: Verify Pod Status

```
kubectl get pods -n backup -L pod
NAME              READY   STATUS    RESTARTS   AGE
cluster-backup-1  1/1     Running   0          2m16s
cluster-backup-2  1/1     Running   0          109s
```

## Step 7: Verify S3 Bucket for Backup Folder

```
aws s3 ls s3://swapnil-backup/test/ --human-readable --summarize

PRE cluster-backup/
2025-01-16 18:28:31    0 Bytes
Total Objects: 1
Total Size: 0 Bytes
```

## Step 8: Verify Cluster Status and WAL Archiving

```
kubectl cnpg status cluster-backup -n backup
Cluster Summary
Name backup/cluster-backup
System ID: 7463110619807920157
PostgreSQL Image: ghcr.io/cloudnative-pg/postgresql:17.2
Primary instance: cluster-backup-1
Primary start time: 2025-01-23 14:00:59 +0000 UTC (uptime 29s)

Status: Cluster in healthy state
Instances: 1
Ready instances: 1
Size: 62M
Current Write LSN: 0/20114B8 (Timeline: 1 - WAL File:
000000010000000000000002)
Continuous Backup status
First Point of Recoverability: Not Available
Working WAL archiving: OK
WALs waiting to be archived: 0
Last Archived WAL: 000000010000000000000001 @
2025-01-23T14:01:25.58927Z
Last Failed WAL: -
```

## Step 9: Take a Backup Using CNPG Plugin

```
kubectl cnpg backup cluster-backup -n backup
backup/cluster-backup-20250123193330 created
```

## Step 10: Verify Backup Status

```
kubectl get backup -n backup
NAME                                  AGE   CLUSTER         METHOD             PHASE     ERROR
cluster-backup-20250123193330         10s   cluster-backup  barmanObjectStore  running
```

## Step 11: Verify Backup Completion

```
kubectl get backup -n backup
NAME                                  AGE    CLUSTER         METHOD             PHASE       ERROR
cluster-backup-20250123193330         105s   cluster-backup  barmanObjectStore  completed
```

## Step 12: Verify Backup in S3 Bucket

```
aws s3 ls s3://swapnil-backup/test/cluster-backup/ --human-readable --summarize --recursive

2025-01-23 19:34:18  1.4 KiB  test/cluster-backup/base/20250123T140331/backup.info
2025-01-23 19:33:39  31.2 MiB test/cluster-backup/base/20250123T140331/data.tar
2025-01-23 19:31:04  16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000001
2025-01-23 19:33:34  16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000002
2025-01-23 19:33:51  16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000003
2025-01-23 19:33:51  348  B   test/cluster-backup/wals/0000000100000000/000000010000000000000003.00000028.backup
2025-01-23 19:34:05  16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000004

Total Objects: 7
Total Size: 95.2 MiB
```

## Step 13: Verify Cluster Status After Backup

```
kubectl cnpg status cluster-backup -n backup
Cluster Summary
Name backup/cluster-backup
System ID: 7463110619807920157
PostgreSQL Image: ghcr.io/cloudnative-pg/postgresql:17.2
Primary instance: cluster-backup-1
Primary start time: 2025-01-23 14:00:59 +0000 UTC (uptime 6m36s)
Status: Cluster in healthy state
Instances: 1
Ready instances: 1
Size: 110M
Current Write LSN: 0/50000C8 (Timeline: 1 - WAL File:
000000010000000000000005)
Continuous Backup status
First Point of Recoverability: 2025-01-23T14:03:39Z
Working WAL archiving: OK
WALs waiting to be archived: 0
Last Archived WAL: 000000010000000000000004 @ 2025-01-23T14:04:24.479506Z
Last Failed WAL:
```

## Step 14: Take a Backup Using Backup Manifest (Method 2)

```
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pg-full
spec:
  cluster:
    name: cluster-backup
```

```
kubectl apply -f backup.yaml -n backup
backup.postgresql.cnpg.io/pg-full created
```

## Step 15: Verify Backup Status (Method 2)

```
kubectl get backup -n backup
NAME                                  AGE    CLUSTER         METHOD             PHASE       ERROR
cluster-backup-20250123193330         10m    cluster-backup  barmanObjectStore  completed
pg-full                               2m4s   cluster-backup  barmanObjectStore  completed
```

## Step 16: Verify Backup Completion in S3 Bucket (Method 2)

```
aws s3 ls s3://swapnil-backup/test/cluster-backup/ --human-readable --summarize --recursive

2025-01-23 19:34:18  1.4 KiB test/cluster-backup/base/20250123T140331/backup.info
2025-01-23 19:33:39  31.2 MiB test/cluster-backup/base/20250123T140331/data.tar
2025-01-23 19:42:46  1.4 KiB test/cluster-backup/base/20250123T141213/backup.info
2025-01-23 19:42:14  31.2 MiB test/cluster-backup/base/20250123T141213/data.tar
2025-01-23 19:31:04  16.0 MiB ...
2025-01-23 19:42:18  348  B   test/cluster-backup/wals/0000000100000000/000000010000000000000006.00000028.backup
2025-01-23 19:42:58  16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000007

Total Objects: 13
Total Size: 174.3 MiB
```

## Step 17: Verify Final Cluster Status

```
kubectl cnpg status cluster-backup -n backup
Cluster Summary
Name backup/cluster-backup
System ID: 7463110619807920157
PostgreSQL Image: ghcr.io/cloudnative-pg/postgresql:17.2
Primary instance: cluster-backup-1
Primary start time: 2025-01-23 14:00:59 +0000 UTC (uptime 14m46s)
Status: Cluster in healthy state
Instances: 1
Ready instances: 1
Size: 158M
Current Write LSN: 0/80000C8 (Timeline: 1 - WAL File:
000000010000000000000008)
Continuous Backup status
First Point of Recoverability: 2025-01-23T14:03:39Z
Working WAL archiving: OK
WALs waiting to be archived: 0
Last Archived WAL: 000000010000000000000007 @ 2025-01-23T14:13:09.877353Z
Last Failed WAL:
```

---

## Reference

- [Backup](https://cloudnative-pg.io/documentation/1.25/backup/)
