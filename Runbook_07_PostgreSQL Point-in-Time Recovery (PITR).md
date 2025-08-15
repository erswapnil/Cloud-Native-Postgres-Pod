# Point-in-Time Recovery (PITR)

This runbook outlines the step-by-step process to perform **Point-in-Time Recovery (PITR)** of a PostgreSQL cluster managed via Kubernetes, leveraging backups stored in AWS S3 and the EnterpriseDB Postgres Kubernetes operator.

---

## **Table of Contents**

- [Pre-requisites](#pre-requisites)  
- [Test Case 1: PITR using targetTime](#test-case-1-pitr-using-targettime)  
  - [Step 1: Verify Cluster Status and Configuration](#step-1-verify-cluster-status-and-configuration)  
  - [Step 2: Verify WAL Archiving Status in S3](#step-2-verify-wal-archiving-status-in-s3)  
  - [Step 3: Create the Table](#step-3-create-the-table)  
  - [Step 4: Perform a Full Backup](#step-4-perform-a-full-backup)  
  - [Step 5: Execute Checkpoint and Record Target Time](#step-5-execute-checkpoint-and-record-target-time)  
  - [Step 6: Create Another Table After Target Time](#step-6-create-another-table-after-target-time)  
  - [Step 7: Create PITR YAML Using targetTime](#step-7-create-pitr-yaml-using-targettime)  
  - [Step 8: Validate Recovered Cluster](#step-8-validate-recovered-cluster)  
- [Test Case 2: PITR using targetLSN](#test-case-2-pitr-using-targetlsn)  
  - [Step 1: Current Tables](#step-1-current-tables)  
  - [Step 2: Capture WAL LSN for PITR](#step-2-capture-wal-lsn-for-pitr)  
  - [Step 3: Create Another Table](#step-3-create-another-table)  
  - [Step 4: Perform PITR to Target LSN](#step-4-perform-pitr-to-target-lsn)  
  - [Step 5: Validate Post-Recovery](#step-5-validate-post-recovery)  

---

## **Pre-requisites**
1. Kubernetes cluster with **EnterpriseDB Postgres Kubernetes operator** installed.  
2. **AWS CLI** configured with appropriate access keys and permissions.  
3. PostgreSQL cluster with **backup enabled** and **WAL archiving** to an S3 bucket.  
4. YAML configuration files for **backup** and **PITR** setup.  

---

# **Test Case 1: PITR using targetTime**

### **Step 1: Verify Cluster Status and Configuration**

```
kubectl get pods -L role

NAME               READY STATUS  RESTARTS AGE ROLE
cluster-backup-1   1/1   Running 0        85s primary
cluster-backup-2   1/1   Running 0        59s replica
```

```
kubectl cnp status cluster-backup

Cluster Summary
...
cluster-backup-2  0/4000060  Standby (async) OK     BestEffort 1.24.1          cnp-1.24.0-control-plane
```

### **Step 2: Verify WAL Archiving Status in S3**

```
aws s3 ls s3://swapnil-backup/ --recursive --human-readable --summarize

2025-01-16 18:28:31 0 Bytes test/
2025-01-16 18:35:33 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000001
2025-01-16 18:35:47 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000002
2025-01-16 18:36:02 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000003
2025-01-16 18:36:16 338 Bytes test/cluster-backup/wals/0000000100000000/000000010000000000000003.00000028.backup
Total Objects: 5
Total Size: 48.0 MiB
swapnilsuryawanshi@LAPTOP385PNIN cnp-yaml %
```

### **Step 3: Create the Table**

```
\dt+ rama

List of relations
Schema | Name | Type  | Owner    | Persistence | Access method | Size | Description
--------+------+-------+----------+-------------+---------------+------+
public | rama | table | postgres | permanent   | heap          | 38 MB |
(1 row)

select count(*) from rama;

count
---------
1110300
(1 row)
```

### **Step 4: Perform a Full Backup**

```
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Backup
metadata:
  name: pg-full
spec:
  cluster:
    name: cluster-backup
```

```
kubectl apply -f backup.yaml
```
```
kubectl get backup

NAME      AGE     CLUSTER         METHOD             PHASE       ERROR
pg-full   5m22s   cluster-backup  barmanObjectStore  completed
```

```
aws s3 ls s3://swapnil-backup/ --recursive --human-readable --summarize

2025-01-16 18:28:31 0 Bytes test/
2025-01-16 18:35:33 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000001
2025-01-16 18:35:47 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000002
2025-01-16 18:36:02 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000003
2025-01-16 18:36:16 338 Bytes test/cluster-backup/wals/0000000100000000/000000010000000000000003.00000028.backup
2025-01-16 18:40:48 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000004
2025-01-16 18:44:37 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000005
2025-01-16 18:44:50 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000006
2025-01-16 18:45:13 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000007
2025-01-16 18:45:26 16.0 MiB test/cluster-backup/wals/0000000100000000/000000010000000000000008
Total Objects: 10
Total Size: 128.0 MiB
```

### **Step 5: Execute Checkpoint and Record Target Time**

```
postgres=# checkpoint; select now();

CHECKPOINT
              now
-------------------------------
2025-01-16 13:25:32.159836+00
(1 row)
```

### **Step 6: Create Another Table After Target Time**

```
postgres=# create table swapnil(id int);

postgres=# insert into swapnil values(generate_series(1,1000));

postgres=# \dt+

List of relations
Schema | Name    | Type  | Owner    | Persistence | Access method | Size  | Description
--------+---------+-------+----------+-------------+---------------+-------+-------------
public | rama    | table | postgres | permanent   | heap          | 38 MB |
public | swapnil | table | postgres | permanent   | heap          | 64 kB |
(2 rows)
```

### **Step 7: Create PITR YAML Using targetTime**

`pitr.yaml`:

```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-pitr
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
  bootstrap:
    recovery:
      recoveryTarget:
        targetTime: '2025-01-16 13:25:32'
      backup:
        name: pg-full
  storage:
    resizeInUseVolumes: true
    size: 2Gi
  instances: 1
```

```
kubectl apply -f pitr.yaml
```

```
kubectl describe pod <pod_name> -n <namespace>

Events:
Type    Reason    Age From       Message
Normal  Scheduled 73s  default-scheduler  Successfully assigned default/cluster-pitr-1-full-recovery-xzw82 to cnp-1.24.0-control-plane
Normal  Pulled    74s  kubelet            Container image "docker.enterprisedb.com/k8s_enterprise/edb-postgres-for-kubernetes:1.24.1" already present on machine
Normal  Created   74s  kubelet            Created container: bootstrap-controller
Normal  Started   73s  kubelet            Started container bootstrap-controller
Normal  Pulled    73s  kubelet            Container image "quay.io/enterprisedb/postgresql:17.0" already present on machine
Normal  Created   73s  kubelet            Created container: full-recovery
Normal  Started   73s  kubelet            Started container: full-recovery
```

### **Step 8: Validate Recovered Cluster**

```
kubectl cnp psql cluster-pitr
```

```
postgres=#\dt+

List of relations
Schema | Name | Type  | Owner    | Persistence | Access method | Size | Description
--------+------+-------+----------+-------------+---------------+------+
public | rama | table | postgres | permanent   | heap          | 38 MB |
(1 row)
```

---

# **Test Case 2: PITR using targetLSN**

### **Step 1: Current Tables**

```
postgres=# \dt+

List of relations
Schema | Name    | Type  | Owner    | Persistence | Access method | Size | Description
--------+---------+-------+----------+-------------+---------------+------+
public | rama    | table | postgres | permanent   | heap          | 38 MB |
public | swapnil | table | postgres | permanent   | heap          | 64 kB |
(2 rows)
```

### **Step 2: Capture WAL LSN for PITR**

```
postgres=# SELECT pg_current_wal_lsn();

pg_current_wal_lsn
--------------------
0/D000200
(1 row)
```

### **Step 3: Create Another Table**

```
postgres=# create table t3 (id int);
postgres=# insert into t3 values(1);
postgres=# insert into t3 values(2);
postgres=# insert into t3 values(3);
postgres=# \dt+

List of relations
Schema | Name    | Type  | Owner    | Persistence | Access method | Size   | Description
--------+---------+-------+----------+-------------+---------------+--------+-------------
public | rama    | table | postgres | permanent   | heap          | 38 MB |
public | swapnil | table | postgres | permanent   | heap          | 64 kB |
public | t3      | table | postgres | permanent   | heap          | 8192 B |
(3 rows)
```

### **Step 4: Perform PITR to Target LSN**

`lsnpitr.yaml`:

```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-pitr-lsn
spec:
  bootstrap:
    recovery:
      recoveryTarget:
        targetLSN: "0/D000200"
        exclusive: true
      backup:
        name: pg-full
  instances: 1
  storage:
    resizeInUseVolumes: true
    size: 2Gi
```

```
kubectl apply -f lsnpitr.yaml
```

### **Step 5: Validate Post-Recovery**

```
postgres=# \dt+

List of relations
Schema | Name    | Type  | Owner    | Persistence | Access method | Size | Description
--------+---------+-------+----------+-------------+---------------+------+
public | rama    | table | postgres | permanent   | heap          | 38 MB |
public | swapnil | table | postgres | permanent   | heap          | 64 kB |
(2 rows)
```

---

## Reference

-[RecoveryTarget](https://cloudnative-pg.io/documentation/1.19/cloudnative-pg.v1/#postgresql-cnpg-io-v1-RecoveryTarget)


