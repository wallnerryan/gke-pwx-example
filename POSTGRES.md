
# Postgres on GKE with PWX

1. create postgres
2. setup cloudsnap

(TODO)

3. create cloudsnap to google object storage
4. restore 

## Create Postgres

```
kubectl create -f  specs/postgres-px.yaml
storageclass.storage.k8s.io "px-postgres-sc" created
persistentvolumeclaim "postgres-data" created
configmap "example-config" created
deployment.extensions "postgres" created
```

## Setup CloudSnaps

```
gcloud iam service-accounts create portworxsa \
    --display-name "portworx-sa"
```

```
gcloud projects add-iam-policy-binding portworx-eng \
    --member serviceAccount:portworxsa@portworx-eng.iam.gserviceaccount.com \
    --role roles/compute.admin \
    --role roles/storage.admin
```

```
gcloud iam service-accounts keys create files/gcs-key.json \
  --iam-account portworxsa@portworx-eng.iam.gserviceaccount.com
```

```
kubectl cp files/gcs-key.json kube-system/$PX_POD:/etc/pwx/gcs-key.json -n kube-system
```

```
pxctl credentials create --provider google --google-project-id portworx-eng --google-json-key-file /etc/pwx/gcs-key.json
Credentials created successfully, UUID:59d1f0ea-9322-446f-b4db-10c918664af9
```

## Create cloudsnap 

```
kubectl create -f specs/cloudsnap-postgres.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io "postgres-data" created
```

## View Snapshot

```
pxctl volume list -s
ID			NAME												SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	STATUS		HA-STATE
759039737635622804	pvc-9dd4e6b0-cd92-11e8-8281-42010a8e0115_394637582233985887_clmanual_2018-10-11T20-37-21	5 GiB	1	no	no		LOW		none - detached	Detached
```

or via cloudsnap

```
pxctl cloudsnap list
SOURCEVOLUME						SOURCEVOLUMEID			CLOUD-SNAP-ID										CREATED-TIME				TYPE		STATUS
pvc-9dd4e6b0-cd92-11e8-8281-42010a8e0115		394637582233985887		d5b722d7-679d-4cbb-8ade-188fbbf1e9a6/394637582233985887-759039737635622804		Thu, 11 Oct 2018 20:37:25 UTC		Manual		Done
```

## Make sure its done
```
pxctl cloudsnap status
SOURCEVOLUME		STATE		NODE		BYTES-PROCESSED	TIME-ELAPSED	COMPLETED
394637582233985887	Backup-Done	10.142.0.3	260491757	3.832537088s	Thu, 11 Oct 2018 20:37:28 UTC
```
