# Perf testing Postgres

Links
- https://cloud.google.com/compute/docs/disks/performance
- https://docs.portworx.com/maintain/performance/tuning.html
- https://cloud.google.com/community/tutorials/setting-up-postgres


## Environment

- Machine Type: `n1-standard-4`
- PD: 500GB SSD

For Portworx, make sure and set journal and lets us 1000GB SSD pd's.

```
args:
            ["-k", "etcd:http://px-etcd1.portworx.com:2379,etcd:http://px-etcd2.portworx.com:2379,etcd:http://px-etcd3.portworx.com:2379", "-c", "px-cluster-e1f7a3b4-0cfc-4e09-a7df-7bf4663845c1", "-s", "type=pd-ssd,size=500", "-secret_type", "k8s",  
             "-j", "auto","-max_storage_nodes_per_zone", "3", "-x", "kubernetes"]
```

## Portworx PX Env

### Storages Class and replica settings
```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
    name: px-postgres-sc-[repl3, repl2, repl1]
provisioner: kubernetes.io/portworx-volume
parameters:
   repl: ["3" or "2" or "1"]

---

##### Portworx persistent volume claim
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: postgres-data
   annotations:
     volume.beta.kubernetes.io/storage-class: px-postgres-sc-[repl3, repl2, repl1]
spec:
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 10Gi
```

### Create repl3 postgres and test

```
kubectl create -f specs/postgres-9.6-px-repl3.yaml
```

Update io profile
```
pxctl volume update --io_profile=db pvc-caab3706-d0be-11e8-ab93-42010a8e000e
Update Volume: Volume update successful for volume pvc-caab3706-d0be-11e8-ab93-42010a8e000e
```

### Create repl2 postgres and test

```
kubectl create -f specs/postgres-9.6-px-repl2.yaml
```

Update io profile
```
pxctl volume update --io_profile=db pvc-caab3706-d0be-11e8-ab93-42010a8e000e
Update Volume: Volume update successful for volume pvc-caab3706-d0be-11e8-ab93-42010a8e000e
```

### Create repl1 postgres and test

```
kubectl create -f specs/postgres-9.6-px-repl1.yaml
```

Update io profile
```
pxctl volume update --io_profile=db pvc-caab3706-d0be-11e8-ab93-42010a8e000e
Update Volume: Volume update successful for volume pvc-caab3706-d0be-11e8-ab93-42010a8e000e
```

## Stand Alone Instance

The data dir will be
`data_directory = '/var/lib/postgresql/9.5/main'         # use data in another directory`

```
ryan_wallner@instance-1:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   0  500G  0 disk 
sda      8:0    0   10G  0 disk 
└─sda1   8:1    0   10G  0 part /
```

```
ryan_wallner@instance-1:~$  mkfs.ext4 /dev/sdb
mke2fs 1.42.13 (17-May-2015)
Discarding device blocks: done                            
Creating filesystem with 131072000 4k blocks and 32768000 inodes
Filesystem UUID: a0cfdf81-29cf-40df-8dcb-b9ab08543396
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
        102400000
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done  
```

Mount the pd
```
root@instance-1:~# mkdir /var/lib/postgresql/
root@instance-1:~# mount /dev/sdb /var/lib/postgresql/
```

```
sudo apt-get update -y
sudo apt-get -y install postgresql postgresql-client postgresql-contrib
```

```
service postgresql status
● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Mon 2018-10-15 20:32:14 UTC; 41s ago
 Main PID: 2044 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/postgresql.service
Oct 15 20:32:14 ryantesting systemd[1]: Starting PostgreSQL RDBMS...
Oct 15 20:32:14 ryantesting systemd[1]: Started PostgreSQL RDBMS.
```

```
root@ryantesting:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            7.4G     0  7.4G   0% /dev
tmpfs           1.5G   11M  1.5G   1% /run
/dev/sda1       9.8G  1.1G  8.2G  12% /
tmpfs           7.4G  4.0K  7.4G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           7.4G     0  7.4G   0% /sys/fs/cgroup
/dev/sdb        492G  110M  467G   1% /var/lib/postgresql
```

Load the DB for perf
```
root@ryantesting:~# su postgres
pgbench -i -s 70 postgres
```

Rand Write
```
pgbench -c 4 -j 2 -T 600 postgres
```

Write
```
pgbench -c 4 -j 2 -T 600 -S postgres
```

Read
```
pgbench -c 4 -j 2 -T 600 -N postgres
```



