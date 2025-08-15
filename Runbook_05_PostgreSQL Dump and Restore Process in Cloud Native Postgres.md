
# PostgreSQL Dump and Restores

This runbook documents the step-by-step procedure for performing a PostgreSQL database dump and restore process in a Cloud Native Postgres environment.

---

## Step 1: Deploy the New Cluster or Use the Existing Cluster

```
% cat <<EOF | sudo tee cluster-example.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example
spec:
  instances: 3

  storage:
    size: 1Gi
EOF
````

````
% kubectl apply -f cluster-example.yaml
cluster.postgresql.cnpg.io/cluster-example created
````

````
% kubectl get pods -L role
NAME                 READY   STATUS    RESTARTS   AGE   ROLE
cluster-example-1    1/1     Running   0          50s   primary
cluster-example-2    1/1     Running   0          30s   replica
cluster-example-3    1/1     Running   0          12s   replica
````

---

## Step 2: Check the Status of the Database Cluster

```
% kubectl cnpg status cluster-example
Cluster Summary
  Name:           default/cluster-example
  System ID:      7459293290899460124
  PostgreSQL Image: ghcr.io/cloudnative-pg/postgresql:17.2

  Primary instance:   cluster-example-1
  Primary start time: 2025-01-13 07:07:44 +0000 UTC (uptime 42m23s)
  Status:             Cluster in healthy state
  Instances:          3
  Ready instances:    3
  Size:               142M
  Current Write LSN:  0/8000000 (Timeline: 1 - WAL File: 000000010000000000000008)

Continuous Backup status
  Not configured

Streaming Replication status
Replication Slots Enabled
Name               Sent LSN    Write LSN   Flush LSN   Replay LSN  Write Lag  Flush Lag  Replay Lag  State     Sync State  Sync Priority  Replication Slot
----               --------    ---------   ---------   ----------  ---------  ---------  ----------  --------  ----------  -------------  ----------------
cluster-example-2  0/8000000   0/8000000   0/8000000   0/8000000    00:00:00   00:00:00   00:00:00    streaming async       0              active
cluster-example-3  0/8000000   0/8000000   0/8000000   0/8000000    00:00:00   00:00:00   00:00:00    streaming async       0              active

Instances status
Name               Current LSN  Replication role   Status  QoS Manager  Version   Node
----               -----------  ----------------  ------  -----------  -------   ----
cluster-example-1  0/8000000    Primary            OK      BestEffort   1.25.0    cnpg-1.25.0-control-plane
cluster-example-2  0/8000000    Standby (async)    OK      BestEffort   1.25.0    cnpg-1.25.0-control-plane
cluster-example-3  0/8000000    Standby (async)    OK      BestEffort   1.25.0    cnpg-1.25.0-control-plane
```

---

## Step 3: Log in to the Primary Database Pod and Create a Database and Table

```
% kubectl exec -it cluster-example-1 -- /bin/bash
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)

postgres@cluster-example-1:/$ psql
psql (17.2 (Debian 17.2-1.pgdg110+1))
Type "help" for help.

postgres=# create database swapnil;
CREATE DATABASE

postgres=# \c swapnil postgres
You are now connected to database "swapnil" as user "postgres".

swapnil=# create table swapnil (id int);
CREATE TABLE

swapnil=# insert into swapnil values (generate_series(1,1000));
INSERT 0 1000

swapnil=# \dt+
                      List of relations
 Schema |  Name   | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+---------+-------+----------+-------------+---------------+-------+-------------
 public | swapnil | table | postgres | permanent   | heap          | 64 kB |
(1 row)

swapnil=# select count(*) from swapnil;
 count
-------
  1000
(1 row)
```

---

## Step 4: Take a Backup of the Database

```
% kubectl exec cluster-example-1 -c postgres -- pg_dump -Fc -d swapnil -C > swapnil.dump
```

```
% ls -l swapnil.dump
-rw-r--r--  1 swapnilsuryawanshi  staff  3131 Feb  1 18:36 swapnil.dump
```

---

## Step 5: Drop the Database

```
% kubectl exec -it cluster-example-1 -- /bin/bash
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)

postgres@cluster-example-1:/$ psql
psql (17.2 (Debian 17.2-1.pgdg110+1))
Type "help" for help.

postgres=# drop database swapnil;
DROP DATABASE

postgres=# vacuum analyze;
VACUUM

postgres=# \q
postgres@cluster-example-1:/$ exit
exit
```

---

## Step 6: Restore the Database

```
% kubectl exec -i cluster-example-1 -c postgres -- pg_restore -d app --verbose --create < swapnil.dump
pg_restore: connecting to database for restore
pg_restore: creating DATABASE "swapnil"
pg_restore: connecting to new database "swapnil"
pg_restore: creating TABLE "public.swapnil"
pg_restore: processing data for table "public.swapnil"
```

---

## Step 7: Verify the Restore

```
% kubectl exec -it cluster-example-1 -- /bin/bash
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)

postgres@cluster-example-1:/$ psql
psql (17.2 (Debian 17.2-1.pgdg110+1))
Type "help" for help.

postgres=# \c swapnil
You are now connected to database "swapnil" as user "postgres".

swapnil=# \dt+
                      List of relations
 Schema |  Name   | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+---------+-------+----------+-------------+---------------+-------+-------------
 public | swapnil | table | postgres | permanent   | heap          | 64 kB |
(1 row)

swapnil=# select count(*) from swapnil;
 count
-------
  1000
(1 row)
```

**Expected Result:** The database should contain the restored table with 1000 records.

---

## Reference

- [Emergency backup](https://cloudnative-pg.io/documentation/1.25/troubleshooting/#emergency-backup)
