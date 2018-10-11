# Cassandra on GKE with PWX

1. create cassandra
2. create some data by running perf

(TODO)

3. create group snap

(TODO)
- backup
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