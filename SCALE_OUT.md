# How to scale out pwx cluster on GKE

### Current cluster

3 nodes from `pxctl`

```
$ pxctl status

Status: PX is operational
License: Trial (expires in 31 days)
Node ID: gke-test-cluster-ryan-default-pool-a072badd-l093
        IP: 10.142.0.3
        Local Storage Pool: 1 pool
        POOL    IO_PRIORITY     RAID_LEVEL      USABLE  USED    STATUS  ZONE            REGION
        0       MEDIUM          raid0           50 GiB  4.3 GiB Online  us-east1-d      us-east1
        Local Storage Devices: 1 device
        Device  Path            Media Type              Size            Last-Scan
        0:1     /dev/sdb        STORAGE_MEDIUM_MAGNETIC 50 GiB          11 Oct 18 19:35 UTC
        total                   -                       50 GiB
Cluster Summary
        Cluster ID: px-cluster-007aa43d-f378-49d6-b410-0670dde8f5b7
        Cluster UUID: d5b722d7-679d-4cbb-8ade-188fbbf1e9a6
        Scheduler: kubernetes
        Nodes: 3 node(s) with storage (3 online)
        IP              ID                                                      StorageNode     Used    Capacity        Status  StorageStatus   Version         Kernel          OS
        10.142.0.4      gke-test-cluster-ryan-default-pool-a072badd-sqbk        Yes             4.3 GiB 50 GiB          Online  Up              1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
        10.142.0.3      gke-test-cluster-ryan-default-pool-a072badd-l093        Yes             4.3 GiB 50 GiB          Online  Up (This node)  1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
        10.142.0.2      gke-test-cluster-ryan-default-pool-a072badd-bztf        Yes             4.3 GiB 50 GiB          Online  Up              1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
Global Storage Pool
        Total Used      :  13 GiB
        Total Capacity  :  150 GiB
```

```
pxctl cluster list
Cluster ID: px-cluster-007aa43d-f378-49d6-b410-0670dde8f5b7
Cluster UUID: d5b722d7-679d-4cbb-8ade-188fbbf1e9a6
Status: OK

Nodes in the cluster:
ID							DATA IP		CPU		MEM TOTAL	MEM FREE	CONTAINERS	VERSION			Kernel			OS	STATUS
gke-test-cluster-ryan-default-pool-a072badd-bztf	10.142.0.2	2.39899		16 GB		14 GB		N/A		1.6.1.1-aaccadf		4.15.0-1017-gcp		Ubuntu 16.04.5 LTS	Online
gke-test-cluster-ryan-default-pool-a072badd-l093	10.142.0.3	5.681818	16 GB		13 GB		N/A		1.6.1.1-aaccadf		4.15.0-1017-gcp		Ubuntu 16.04.5 LTS	Online
gke-test-cluster-ryan-default-pool-a072badd-sqbk	10.142.0.4	9.240506	16 GB		11 GB		N/A		1.6.1.1-aaccadf		4.15.0-1017-gcp		Ubuntu 16.04.5 LTS	Online
```

3 nodes from gcloud cli
```
gcloud container clusters list
NAME               LOCATION       MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
test-cluster-ryan  us-east1-d     1.9.7-gke.6     35.227.37.61    n1-standard-4  1.9.7-gke.6   3          RUNNING
```

View from pxctl
```
kubectl get po -n kube-system -l name=portworx                                                                                    
NAME             READY     STATUS    RESTARTS   AGE
portworx-dgckz   1/1       Running   4          22h
portworx-jhxbd   1/1       Running   4          22h
portworx-tbbx6   1/1       Running   4          22h
```

### Scale out

Increase the cluster
```
gcloud container clusters resize test-cluster-ryan --size=4
Pool [default-pool] for [test-cluster-ryan] will be resized to 4.

Do you want to continue (Y/n)?  Y

Resizing test-cluster-ryan...done.
Updated [https://container.googleapis.com/v1/projects/portworx-eng/zones/us-east1-d/clusters/test-cluster-ryan].
```

Now youll see 4
```
gcloud container clusters list | grep test-cluster-ryan
test-cluster-ryan  us-east1-d     1.9.7-gke.6     35.227.37.61    n1-standard-4  1.9.7-gke.6   4          RUNNING
```

You'll notice portworx automatically starts adding support for PX to that node
```
kubectl get po -n kube-system -l name=portworx                                                  
NAME             READY     STATUS    RESTARTS   AGE
portworx-dgckz   1/1       Running   4          22h
portworx-ghmqn   0/1       Running   0          13s
portworx-jhxbd   1/1       Running   4          22h
portworx-tbbx6   1/1       Running   4          22h
```

Then it succefully adds the node
```
```
kubectl get po -n kube-system -l name=portworx                                                  
NAME             READY     STATUS    RESTARTS   AGE
portworx-dgckz   1/1       Running   4          22h
portworx-ghmqn   1/1       Running   0          13s
portworx-jhxbd   1/1       Running   4          22h
portworx-tbbx6   1/1       Running   4          22h
```

You can see via pxctl there is more space and 4 nodes

> note there is `200GB` instead of `150GB` now since we scaled out.

```
pxctl status
Status: PX is operational
License: Trial (expires in 30 days)
Node ID: gke-test-cluster-ryan-default-pool-a072badd-l093
        IP: 10.142.0.3
        Local Storage Pool: 1 pool
        POOL    IO_PRIORITY     RAID_LEVEL      USABLE  USED    STATUS  ZONE            REGION
        0       MEDIUM          raid0           50 GiB  11 GiB  Online  us-east1-d      us-east1
        Local Storage Devices: 1 device
        Device  Path            Media Type              Size            Last-Scan
        0:1     /dev/sdb        STORAGE_MEDIUM_MAGNETIC 50 GiB          11 Oct 18 19:35 UTC
        total                   -                       50 GiB
Cluster Summary
        Cluster ID: px-cluster-007aa43d-f378-49d6-b410-0670dde8f5b7
        Cluster UUID: d5b722d7-679d-4cbb-8ade-188fbbf1e9a6
        Scheduler: kubernetes
        Nodes: 4 node(s) with storage (4 online)
        IP              ID                                                      StorageNode     Used    Capacity        Status  StorageStatus   Version         Kernel          OS
        10.142.0.4      gke-test-cluster-ryan-default-pool-a072badd-sqbk        Yes             5.6 GiB 50 GiB          Online  Up              1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
        10.142.0.3      gke-test-cluster-ryan-default-pool-a072badd-l093        Yes             11 GiB  50 GiB          Online  Up (This node)  1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
        10.142.0.5      gke-test-cluster-ryan-default-pool-a072badd-h087        Yes             0 B     50 GiB          Online  Up              1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
        10.142.0.2      gke-test-cluster-ryan-default-pool-a072badd-bztf        Yes             11 GiB  50 GiB          Online  Up              1.6.1.1-aaccadf 4.15.0-1017-gcp Ubuntu 16.04.5 LTS
Global Storage Pool
        Total Used      :  27 GiB
        Total Capacity  :  200 GiB
```

```
pxctl cluster list
Cluster ID: px-cluster-007aa43d-f378-49d6-b410-0670dde8f5b7
Cluster UUID: d5b722d7-679d-4cbb-8ade-188fbbf1e9a6
Status: OK

Nodes in the cluster:
ID                                                      DATA IP         CPU             MEM TOTAL       MEM FREE        CONTAINERS      VERSION                 Kernel                  OS                      STATUS
gke-test-cluster-ryan-default-pool-a072badd-bztf        10.142.0.2      2.143758        16 GB           14 GB           N/A             1.6.1.1-aaccadf         4.15.0-1017-gcp         Ubuntu 16.04.5 LTS      Online
gke-test-cluster-ryan-default-pool-a072badd-h087        10.142.0.5      1.38539         16 GB           14 GB           N/A             1.6.1.1-aaccadf         4.15.0-1017-gcp         Ubuntu 16.04.5 LTS      Online
gke-test-cluster-ryan-default-pool-a072badd-l093        10.142.0.3      1.903553        16 GB           13 GB           N/A             1.6.1.1-aaccadf         4.15.0-1017-gcp         Ubuntu 16.04.5 LTS      Online
gke-test-cluster-ryan-default-pool-a072badd-sqbk        10.142.0.4      28.535032       16 GB           11 GB           N/A             1.6.1.1-aaccadf         4.15.0-1017-gcp         Ubuntu 16.04.5 LTS      Online
```