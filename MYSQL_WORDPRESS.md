# Create Wordpress MySQL

This guide will: 

- create wordpress and mysql.
- add some data

(TODO)
- take 3dsnapshot of wordpress + mysql
- cloudsnap
- backup
- restore

## Create volumes

`kubectl create -f specs/wordpress-mysql-vols.yaml`

> Note, there is 2 volumes in then above file, one uses `io_profile=cms` and is `shared` for wordpress and the other using 3 replicas for MySQL and sets a `snap_schedule` to snapshot every 30 minutes and keep the last 5.

## Create MySQL Pass

`kubectl create secret generic mysql-pass --from-file=files/mysqlpasswd.txt`

## Create MySQL

`kubectl create -f specs/wp-mysql.yaml`

## Create Wordpress

`kubectl create -f specs/wordpress.yaml`

## View DB and WP

```
 kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
wordpress-566c9858f6-fvl22        1/1       Running   0          1m
wordpress-566c9858f6-jjrzv        1/1       Running   0          1m
wordpress-566c9858f6-wx5m9        1/1       Running   0          1m
wordpress-mysql-f7df569c8-qmdz6   1/1       Running   0          3m
```

## Get Wordpress SVC
```
kubectl get svc wordpress
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
wordpress   NodePort   10.15.246.184   <none>        80:30303/TCP   2m
```

## Open Access to WP

```
gcloud container get-server-config

gcloud compute firewall-rules create wordpress-ingress \
  --allow tcp:30303 \
  --target-tags=portworx
```

## Get an external IP of one of the worker nodes
```
kubectl describe no gke-test-cluster-ryan-default-pool-a072badd-bztf | grep ExternalIP
  ExternalIP:  35.237.126.155
```

## Access Wordpress
```
http://35.237.126.155:30303/wp-admin/install.php
```

## Install Wordpress and Add Data

1. Run through the install and set up your username and password
2. create your first post. (example: http://35.237.126.155:30303/2018/10/11/first-post/)


### IO Profiles
- https://docs.portworx.com/maintain/performance/tuning.html

```
CMS
This is useful for content management systems, like WordPress. This option applies to a PX shared (global namespace) volume. It implements an attribute cache and supports async writes. This increases the PX memory footprint by 100MB. Use io_profile=cms.
```


## Volume Managment

View Snapshots from Schedules

`pxctl volume list -ss`

View snapshot schedule in an `inspect`

> Notice: `periodic 30m0s,keep last 5`

```
pxctl volume inspect 989013518918507817
Volume       :  989013518918507817
        Name                     :  pvc-26e445e2-d0a0-11e8-9abd-42010a8e00b9
        Size                     :  2.0 GiB
        Format                   :  ext4
        HA                       :  3
        IO Priority              :  MEDIUM
        Creation time            :  Oct 15 17:31:30 UTC 2018
        Snapshot                 :  periodic 30m0s,keep last 5
        Shared                   :  no
        Status                   :  none
        State                    :  detached
        Labels                   :  namespace=default,pvc=mysql-pvc-1
        Reads                    :  0
        Reads MS                 :  0
        Bytes Read               :  0
        Writes                   :  0
        Writes MS                :  0
        Bytes Written            :  0
        IOs in progress          :  0
        Bytes used               :  65 MiB
        Replica sets on nodes:
                Set 0
                  Node           : 10.142.0.2 (Pool 0)
                  Node           : 10.142.0.4 (Pool 0)
                  Node           : 10.142.0.3 (Pool 0)
        Replication Status       :  Detached
```
