# Configuring Synchronous Streaming Replication

The default replication method in PostgreSQL is **asynchronous**.  
This runbook provides step-by-step instructions for configuring and verifying **synchronous streaming replication** in a CNPG PostgreSQL cluster.

We will configure:
- **Method:** `any` (quorum-based synchronous replication)
- **Number:** `1` (transactions must wait for acknowledgement from at least one standby)

---

## Step 1: Create a YAML for the CNPG cluster with synchronous replication

We define a **3-instance PostgreSQL cluster** with synchronous replication enabled.

```
cat <<EOF | sudo tee cluster-example-SYNC.yaml
apiVersion: postgresql.k8s.enterprisedb.io/v1
kind: Cluster
metadata:
  name: cluster-example
spec:
  instances: 3

  storage:
    size: 1G

  postgresql:
    synchronous:
      method: any
      number: 1
EOF
```

---

## Step 2: Apply the cluster manifest

```
kubectl apply -f cluster-example-SYNC.yaml
```

---

## Step 3: Check the status of the pods

```
kubectl get pods -L role
NAME                READY   STATUS    RESTARTS   AGE     ROLE
cluster-example-1   1/1     Running   0          153m    primary
cluster-example-2   1/1     Running   0          153m    replica
cluster-example-3   1/1     Running   0          152m    replica
```

---

## Step 4: Check the cluster status

```
kubectl cnp status cluster-example
Cluster Summary
  Name: default/cluster-example
  System ID: 7452311956268736536
  PostgreSQL Image: quay.io/enterprisedb/postgresql:17.0
  Primary instance: cluster-example-1
  Primary start time: 2024-12-25 11:36:35 +0000 UTC (uptime 2h33m52s)
  Status: Cluster in healthy state

Instances: 3
Ready instances: 3
Size: 142M
Current Write LSN: 0/7000060 (Timeline: 1 - WAL File: 000000010000000000000007)

Streaming Replication status
Replication Slots Enabled
Name               Sent LSN   Write LSN  Flush LSN  Replay LSN  Write Lag  Flush Lag  Replay Lag  State     Sync State  Sync Priority  Replication Slot
----               --------   ---------  ---------  ----------  ---------  ---------  ----------  -----     ----------  -------------  ----------------
cluster-example-2  0/7000060  0/7000060  0/7000060  0/7000060   00:00:00   00:00:00   00:00:00    streaming quorum       1             active
cluster-example-3  0/7000060  0/7000060  0/7000060  0/7000060   00:00:00   00:00:00   00:00:00    streaming quorum       1             active

Instances status
Name               Current LSN  Replication role  Status  QoS         Manager Version  Node
----               -----------  ----------------  ------  ---         ---------------  ----
cluster-example-1  0/7000060    Primary            OK      BestEffort  1.24.1           cnp-1.24.1-control-plane
cluster-example-2  0/7000060    Standby (sync)     OK      BestEffort  1.24.1           cnp-1.24.1-control-plane
cluster-example-3  0/7000060    Standby (sync)     OK      BestEffort  1.24.1           cnp-1.24.1-control-plane
```
 
Both replicas are in **sync** state, meaning the primary will wait for confirmation from at least one standby before committing a transaction.

---

## Step 5: Log in to the database and check synchronous replication configuration

```
kubectl cnp psql cluster-example
SHOW synchronous_standby_names;
synchronous_standby_names
--------------------------------------------------
ANY 1 ("cluster-example-2","cluster-example-3","cluster-example-1")
(1 row)
```

- **ANY 1** means quorum-based replication — any one of the listed nodes must acknowledge before commit.  
- All three instances are eligible synchronous standbys.

---

## Reference

- [Streaming Replication is now configured and verified successfully](https://cloudnative-pg.io/documentation/1.25/replication/)
