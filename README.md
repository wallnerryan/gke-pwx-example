# gke-pwx-example

### relative links
- machine types
  - https://cloud.google.com/compute/docs/machine-types
- zones
  - https://cloud.google.com/compute/docs/regions-zones/#available  
- regions
  - https://cloud.google.com/compute/docs/regions-zones/#available
- clusters
  - https://cloud.google.com/kubernetes-engine/docs/how-to/managing-clusters
- kubectl
  - https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl
- etcd TLS
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/cluster_tls.md
- etcd backup operator
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/walkthrough/backup-operator.md
- etcd restore operator
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/walkthrough/restore-operator.md
- etcd spec example
  - https://github.com/coreos/etcd-operator/blob/master/doc/user/spec_examples.md

## Setup
```
▶gcloud config set project <project>

▶gcloud config set compute/region <region> (needed for regional/multi-az)

▶gcloud config set compute/zone <zone>

Update
▶ gcloud components update
```

## Create a cluster (regional/multizone)
```
gcloud container clusters create test-cluster-ryan \
--region us-east1 \
--node-locations us-east1-b,us-east1-c,us-east1-d \
--disk-type=pd-ssd \
--disk-size=50GB \
--labels=portworx=gke \
--machine-type=n1-standard-8 \
--num-nodes=3 \
--image-type ubuntu \
--enable-autoscaling --max-nodes=6 --min-nodes=3
```

## Create a cluster (single zone)
```
gcloud container clusters create test-cluster-ryan \
--zone us-east1-d \
--disk-type=pd-ssd \
--disk-size=50GB \
--labels=portworx=gke \
--machine-type=n1-standard-4 \
--num-nodes=3 \
--image-type ubuntu \
--enable-autoscaling --max-nodes=6 --min-nodes=3
```

## Manage your cluster

Describe cluster
`gcloud container clusters describe test-cluster-ryan`

Set default cluster
`gcloud config set container/cluster test-cluster-ryan`

Get endpoint
```
gcloud container clusters describe test-cluster-ryan | grep endpoint
endpoint: 35.231.17.210
```

Configure kubectl
`gcloud container clusters get-credentials test-cluster-ryan`

Get nodes
```
kubectl get no
NAME                                               STATUS    ROLES     AGE       VERSION
gke-test-cluster-ryan-default-pool-ac8eed24-c38h   Ready     <none>    16m       v1.9.7-gke.6
gke-test-cluster-ryan-default-pool-ac8eed24-clh1   Ready     <none>    15m       v1.9.7-gke.6
gke-test-cluster-ryan-default-pool-ac8eed24-ftf7   Ready     <none>    16m       v1.9.7-gke.6
```

## Deploy Portworx

### Enable Compute API
`gcloud services enable compute.googleapis.com`

Create a ClusterRoleBinding for your user using the following commands:

### Get current google identity
``` 
gcloud info | grep Account
Account: [myname@example.org]
```

### Grant cluster-admin to your current identity
```
kubectl create clusterrolebinding myname-cluster-admin-binding \
  --clusterrole=cluster-admin --user=myname@example.org
Clusterrolebinding "myname-cluster-admin-binding" created
```

## Deploy Portworx DaemonSet

### Create PX Spec

> This guide proviced a pre-configure portworx spec in `specs/px-spec.yaml`

Gather the Etcd endpoint that portworx needs in https://install.portworx.com/
```
kubectl describe svc portworx-etcd-cluster-client -n kube-system | grep IP:
IP:                10.15.250.249
```

1. Visit and fill out https://install.portworx.com/
2. Make sure and modify the portworx spec to include etcd certs (https://docs.portworx.com/scheduler/kubernetes/etcd-certs-using-secrets.html#edit-portworx-spec)

### Use your produces spec or the one in `specs/`

```
$ kubectl create -f specs/px-spec.yaml
configmap/stork-config created
serviceaccount/stork-account created
clusterrole.rbac.authorization.k8s.io/stork-role created
clusterrolebinding.rbac.authorization.k8s.io/stork-role-binding created
service/stork-service created
deployment.extensions/stork created
storageclass.storage.k8s.io/stork-snapshot-sc created
serviceaccount/stork-scheduler-account created
clusterrole.rbac.authorization.k8s.io/stork-scheduler-role created
clusterrolebinding.rbac.authorization.k8s.io/stork-scheduler-role-binding created
deployment.apps/stork-scheduler created
serviceaccount/portworx-pvc-controller-account created
clusterrole.rbac.authorization.k8s.io/portworx-pvc-controller-role created
clusterrolebinding.rbac.authorization.k8s.io/portworx-pvc-controller-role-binding created
deployment.extensions/portworx-pvc-controller created
service/portworx-service created
serviceaccount/px-account created
clusterrole.rbac.authorization.k8s.io/node-get-put-list-role created
clusterrolebinding.rbac.authorization.k8s.io/node-role-binding created
namespace/portworx created
role.rbac.authorization.k8s.io/px-role created
rolebinding.rbac.authorization.k8s.io/px-role-binding created
daemonset.extensions/portworx created
serviceaccount/px-lh-account created
role.rbac.authorization.k8s.io/px-lh-role created
rolebinding.rbac.authorization.k8s.io/px-lh-role-binding created
service/px-lighthouse created
deployment.apps/px-lighthouse created
```

### List Portworx Pods
```
kubectl get pods -o wide -n kube-system -l name=portworx
NAME             READY     STATUS    RESTARTS   AGE       IP           NODE                                               NOMINATED NODE
portworx-5tvdh   0/1       Running   0          34s       10.142.0.3   gke-test-cluster-ryan-default-pool-ac8eed24-clh1   <none>
portworx-9mtvm   0/1       Running   0          34s       10.142.0.2   gke-test-cluster-ryan-default-pool-ac8eed24-ftf7   <none>
portworx-kk42h   0/1       Running   0          34s       10.142.0.4   gke-test-cluster-ryan-default-pool-ac8eed24-c38h   <none>
```

### When they are running, create access to `pxctl`
```
export PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[2].metadata.name}')

alias pxctl="kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl"
```

### Get pxctl status
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


## Expose LightHouse

```
gcloud compute firewall-rules create lighthouse-ingress-http \
  --allow tcp:32678 \
  --target-tags=portworx
```


Visit the following
```
echo http://"$(kubectl describe no <ANY_GKE_WORKER_NODE_NAME> | grep ExternalIP | awk '{print $2}')":32678

http://35.237.126.155:32678
```

Login
```
admin/Password1
```

Use the Cluster IP in the output below for `Cluster Endpoint` in the UI and click `Verify` then `Attach` 
```
kubectl get svc -n kube-system portworx-service
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
portworx-service   ClusterIP   10.15.245.175   <none>        9001/TCP   23h
```

