# Cassandra on GKE with PWX

1. create cassandra
2. create some data by running perf
3. create group snap

(TODO)
- backup?
- cloudsnap?
- restore?

## Create

```
kubectl create -f specs/cassandra-pwx.yaml
service/cassandra created
storageclass.storage.k8s.io/px-storageclass created
statefulset.apps/cassandra created
```

## Create data via perf

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