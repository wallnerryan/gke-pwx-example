
# Postgres on GKE with PWX

1. create postgres
2. create data
3. setup cloudsnap
4. create cloudsnap to google object storage
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

## Setup CloudSnaps

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

## Create cloudsnap 

Example cloudsnap
```
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-cheese
  namespace: default
  annotations:
    portworx/snapshot-type: cloud
spec:
  persistentVolumeClaimName: postgres-data
```

Create the cloudsnap
```
kubectl create -f specs/cloudsnap-postgres.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "postgres-data" created
```

## View Snapshot

```
pxctl volume list -s
ID			NAME												SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	STATUS		HA-STATE
759039737635622804	pvc-9dd4e6b0-cd92-11e8-8281-42010a8e0115_394637582233985887_clmanual_2018-10-11T20-37-21	5 GiB	1	no	no		LOW		none - detached	Detached
```

or via cloudsnap

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

Create a PVC from the cloudsnap `postgres-snapshot-cheese` we took before

> Note that we are using the `stork-snapshot-sc` StorageClass to allow stork to instruct cloning our cloudsnap

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-snapshot-cheese-clone
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: postgres-snapshot-cheese
spec:
  accessModes:
     - ReadWriteOnce
  storageClassName: stork-snapshot-sc
  resources:
    requests:
      storage: 2Gi
```

And in your Postgres Spec make sure and use it from the volume
```
volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-snapshot-cheese-clone
```

## Create postgres from cloudsnap

Create postgres
```
kubectl create -f specs/postgres-from-cloudsnap.yaml
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