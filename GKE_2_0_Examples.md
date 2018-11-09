
# Portworx 2.0 Augmentation, Cluster Utilization

> Note: we could also use `local-ssd` https://cloud.google.com/compute/docs/disks/local-ssd#create_local_ssd or https://cloud.google.com/sdk/gcloud/reference/container/clusters/create#--local-ssd-count for one instance type to show a more impactful migration and not just moving pd's or ebs.

> https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/local-ssd#restrictions

```
Because local SSDs are physically attached to the node's host virtual machine instance, any data stored in them only exists on that node. Since the data stored on the disks is local, your application must be resilient to this data being unavailable.

Data stored on local SSDs are ephemeral. A Pod that writes to a local SSD might lose access to the data stored on the disk if the Pod is rescheduled away from that node. Additionally, if the node is terminated, upgraded, or repaired the data will be erased.

You cannot add local SSDs to an existing node pool.
```

## Cluster 1 (n1-standard-2: 2vCPU 7.50GB Mem uses Local SSD)

```
gcloud container clusters create ryan-2-0-cluster-1 \
--zone us-east1-d \
--disk-type=pd-ssd \
--disk-size=20GB \
--labels=portworx=gke \
--machine-type=n1-standard-2 \
--num-nodes=3 \
--image-type ubuntu \
--local-ssd-count=1 \
--enable-autoscaling --max-nodes=3 --min-nodes=3
```

`gcloud container clusters get-credentials --zone us-east1-d ryan-2-0-cluster-1`

Make sure and unmount the SSDs `/dev/sdb` from `ssd0` on each node
```
gcloud compute ssh <node> --zone us-east1-d --command "sudo umount /mnt/disks/ssd0"
```

Set the role to use
```
kubectl create clusterrolebinding \
    myname-cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=ryan.wallner@portworx.com
```

Create px cluster
```
kubectl create -f px-spec-pxhostesetcd-2.0-localssd.yaml
```

```
kubectl exec -it -n kube-system portworx-4gd2d bash
bash-4.2# /opt/pwx/bin/pxctl status
Status: PX is operational
License: Trial (expires in 31 days)
Node ID: b94ce17a-a5ff-40ef-b7eb-c134c685bce2
	IP: 10.142.0.2
 	Local Storage Pool: 1 pool
	POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE		REGION
	0	HIGH		raid0		375 GiB	11 GiB	Online	us-east1-d	us-east1
	Local Storage Devices: 1 device
	Device	Path		Media Type		Size		Last-Scan
	0:1	/dev/sdb	STORAGE_MEDIUM_SSD	375 GiB		05 Nov 18 18:52 UTC
	total			-			375 GiB
Cluster Summary
	Cluster ID: px-cluster-1-4e2d7175-44df-43f9-b652-f47831a06779
	Cluster UUID: a14cc184-57e8-4436-b077-1ed8f8d1c6f0
	Scheduler: kubernetes
	Nodes: 3 node(s) with storage (3 online)
	IP		ID					SchedulerName						StorageNode	Used	Capacity	Status	StorageStatus	Version		Kernel		OS
	10.142.0.3	e2cae308-f527-4697-944c-02ffd34a61e3	gke-ryan-2-0-cluster-1-default-pool-eb1108b4-3zh9	Yes		11 GiB	375 GiB		Online	Up		2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
	10.142.0.2	b94ce17a-a5ff-40ef-b7eb-c134c685bce2	gke-ryan-2-0-cluster-1-default-pool-eb1108b4-lhr6	Yes		11 GiB	375 GiB		Online	Up (This node)	2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
	10.142.0.4	814bc809-890d-46dd-868b-9364c9541742	gke-ryan-2-0-cluster-1-default-pool-eb1108b4-94nr	Yes		11 GiB	375 GiB		Online	Up		2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
Global Storage Pool
	Total Used    	:  33 GiB
	Total Capacity	:  1.1 TiB
```

Deploy prometheus

```
kubectl create -f specs-common/prometheus-operator.yaml
kubectl create -f specs-common/service-monitor.yaml
kubectl create -f specs-common/prometheus-cluster.yaml
```

Get IP to use `:30900` for Prometheus
```
kubectl describe no <node>| grep ExternalIP
```

Use the following metrics to show
- px_node_stats_free_mem 
- px_node_stats_cpu_usage

```
▶ kubectl config  use-context gke_portworx-eng_us-east1-d_ryan-2-0-cluster-1
Switched to context "gke_portworx-eng_us-east1-d_ryan-2-0-cluster-1".

                                                                     11d ◒
▶ kubectl get no
NAME                                                STATUS    ROLES     AGE       VERSION
gke-ryan-2-0-cluster-1-default-pool-eb1108b4-3zh9   Ready     <none>    2h        v1.9.7-gke.6
gke-ryan-2-0-cluster-1-default-pool-eb1108b4-94nr   Ready     <none>    2h        v1.9.7-gke.6
gke-ryan-2-0-cluster-1-default-pool-eb1108b4-lhr6   Ready     <none>    2h        v1.9.7-gke.6
```

## Cluster 2 (n1-standard-8: 8vCPU 30GB Mem uses PD Auto Provision Disk)

```
gcloud container clusters create ryan-2-0-cluster-2 \
--zone us-east1-d \
--disk-type=pd-ssd \
--disk-size=20GB \
--labels=portworx=gke \
--machine-type=n1-standard-8 \
--num-nodes=3 \
--image-type ubuntu \
--enable-autoscaling --max-nodes=3 --min-nodes=3
```

`gcloud container clusters get-credentials --zone us-east1-d ryan-2-0-cluster-2`

Set the role to use
```
kubectl create clusterrolebinding \
    myname-cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=ryan.wallner@portworx.com
```

Create px cluster
```
kubectl create -f px-spec-pxhostesetcd-2.0-pdauto.yaml
```

```
▶ kubectl exec -it -n kube-system portworx-26w7n bash
bash-4.2# /opt/pwx/bin/pxctl status
Status: PX is operational
License: Trial (expires in 31 days)
Node ID: befc1de1-24e5-4f3a-b831-9c38ef614a77
	IP: 10.142.0.5
 	Local Storage Pool: 1 pool
	POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE		REGION
	0	MEDIUM		raid0		100 GiB	6.0 GiB	Online	us-east1-d	us-east1
	Local Storage Devices: 1 device
	Device	Path		Media Type		Size		Last-Scan
	0:1	/dev/sdb	STORAGE_MEDIUM_MAGNETIC	100 GiB		05 Nov 18 19:09 UTC
	total			-			100 GiB
Cluster Summary
	Cluster ID: px-cluster-2-a88c5e33-e632-41e2-884a-6a93e43b5ac1
	Cluster UUID: c0e18b3a-3afd-443c-84e3-d7c503a90a91
	Scheduler: kubernetes
	Nodes: 3 node(s) with storage (3 online)
	IP		ID					SchedulerName						StorageNode	Used	Capacity	Status	StorageStatus	Version		Kernel		OS
	10.142.0.5	befc1de1-24e5-4f3a-b831-9c38ef614a77	gke-ryan-2-0-cluster-2-default-pool-36435116-bl8x	Yes		0 B	100 GiB		Online	Up (This node)	2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
	10.142.0.7	21a77177-0832-4e57-8210-c4f2b8972683	gke-ryan-2-0-cluster-2-default-pool-36435116-j5wp	Yes		0 B	100 GiB		Online	Up		2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
	10.142.0.6	0a41c350-52c9-4baf-a2ce-28f98ffcd99f	gke-ryan-2-0-cluster-2-default-pool-36435116-4zgj	Yes		0 B	100 GiB		Online	Up		2.0.0.0-7e131ac	4.15.0-1017-gcp	Ubuntu 16.04.5 LTS
Global Storage Pool
	Total Used    	:  0 B
	Total Capacity	:  300 GiB
```

Deploy prometheus

```
kubectl create -f specs-common/prometheus-operator.yaml
kubectl create -f specs-common/service-monitor.yaml
kubectl create -f specs-common/prometheus-cluster.yaml
```

Get IP to use `:30900` for Prometheus
```
kubectl describe no <node> | grep ExternalIP
```

Use the following metrics to show
- px_node_stats_free_mem 
- px_node_stats_cpu_usage

```
▶ kubectl config  use-context gke_portworx-eng_us-east1-d_ryan-2-0-cluster-2
Switched to context "gke_portworx-eng_us-east1-d_ryan-2-0-cluster-2".

                                                                        11d ◒
▶ kubectl get no
NAME                                                STATUS    ROLES     AGE       VERSION
gke-ryan-2-0-cluster-2-default-pool-36435116-4zgj   Ready     <none>    1h        v1.9.7-gke.6
gke-ryan-2-0-cluster-2-default-pool-36435116-bl8x   Ready     <none>    1h        v1.9.7-gke.6
gke-ryan-2-0-cluster-2-default-pool-36435116-j5wp   Ready     <none>    1h        v1.9.7-gke.6
```

## Deploy to cluster 1


Create Apps
```
kubectl create -f specs-common/postgres-px-1.yaml
kubectl create -f specs-common/postgres-px-2.yaml
kubectl create -f specs-common/cassandra-pwx-1.yaml
kubectl create -f specs-common/cassandra-pwx-2.yaml
kubectl create -f specs-common/wordpress-mysql-vols-2.yaml
kubectl create secret generic mysql-pass-2 -n demo --from-file=files/mysqlpasswd.txt
kubectl create -f specs-common/wp-mysql-2.yaml
kubectl create -f specs-common/mysql-1.yaml
kubectl create -f specs-common/mysql-2.yaml
kubectl create -f specs-common/mysql-3.yaml
```

Show Utilization

## Pair the clusters

Open ports 9100 / 9010
```
gcloud compute firewall-rules create pwx-ingress-api \
  --allow tcp:9010 \
  --target-tags=portworx

gcloud compute firewall-rules create pwx-ingress-object1 \
  --allow tcp:9010 \
  --target-tags=portworx
```

Edit `specs-common/clusterpair.yaml`

Create the cluster pair
```
▶ kubectl create -f clusterpair.yaml
clusterpair.stork.libopenstorage.org "gke-pd-clusterpair" created
```

## GOTCHYAS

update cmd-path
```
cmd-path: /usr/local/bin/gcloud
```

download gcloud to stork pod
```
```

## General Plan

a. deploy portworx 2.0 on both clusters (done)
b. deploy cluster monitoring with node-exporter and prometheus on both clusters (done)
d. Deploy a whole bunch of applications/databases on Cluster 1 (cassandra)
c. Show utilization (done)
d. Create migration to larger cluster with different compute/storage.
e. Show its using the same data. (web front end?)
f. Show utilization on new cluster once migration is complete.