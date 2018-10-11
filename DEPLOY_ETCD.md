## Deploy Etcd 

Deploy Operator
```
kubectl apply -f etcd/rbac/cluster-role.yaml
kubectl apply -f etcd/rbac/cluster-role-binding.yaml
kubectl apply -f etcd/operator/etcd-operator.yaml
```

Wait until its ready
```
kubectl get pods -o wide -n kube-system -l name=etcd-operator
NAME                             READY     STATUS    RESTARTS   AGE       IP          NODE                                               NOMINATED NODE
etcd-operator-768dc99865-p2l7p   1/1       Running   0          22s       10.12.1.7   gke-test-cluster-ryan-default-pool-ac8eed24-c38h   <none>
```

Deploy Etcd with TLS
`kubectl apply -f etcd/etcd-cluster.yaml`

Monitor if its ready
```
kubectl get pods -o wide -n kube-system -l app=etcd
▶ kubectl get pods -o wide -n kube-system -l app=etcd
NAME                               READY     STATUS    RESTARTS   AGE       IP          NODE                                               NOMINATED NODE
portworx-etcd-cluster-hc7w6cfxtv   1/1       Running   0          49s       10.12.0.6   gke-test-cluster-ryan-default-pool-ac8eed24-ftf7   <none>
portworx-etcd-cluster-qfnqg6jj7q   1/1       Running   0          33s       10.12.2.6   gke-test-cluster-ryan-default-pool-ac8eed24-clh1   <none>
portworx-etcd-cluster-trckvhxp9w   1/1       Running   0          1m        10.12.1.8   gke-test-cluster-ryan-default-pool-ac8eed24-c38h   <none>
```

## Issues

### TLS with etcd-operator
Possible etcd tls bug with 3.2.7
 - https://github.com/etcd-io/etcd/issues/8603 
 (use non `-tls` versions of specs)

> TODO: currently this TLS isnt working with etcd 3.2.X. The cert doesnt match since pwx uses host networking and therefore doesnt used the kubernetes DNS and catch match the DNS name of cert. The IP is auto assigned so we cant place IP in the cert. Would need some level of DNS for this to work or for pwx to use internal dns and not host network.

```
kubectl create secret -n kube-system generic etcd-peer-tls --from-file=etcd/certs/peer-ca.crt --from-file=etcd/certs/peer.crt --from-file=etcd/certs/peer.key

kubectl create secret -n kube-system  generic etcd-server-tls --from-file=etcd/certs/server-ca.crt --from-file=etcd/certs/server.crt --from-file=etcd/certs/server.key

kubectl create secret -n kube-system  generic etcd-client-tls --from-file=etcd/certs/etcd-client-ca.crt --from-file=etcd/certs/etcd-client.crt --from-file=etcd/certs/etcd-client.key
```

### Using TLS or Not, etcd is killed? exit 137 in GKE cluster

See below
```
▶ kubectl get pods -o wide -n kube-system -l app=etcd
NAME                               READY     STATUS    RESTARTS   AGE       IP          NODE                                               NOMINATED NODE
portworx-etcd-cluster-c6stxcsmjw   0/1       Error     0          16m       10.12.2.7   gke-test-cluster-ryan-default-pool-ef6575a7-k3t8   <none>
portworx-etcd-cluster-hd5xzz8mjm   0/1       Error     0          16m       10.12.0.6   gke-test-cluster-ryan-default-pool-ef6575a7-w3p2   <none>
portworx-etcd-cluster-lc6kpnncsj   0/1       Error     0          17m       10.12.1.7   gke-test-cluster-ryan-default-pool-ef6575a7-k8jb   <none>
```

- Etcd is healthy for 5 mins, then dies once portworx starts? or maybe unrelated. 
- Even varified etcdctl works fine with certs or without.

```
▶ kubectl describe po portworx-etcd-cluster-c6stxcsmjw -n kube-system
Name:           portworx-etcd-cluster-c6stxcsmjw
Namespace:      kube-system
Node:           gke-test-cluster-ryan-default-pool-ef6575a7-k3t8/10.142.0.4
Start Time:     Thu, 11 Oct 2018 13:00:15 -0400
Labels:         app=etcd
                etcd_cluster=portworx-etcd-cluster
                etcd_node=portworx-etcd-cluster-c6stxcsmjw
Annotations:    etcd.version=3.2.7
Status:         Failed
IP:             10.12.2.7
Controlled By:  EtcdCluster/portworx-etcd-cluster
Init Containers:
  check-dns:
    Container ID:  docker://4b49d91d65257c130c54cfe2d6bd3c46c106409ad22188246b38af9fd3f6cd0d
    Image:         busybox:1.28.0-glibc
    Image ID:      docker-pullable://busybox@sha256:0b55a30394294ab23b9afd58fab94e61a923f5834fba7ddbae7f8e0c11ba85e6
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c

                            while ( ! nslookup portworx-etcd-cluster-c6stxcsmjw.portworx-etcd-cluster.kube-system.svc )
                            do
                              sleep 2
                            done
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Thu, 11 Oct 2018 13:00:18 -0400
      Finished:     Thu, 11 Oct 2018 13:00:20 -0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
Containers:
  etcd:
    Container ID:  docker://149c558792a6db44ba203c25ccb71428ac9ba8741f91d772448f3c607d5f2875
    Image:         quay.io/coreos/etcd:v3.2.7
    Image ID:      docker-pullable://quay.io/coreos/etcd@sha256:21c257bb292251372178f57fdefbced6b1e5f0c3b5214314701990454099eae0
    Ports:         2380/TCP, 2379/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /usr/local/bin/etcd
      --data-dir=/var/etcd/data
      --name=portworx-etcd-cluster-c6stxcsmjw
      --initial-advertise-peer-urls=http://portworx-etcd-cluster-c6stxcsmjw.portworx-etcd-cluster.kube-system.svc:2380
      --listen-peer-urls=http://0.0.0.0:2380
      --listen-client-urls=http://0.0.0.0:2379
      --advertise-client-urls=http://portworx-etcd-cluster-c6stxcsmjw.portworx-etcd-cluster.kube-system.svc:2379
      --initial-cluster=portworx-etcd-cluster-lc6kpnncsj=http://portworx-etcd-cluster-lc6kpnncsj.portworx-etcd-cluster.kube-system.svc:2380,portworx-etcd-cluster-hd5xzz8mjm=http://portworx-etcd-cluster-hd5xzz8mjm.portworx-etcd-cluster.kube-system.svc:2380,portworx-etcd-cluster-c6stxcsmjw=http://portworx-etcd-cluster-c6stxcsmjw.portworx-etcd-cluster.kube-system.svc:2380
      --initial-cluster-state=existing
    State:          Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Thu, 11 Oct 2018 13:00:24 -0400
      Finished:     Thu, 11 Oct 2018 13:06:17 -0400
    Ready:          False
    Restart Count:  0
    Liveness:       exec [/bin/sh -ec ETCDCTL_API=3 etcdctl get foo] delay=10s timeout=10s period=60s #success=1 #failure=3
    Readiness:      exec [/bin/sh -ec ETCDCTL_API=3 etcdctl get foo] delay=1s timeout=5s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/etcd from etcd-data (rw)
Conditions:
  Type           Status
  Initialized    True
  Ready          False
  PodScheduled   True
Volumes:
  etcd-data:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                 Age              From                                                       Message
  ----     ------                 ----             ----                                                       -------
  Normal   Scheduled              12m              default-scheduler                                          Successfully assigned portworx-etcd-cluster-c6stxcsmjw to gke-test-cluster-ryan-default-pool-ef6575a7-k3t8
  Normal   SuccessfulMountVolume  12m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  MountVolume.SetUp succeeded for volume "etcd-data"
  Normal   Pulling                12m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  pulling image "busybox:1.28.0-glibc"
  Normal   Pulled                 12m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Successfully pulled image "busybox:1.28.0-glibc"
  Normal   Created                12m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Created container
  Normal   Started                12m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Started container
  Normal   Pulling                11m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  pulling image "quay.io/coreos/etcd:v3.2.7"
  Normal   Pulled                 11m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Successfully pulled image "quay.io/coreos/etcd:v3.2.7"
  Normal   Created                11m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Created container
  Normal   Started                11m              kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Started container
  Warning  Unhealthy              6m               kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Liveness probe errored: rpc error: code = Unknown desc = container not running (149c558792a6db44ba203c25ccb71428ac9ba8741f91d772448f3c607d5f2875)
  Warning  Unhealthy              6m (x2 over 6m)  kubelet, gke-test-cluster-ryan-default-pool-ef6575a7-k3t8  Readiness probe errored: rpc error: code = Unknown desc = container not running (149c558792a6db44ba203c25ccb71428ac9ba8741f91d772448f3c607d5f2875)
```