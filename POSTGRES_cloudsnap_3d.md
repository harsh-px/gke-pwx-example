
# Postgres on GKE with PWX
_CloudSnap used 3DSnap directly to GCS_

1. create postgres
2. create data
3. setup cloudsnap and 3DSnapshot using `CHECKPOINT`
4. create cloudsnap that used 3DSnapshots to google object storage
5. restore postgres from cloudsnap backup

## Create Postgres

```
kubectl create -f  specs/postgres-px.yaml
storageclass.storage.k8s.io "px-postgres-sc" created
persistentvolumeclaim "postgres-data" created
configmap "example-config" created
deployment.extensions "postgres" created
```

## Add data to postgres
```
kubectl exec -it postgres-77bf94ccb5-ff856 /bin/bash
root@postgres-77bf94ccb5-ff856:/# psql -U postgres
psql (10.1)
Type "help" for help.

postgres=# 
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)
```

## Create database
```
postgres=# create database testdb;
CREATE DATABASE
postgres=# grant all privileges on database testdb to postgres;
GRANT
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/postgres         +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
```

## Insert Data
```
postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
CREATE TABLE

testdb=# INSERT INTO products VALUES (1, 'Cheese', 9.99);
INSERT 0 1
testdb=# SELECT * FROM products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
(1 row)
```

### Setup CloudSnaps and 3DSnapshot
#### Configure CloudSnaps

```
gcloud iam service-accounts create portworxsa \
    --display-name "portworx-sa"
```

```
gcloud projects add-iam-policy-binding portworx-eng \
    --member serviceAccount:portworxsa@portworx-eng.iam.gserviceaccount.com \
    --role roles/compute.admin \
    --role roles/storage.admin
```

```
gcloud iam service-accounts keys create files/gcs-key.json \
  --iam-account portworxsa@portworx-eng.iam.gserviceaccount.com
```

```
kubectl cp files/gcs-key.json kube-system/$PX_POD:/etc/pwx/gcs-key.json -n kube-system
```

```
pxctl credentials create --provider google --google-project-id portworx-eng --google-json-key-file /etc/pwx/gcs-key.json
Credentials created successfully, UUID:59d1f0ea-9322-446f-b4db-10c918664af9
```

### Taking a 3DSnap of Postgres

3D Snapshots allows `command`s to be run before and after a consistent snapshot is taken. These commands could be used for lock/unlock, flushing, archival purposed before and after snapshots are taken running them inside the PoDs which access the data. These are used by creating `Rule`s for a snapsho pre and post rule.

For Postgres, it may be a good idea to `CHECKPOINT` or run a archival cleanup
 - CHECKPOINT (https://www.postgresql.org/docs/10/static/sql-checkpoint.html)
 - Archival Cleanup (https://www.postgresql.org/docs/10/static/pgarchivecleanup.html)

 Example of a Rule for CHECKPOINT
 ```
apiVersion: stork.libopenstorage.org/v1alpha1
kind: Rule
metadata:
  name: postgres-3dsnap-prerule
spec:
  - podSelector:
      app: postgres
    actions:
    - type: command
      # https://www.postgresql.org/docs/10/static/sql-checkpoint.html
      # a point in the write-ahead log sequence at which all 
      # data files have been updated to reflect the information in the log.
      # All data files will be flushed to disk.
      value: psql -U postgres -c CHECKPOINT;
 ```

#### Create and View the Pre Snapshot Rule
```
kubectl create -f specs/postgres-3dsnap-prerule.yaml
rule.stork.libopenstorage.org/postgres-3dsnap-prerule created
```

```
kubectl get rule
NAME                      AGE
postgres-3dsnap-prerule   24s
```

```
kubectl describe rule postgres-3dsnap-prerule
Name:         postgres-3dsnap-prerule
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  stork.libopenstorage.org/v1alpha1
Kind:         Rule
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-10-15T17:16:44Z
  Resource Version:    46171
  Self Link:           /apis/stork.libopenstorage.org/v1alpha1/namespaces/default/rules/postgres-3dsnap-prerule
  UID:                 16e99fd2-d09e-11e8-9d4b-42010a8e00a9
Spec:
  Actions:
    Type:   command
    Value:  psql -U postgres -c CHECKPOINT;
  Pod Selector:
    App:  postgres
Events:   <none>
```


## Create CloudSnap with 3DSnapshots

Example cloudsnap with 3DSnapshot
```
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-cheese
  namespace: default
  annotations:
    stork.rule/pre-snapshot: postgres-3dsnap-prerule
    portworx/snapshot-type: cloud
spec:
  persistentVolumeClaimName: postgres-data
```

Create the cloudsnap
```
kubectl create -f specs/cloudsnap-3d-postgres.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "postgres-data" created
```

## View the CloudSnap

```
pxctl cloudsnap list
SOURCEVOLUME						SOURCEVOLUMEID			CLOUD-SNAP-ID										CREATED-TIME				TYPE		STATUS
pvc-9dd4e6b0-cd92-11e8-8281-42010a8e0115		394637582233985887		d5b722d7-679d-4cbb-8ade-188fbbf1e9a6/394637582233985887-759039737635622804		Thu, 11 Oct 2018 20:37:25 UTC		Manual		Done
```

## Make sure its done
```
pxctl cloudsnap status
SOURCEVOLUME		STATE		NODE		BYTES-PROCESSED	TIME-ELAPSED	COMPLETED
394637582233985887	Backup-Done	10.142.0.3	260491757	3.832537088s	Thu, 11 Oct 2018 20:37:28 UTC
```

# Now lets simulate failure and restore

## Fail (delete) postgres

```
$ kubectl get deployment
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
postgres          1         1         1            1           17h
wordpress         3         3         3            3           17h
wordpress-mysql   1         1         1            1           17h

$ kubectl delete deployment postgres
deployment.extensions "postgres" deleted

~/work/gke-pwx-example  master ✗                                                                                                                                           25m ⚑ ◒
$ kubectl get po
NAME                              READY     STATUS        RESTARTS   AGE
cassandra-0                       1/1       Running       0          35m
cassandra-1                       1/1       Running       0          35m
cassandra-2                       1/1       Running       0          34m
postgres-77bf94ccb5-ff856         1/1       Terminating   0          17h
wordpress-566c9858f6-fvl22        1/1       Running       0          17h
wordpress-566c9858f6-jjrzv        1/1       Running       0          17h
wordpress-566c9858f6-wx5m9        1/1       Running       0          17h
wordpress-mysql-f7df569c8-qmdz6   1/1       Running       0          17h

$ kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
cassandra-0                       1/1       Running   0          36m
cassandra-1                       1/1       Running   0          35m
cassandra-2                       1/1       Running   0          35m
wordpress-566c9858f6-fvl22        1/1       Running   0          17h
wordpress-566c9858f6-jjrzv        1/1       Running   0          17h
wordpress-566c9858f6-wx5m9        1/1       Running   0          17h
wordpress-mysql-f7df569c8-qmdz6   1/1       Running   0          17h
```

## Create a PVC from the cloudsnap

> Note that we are using the `stork-snapshot-sc` StorageClass to allow stork to instruct cloning our cloudsnap

## Create postgres from cloudsnap with 3DSnapshot

Create postgres
```
kubectl create -f specs/postgres-from-cloudsnap-3d.yaml
persistentvolumeclaim "postgres-snapshot-cheese-clone" created
deployment.extensions "postgres" created
```

Wait for it to complete
```
$ kubectl get pvc
NAME                             STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               AGE
postgres-snapshot-cheese-clone   Pending                                                                        stork-snapshot-sc          8s

$ kubectl get pvc
NAME                             STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               AGE
postgres-snapshot-cheese-clone   Bound     pvc-59d5fd41-ce25-11e8-8281-42010a8e0115   2Gi        RWO            stork-snapshot-sc          1m
```

View our pvc with `pxctl`
```
pxctl volume list | grep pvc-59d5fd41-ce25-11e8-8281-42010a8e0115
971865402596412981	pvc-59d5fd41-ce25-11e8-8281-42010a8e0115				5 GiB	1	no	no		LOW		up - attached on 10.142.0.2	Up
```

Update replication from 1 to 2
```
pxctl volume ha-update 971865402596412981 --repl 2
Update Volume Replication: Replication update started successfully for volume 971865402596412981
```

List Postgres
```
kubectl get po | grep postgres
postgres-95c4d46d5-4sqb9          1/1       Running   0          2m
```

## Verify the data is restored from the cloud backup snapshot
```
kubectl exec -it postgres-95c4d46d5-4sqb9 /bin/bash
root@postgres-95c4d46d5-4sqb9:/# psql -U postgres
psql (10.1)
Type "help" for help.

postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
testdb=# SELECT * FROM products;
 product_no |  name  | price
------------+--------+-------
          1 | Cheese |  9.99
(1 row)
```

## Perf Postgress on PWX 

> using `"type=pd-ssd,size=50"` on `--machine-type=n1-standard-4` and `postgres:10.1` image.


### Read-Write Test

```
pgbench -i -s 70 postgres
pgbench -c 4 -j 2 -T 600 postgres
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 70
query mode: simple
number of clients: 4
number of threads: 2
duration: 600 s
number of transactions actually processed: 36799
latency average = 65.228 ms
tps = 61.323766 (including connections establishing)
tps = 61.324093 (excluding connections establishing)
```
### Read-Only Test

```
pgbench -i -s 70 postgres
pgbench -c 4 -j 2 -T 600 -S postgres
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 70
query mode: simple
number of clients: 4
number of threads: 2
duration: 600 s
number of transactions actually processed: 15115559
latency average = 0.159 ms
tps = 25192.234667 (including connections establishing)
tps = 25192.390429 (excluding connections establishing)
```

### Simple Write Test

```
pgbench -i -s 70 postgres
pgbench -c 4 -j 2 -T 600 -N postgres
starting vacuum...end.
transaction type: <builtin: simple update>
scaling factor: 70
query mode: simple
number of clients: 4
number of threads: 2
duration: 600 s
number of transactions actually processed: 43862
latency average = 54.720 ms
tps = 73.098831 (including connections establishing)
tps = 73.099317 (excluding connections establishing)
```



### IO profiles
- https://docs.portworx.com/maintain/performance/tuning.html

```
"DB"
This implements a write-back flush coalescing algorithm. This algorithm attempts to coalesce multiple syncs that occur within a 50ms window into a single sync. Coalesced syncs are acknowledged only after copying to all replicas. In order to do this, the algorithm requires a minimum replication (HA factor) of 3. This mode assumes all replicas do not fail (kernel panic or power loss) simultaneously in a 50 ms window. Use io_profile=db.

Note:
If there are not enough nodes online, PX will automatically disable this algorithm.
```
