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



