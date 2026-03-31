This runbook outlines the process for enabling detailed PostgreSQL logging, installing the **pgBadger** analysis tool, and generating performance reports within a Kubernetes or OpenShift environment.

---

## 1. Configure PostgreSQL Logging
To get meaningful insights from pgBadger, the PostgreSQL instance must be configured to log specific details, such as durations, checkpoints, and connections.

### Update Cluster Configuration
Modify the PostgreSQL Custom Resource (CR) to include the required logging parameters.

```

      log_min_duration_statement: "0"
      log_line_prefix: "%t [%p]: user=%u,db=%d,app=%a,client=%h"
      log_checkpoints: "on"
      log_connections: "on"
      log_disconnections: "on"
      log_lock_waits: "on"
      log_temp_files: "0"
      log_autovacuum_min_duration: "0"
      log_error_verbosity: "default"
      lc_messages: "en_US.UTF-8"
```

```      
swapnilsuryawanshi@MAC-CR9L20YFN6 plugin % kubectl edit cluster postgresql-advanced-cluster
cluster.postgresql.k8s.enterprisedb.io/postgresql-advanced-cluster edited
```

### Verify Running Configuration
Connect to the database via the CloudNativePG (CNP) plugin to confirm that the changes have been applied to the runtime environment.

```
swapnilsuryawanshi@MAC-CR9L20YFN6 plugin % kubectl cnp psql postgresql-advanced-cluster    
psql (18.3.0)
Type "help" for help.

postgres=# show log_min_duration_statement;
 log_min_duration_statement 
----------------------------
 0
(1 row)

postgres=# show log_line_prefix;
               log_line_prefix             
-----------------------------------------
 %t [%p]: user=%u,db=%d,app=%a,client=%h
(1 row)

postgres=# show log_checkpoints;
 log_checkpoints 
-----------------
 on
(1 row)

postgres=# show log_connections;
 log_connections 
-----------------
 on
(1 row)

postgres=# show log_disconnections;
 log_disconnections 
--------------------
 on
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits 
----------------
 on
(1 row)

postgres=# show log_temp_files;
 log_temp_files 
----------------
 0
(1 row)

postgres=# show log_autovacuum_min_duration;
 log_autovacuum_min_duration 
-----------------------------
 0
(1 row)

postgres=# show log_error_verbosity;
 log_error_verbosity 
---------------------
 default
(1 row)

postgres=# show lc_messages;
 lc_messages 
-------------
 en_US.UTF-8
(1 row)
```

---

## 2. Install pgBadger
Download and compile pgBadger on your local machine or an analysis server.

### Clone and Build
```
swapnilsuryawanshi@MAC-CR9L20YFN6 59686-pgbadger % git clone https://github.com/darold/pgbadger.git
Cloning into 'pgbadger'...
...
Resolving deltas: 100% (3257/3257), done.
```

```
swapnilsuryawanshi@MAC-CR9L20YFN6 59686-pgbadger % cd pgbadger
```

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % perl Makefile.PL
Checking if your kit is complete...
Looks good
Generating a Unix-style Makefile
Writing Makefile for pgBadger
```

### Install to System
Use `make install` to move the pgbadger binary to your local path.

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % make && sudo make install
...
Installing /usr/local/bin/pgbadger
Appending installation info to /Library/Perl/Updates/5.34.1/darwin-thread-multi-2level/perllocal.pod
```

---

## 3. Extract Logs from Kubernetes
Identify the primary database pod and redirect the logs to a local file for analysis.

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % kubectl get pods -L role
NAME                            READY   STATUS    RESTARTS   AGE   ROLE
postgresql-advanced-cluster-1   2/2     Running   0          26h   primary
postgresql-advanced-cluster-2   2/2     Running   0          26h   replica
postgresql-advanced-cluster-3   2/2     Running   0          26h   replica
```

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % kubectl logs postgresql-advanced-cluster-1 > postgresql.log
Defaulted container "postgres" out of: postgres, bootstrap-controller (init), plugin-barman-cloud (init)
```

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % ls -l postgresql.log
-rw-r--r--  1 swapnilsuryawanshi  staff  1158967 31 Mar 19:17 postgresql.log
```

---

## 4. Generate Reports
Analyze the logs using pgBadger based on the format output by the cluster.

### For JSON Format Logs
If the logs are in JSON format, use the `-f jsonlog` flag:
```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % pgbadger -f jsonlog postgresql.log -o postgresql_report.html
[========================>] Parsed 1158967 bytes of 1158967 (100.00%), queries: 127, events: 16
LOG: Ok, generating html report...
```

### For Standard Text Logs
Use the `--prefix` flag to match the `log_line_prefix` configured in Step 1:
```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % pgbadger --prefix '%t [%p]: user=%u,db=%d,app=%a,client=%h' postgresql.log -o postgresql_report1.html
[========================>] Parsed 1158967 bytes of 1158967 (100.00%), queries: 127, events: 16
LOG: Ok, generating html report...
```

---

## 5. Reviewing the Results
To open the HTML report, look exactly where you have stored the files (e.g., the `pgbadger` directory). Click on the generated reports, such as **postgresql_report.html** or **postgresql_report1.html**, and they will open in your default web browser for interactive viewing.

```
swapnilsuryawanshi@MAC-CR9L20YFN6 pgbadger % ls -l *.html
-rw-r--r--  1 swapnilsuryawanshi  staff  1614195 31 Mar 19:19 postgresql_report.html
-rw-r--r--  1 swapnilsuryawanshi  staff  1614189 31 Mar 19:20 postgresql_report1.html
```