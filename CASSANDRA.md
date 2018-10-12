# Cassandra on GKE with PWX

1. create cassandra
2. create some data by running perf
3. example snapgroup directly.
4. create 3dsnap (creates snapgroup indirectly)
  - Included rules to flushe the tables from the memtable on all cassandra pods.
5. backup via above 3dsnap
6. restore cassandra from 3dsnap clones

## Create

```
kubectl create -f specs/cassandra-pwx.yaml
service/cassandra created
storageclass.storage.k8s.io/px-storageclass created
statefulset.apps/cassandra created
```

## Create data
```
cqlsh> CREATE KEYSPACE newkeyspace WITH replication = {'class':'SimpleStrategy', 'replication_factor' : 3};
cqlsh> DESCRIBE keyspaces;

system_auth  system              system_traces
system_schema   newkeyspace  system_distributed

cqlsh:newkeyspace> CREATE TABLE emp(
               ...    emp_id int PRIMARY KEY,
               ...    emp_name text,
               ...    emp_city text,
               ...    emp_sal varint,
               ...    emp_phone varint
               ...    );
cqlsh:newkeyspace> select * from emp;

 emp_id | emp_city | emp_name | emp_phone | emp_sal
--------+----------+----------+-----------+---------

(0 rows)
cqlsh:newkeyspace> INSERT INTO emp (emp_id, emp_name, emp_city,
               ...    emp_phone, emp_sal) VALUES(1,'ram', 'Hyderabad', 9848022338, 50000);
cqlsh:newkeyspace> select * from emp;

 emp_id | emp_city  | emp_name | emp_phone  | emp_sal
--------+-----------+----------+------------+---------
      1 | Hyderabad |      ram | 9848022338 |   50000

(1 rows)
```

## Perf

```
kubectl exec -it cassandra-0 /bin/bash
root@cassandra-0:/#
```

### Write
```
 ./cassandra-stress write n=1000000 -rate threads=50 -graph file=ocs-write-1000000-benchmark.html title=ocs-write-1000000 revision=benchmark-0

 Results:
Op rate                   :    3,606 op/s  [WRITE: 3,606 op/s]
Partition rate            :    3,606 pk/s  [WRITE: 3,606 pk/s]
Row rate                  :    3,606 row/s [WRITE: 3,606 row/s]
Latency mean              :   13.8 ms [WRITE: 13.8 ms]
Latency median            :    1.6 ms [WRITE: 1.6 ms]
Latency 95th percentile   :   84.5 ms [WRITE: 84.5 ms]
Latency 99th percentile   :  100.6 ms [WRITE: 100.6 ms]
Latency 99.9th percentile :  219.9 ms [WRITE: 219.9 ms]
Latency max               :  495.2 ms [WRITE: 495.2 ms]
Total partitions          :  1,000,000 [WRITE: 1,000,000]
Total errors              :          0 [WRITE: 0]
Total GC count            : 0
Total GC memory           : 0.000 KiB
Total GC time             :    0.0 seconds
Avg GC time               :    NaN ms
StdDev GC time            :    0.0 ms
Total operation time      : 00:04:37
```

### Read
```
 ./cassandra-stress read n=200000 -rate threads=50 -graph file=ocs-read-200000-benchmark.html title=read-200000 revision=benchmark-1

Results:
Op rate                   :    3,258 op/s  [READ: 3,258 op/s]
Partition rate            :    3,258 pk/s  [READ: 3,258 pk/s]
Row rate                  :    3,258 row/s [READ: 3,258 row/s]
Latency mean              :   15.3 ms [READ: 15.3 ms]
Latency median            :    1.8 ms [READ: 1.8 ms]
Latency 95th percentile   :   86.2 ms [READ: 86.2 ms]
Latency 99th percentile   :  104.9 ms [READ: 104.9 ms]
Latency 99.9th percentile :  209.6 ms [READ: 209.6 ms]
Latency max               :  404.0 ms [READ: 404.0 ms]
Total partitions          :    200,000 [READ: 200,000]
Total errors              :          0 [READ: 0]
Total GC count            : 0
Total GC memory           : 0.000 KiB
Total GC time             :    0.0 seconds
Avg GC time               :    NaN ms
StdDev GC time            :    0.0 ms
Total operation time      : 00:01:01
```

## Create a snapgroup (consistent group snapshot)

Snapgroups (https://docs.portworx.com/scheduler/kubernetes/snaps-group.html) can be used by snapshotting a group of volumes with a label or group id. Below, is a snippet that shows you each volume gets the `label` of `app=cassandra` so we can create a snapgroup from this informaiton.

```
volumeClaimTemplates:
  - metadata:
      name: cassandra-data
      annotations:
        volume.beta.kubernetes.io/storage-class: px-storageclass
      labels:
        app: cassandra
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

An example snapshot group is below. You can see we include the label selector in the annotations as well as this will be a local snapgroup (optionally we can make it a cloudsnap).

```
# This consistent snapshot group will snapshot ALL
# volumes with label `db-group: db-data`
# as a consistent portworx local snapgroup
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cassandra-snapshot-group
  namespace: default
  annotations:
    portworx/snapshot-type: local
    portworx.selector/app: cassandra
spec:
  # persistentVolumeClaimName in the above spec can be name of 
  # any single PVC that will get matched using the selector
  persistentVolumeClaimName: cassandra-data-cassandra-0
```

## Create the snapshot group
```
kubectl create -f specs/cassandra-snapgroup.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "cassandra-snapshot-group" created
```

## View the snapshot group
```
kubectl get volumesnapshot
NAME                                                                                       AGE
cassandra-snapshot-group                                                                   20s
cassandra-snapshot-group-cassandra-data-cassandra-0-c567c826-ce20-11e8-8281-42010a8e0115   18s
cassandra-snapshot-group-cassandra-data-cassandra-1-c567c826-ce20-11e8-8281-42010a8e0115   16s
cassandra-snapshot-group-cassandra-data-cassandra-2-c567c826-ce20-11e8-8281-42010a8e0115   17s
```

## View the group snapshot in `pxctl`
```
pxctl volume list -s
ID			NAME												SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	STATUS		HA-STATE
623237428776128574	group_snap_2_1539350078_pvc-011958c8-ce20-11e8-8281-42010a8e0115				10 GiB	2	no	no		MEDIUM		none - detached	Detached
1063977941417974563	group_snap_2_1539350078_pvc-18076b5b-ce20-11e8-8281-42010a8e0115				10 GiB	2	no	no		MEDIUM		none - detached	Detached
586092072464896250	group_snap_2_1539350078_pvc-ee62365d-ce1f-11e8-8281-42010a8e0115				10 GiB	2	no	no		MEDIUM		none - detached	Detached
```

# Create 3DSnap

Portworx supports 3DSnapshots which allows specifying pre and post rules that are run on the application pods using the volumes being snapshotted. This allows users to quiesce the applications before the snapshot is taken and resume I/O after the snapshot is taken.

## Presnap rule

Use `nodetool flush` on cassandra before snapshoting.

```
apiVersion: stork.libopenstorage.org/v1alpha1
kind: Rule
metadata:
  name: px-cassandra-presnap-rule
spec:
  - podSelector:
      app: cassandra
    actions:
    - type: command
      value: nodetool flush
```

### create the rule
```
kubectl create -f specs/cassandra-3dsnap-presnap-rule.yaml
rule.stork.libopenstorage.org "px-cassandra-presnap-rule" created
```

## Define a PVC that uses this pre-snapshot rule.

> Note becuase we are using `portworx.selector/app: cassandra` the pods also have this label.

```
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cassandra-3d-snapshot
  annotations:
    portworx.selector/app: cassandra
    stork.rule/pre-snapshot: px-cassandra-presnap-rule
spec:
  persistentVolumeClaimName: cassandra-data-cassandra-0
```

### Create the snapshot
```
kubectl create -f specs/cassandra-3d-snap.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "cassandra-3d-snapshot" created
```

### View the snapshot
```
kubectl describe volumesnapshot cassandra-3d-snapshot
Name:         cassandra-3d-snapshot
Namespace:    default
Labels:       SnapshotMetadata-PVName=pvc-ee62365d-ce1f-11e8-8281-42010a8e0115
              SnapshotMetadata-Timestamp=1539354684474325222
Annotations:  portworx.selector/app=cassandra
              stork.rule/pre-snapshot=px-cassandra-presnap-rule
API Version:  volumesnapshot.external-storage.k8s.io/v1
Kind:         VolumeSnapshot
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-10-12T14:31:24Z
  Generation:          0
  Resource Version:    238299
  Self Link:           /apis/volumesnapshot.external-storage.k8s.io/v1/namespaces/default/volumesnapshots/cassandra-3d-snapshot
  UID:                 7ef55ed0-ce2b-11e8-8281-42010a8e0115
Spec:
  Persistent Volume Claim Name:  cassandra-data-cassandra-0
  Snapshot Data Name:            k8s-volume-snapshot-83f2234c-ce2b-11e8-aa99-0a580a0c001b
Status:
  Conditions:
    Last Transition Time:  2018-10-12T14:31:32Z
    Message:               Snapshot created successfully and it is ready
    Reason:
    Status:                True
    Type:                  Ready
  Creation Timestamp:      <nil>
Events:                    <none>
```

### List the 3d snapshot

> With this snapshot, Stork will run the px-cassandra-presnap-rule rule on all pods that are using PVCs that match labels app=cassandra. Hence this will be a group snapshot.

```
kubectl get volumesnapshot
NAME                                                                                       AGE
cassandra-3d-snapshot                                                                      10s
cassandra-3d-snapshot-cassandra-data-cassandra-0-7ef55ed0-ce2b-11e8-8281-42010a8e0115      2s
cassandra-3d-snapshot-cassandra-data-cassandra-1-7ef55ed0-ce2b-11e8-8281-42010a8e0115      3s
cassandra-3d-snapshot-cassandra-data-cassandra-2-7ef55ed0-ce2b-11e8-8281-42010a8e0115      3s
```

## Fail Cassandra and Restore from 3d Snap

### Fail

```
$ kubectl get statefulset
NAME        DESIRED   CURRENT   AGE
cassandra   3         3         2h

$  kubectl delete statefulset cassandra
statefulset.apps "cassandra" deleted
```

### Restore from 3dsnap

Create clones specs from 3d snapshots

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cassandra-data-cassandra-clone-0
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: cassandra-3d-snapshot-cassandra-data-cassandra-0-7ef55ed0-ce2b-11e8-8281-42010a8e0115
spec:
  accessModes:
     - ReadWriteOnce
  storageClassName: stork-snapshot-sc
  resources:
    requests:
      storage: 10Gi
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cassandra-data-cassandra-clone-1
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: cassandra-3d-snapshot-cassandra-data-cassandra-1-7ef55ed0-ce2b-11e8-8281-42010a8e0115
spec:
  accessModes:
     - ReadWriteOnce
  storageClassName: stork-snapshot-sc
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cassandra-data-cassandra-clone-2
  annotations:
    snapshot.alpha.kubernetes.io/snapshot: cassandra-3d-snapshot-cassandra-data-cassandra-2-7ef55ed0-ce2b-11e8-8281-42010a8e0115
spec:
  accessModes:
     - ReadWriteOnce
  storageClassName: stork-snapshot-sc
  resources:
    requests:
      storage: 10Gi
```

Create them.
```
kubectl create -f specs/cassandra-pwx-clones-from-3dsnap.yaml
persistentvolumeclaim "cassandra-data-cassandra-clone-0" created
persistentvolumeclaim "cassandra-data-cassandra-clone-1" created
persistentvolumeclaim "cassandra-data-cassandra-clone-2" created
```

### View available PVCs
```
kubectl get pvc | grep cassandra-data-cassandra-clone
cassandra-data-cassandra-clone-0   Bound     pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115   10Gi       RWO            stork-snapshot-sc          6m
cassandra-data-cassandra-clone-1   Bound     pvc-f772b417-ce31-11e8-8281-42010a8e0115   10Gi       RWO            stork-snapshot-sc          6m
cassandra-data-cassandra-clone-2   Bound     pvc-f7795675-ce31-11e8-8281-42010a8e0115   10Gi       RWO            stork-snapshot-sc          6m
```

They are all unattached
```
pxctl volume list | grep pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115
322274964403122850       pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115                            10 GiB  2       no      no              MEDIUM          none - detached                 Detached

▶ pxctl volume list | grep pvc-f772b417-ce31-11e8-8281-42010a8e0115
760161115976553002       pvc-f772b417-ce31-11e8-8281-42010a8e0115                            10 GiB  2       no      no              MEDIUM          none - detached                 Detached

▶ pxctl volume list | grep pvc-f7795675-ce31-11e8-8281-42010a8e0115
1047783292593844590      pvc-f7795675-ce31-11e8-8281-42010a8e0115                            10 GiB  2       no      no              MEDIUM          none - detached                 Detached

```

create a StatefulSet that now uses `cassandra-clone` for its name and other parameters so it picks up our PVC clones.

>There are others, please see `specs/cassandra-clone-from-3dsnap.yaml`

```
kind: StatefulSet
metadata:
  name: cassandra-clone
.
.
.
- name: CASSANDRA_SEEDS
  value: "cassandra-0-clone.cassandra-clone.default.svc.cluster.local"
```

Create the clone-based cassandra cluster
```
kubectl create -f specs/cassandra-clone-from-3dsnap.yaml
service/cassandra-clone created
statefulset.apps/cassandra-clone created
```

### view the clone
```
kubectl get po -l app=cassandra-clone
NAME                READY     STATUS    RESTARTS   AGE
cassandra-clone-0   1/1       Running   0          1m
cassandra-clone-1   1/1       Running   0          1m
cassandra-clone-2   1/1       Running   0          53s
```

### Make sure cassandra is using our 3dsnaped clones.

> `pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115` is a pvc name from `kubectl get pvc | grep cassandra-data-cassandra-clone-0`

```
$ kubectl describe po cassandra-clone-0 | grep pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115
Normal  SuccessfulMountVolume  7m    kubelet, gke-test-cluster-ryan-default-pool-a072badd-sqbk  MountVolume.SetUp succeeded for volume "pvc-f76b4ebb-ce31-11e8-8281-42010a8e0115"
```

### View the data is still there
```
kubectl exec -it cassandra-clone-0 /bin/bash

root@cassandra-clone-0:/# apt update -y; apt install -y python python-pip; pip install cqlsh;

root@cassandra-clone-0:/# /usr/local/bin/cqlsh --cqlversion="3.4.2"
Connected to K8Demo at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.9 | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
cqlsh>
cqlsh> describe keyspaces;

tutorialspoint  system_auth  system              system_traces
system_schema   newkeyspace  system_distributed

cqlsh> use newkeyspace;
cqlsh:newkeyspace> select * from emp;

 emp_id | emp_city  | emp_name | emp_phone  | emp_sal
--------+-----------+----------+------------+---------
      1 | Hyderabad |      ram | 9848022338 |   50000
```