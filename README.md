# GKE Install

You *must* use `container-vm` Node image.  The Container-Optimized OS (cos) does not allow mount propagation.

There is now an Ubuntu image available but this hasn't been tested yet.

NOTE:  

## Create Cluster Reservation

```bash
$ storageos cluster create --size 3
2e59f369-bf84-4524-939b-b23615a8e1d4
```

## Manual Steps

On each cluster node:

```bash
sudo -i
mkdir /var/lib/storageos
mount --bind /var/lib/storageos /var/lib/storageos
mount --make-shared /var/lib/storageos
export CLUSTER_ID=2e59f369-bf84-4524-939b-b23615a8e1d4
export ADVERTISE_IP=`curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip`

docker run -d --name storageos \
    --restart=always \
    -e HOSTNAME \
    -e ADVERTISE_IP=${ADVERTISE_IP} \
    -e CLUSTER_ID=${CLUSTER_ID} \
    --net=host \
    --pid=host \
    --privileged \
    --cap-add SYS_ADMIN \
    --device /dev/fuse \
    -v /var/lib/storageos:/var/lib/storageos:rshared \
    -v /run/docker/plugins:/run/docker/plugins \
    storageos/node:0.8.1 server

curl -sSL https://github.com/storageos/go-cli/releases/download/0.0.13/storageos_linux_amd64 > /usr/local/bin/storageos
chmod +x /usr/local/bin/storageos

```

## Create LoadBalancer Service

This creates a load-balanced StorageOS API endpoint, required for the Kubernetes controller to provision new PVs.  *By default the API is accessible over the Internet so that the CLI can be used remotely.  Change this if this is not what you want.*

Since we're using node containers started outside of Kubernetes, we need to create the endpoints manually and not using selectors.

*IMPORTANT*:  Edit `02-service.yaml` and enter the node ip addresses in the `Endpoints` definition.

```bash
kubectl create -f 02-service.yaml
service "storageos-api" created
```

Verify:

```bash
$ kubectl get svc storageos-api
NAME            CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
storageos-api   10.55.253.205   35.197.193.90   5705:32200/TCP   3m
```

Then using the StorageOS CLI:

```bash
$ export STORAGEOS_HOST=35.197.193.90:5705
$ export STORAGEOS_USERNAME=storageos
$ export STORAGEOS_PASSWORD=storageos
$ storageos node ls
NAME                                                 ADDRESS             HEALTH              SCHEDULER           VOLUMES             TOTAL               USED                VERSION                 LABELS
gke-demo-storageos-node-default-pool-478bef52-29x1   10.154.0.7          Healthy 6 minutes   true                M: 0, R: 0          98.3 GiB            4.13%               38f3d80 (38f3d80 rev)
gke-demo-storageos-node-default-pool-478bef52-wljs   10.154.0.5          Healthy 6 minutes   false               M: 0, R: 0          98.3 GiB            4.19%               38f3d80 (38f3d80 rev)
gke-demo-storageos-node-default-pool-478bef52-xsdg   10.154.0.6          Healthy 6 minutes   false               M: 0, R: 0          98.3 GiB            4.22%               38f3d80 (38f3d80 rev)
```

## Create API Secret

The controller needs to know how to connect to the API.  First, encode the API endpoint:

```bash
$ echo -n "tcp://35.197.193.90:5705" | base64
dGNwOi8vMzUuMTk3LjE5My45MDo1NzA1
```

Replace the `apiAddress` field in `03-secret.yaml` with the encoded value.

Then create the secret:

```bash
$ kubectl create -f 03-secret.yaml
secret "storageos-secret" created
```

## Create StorageClass

```bash
$ kubectl create -f 04-storageclass.yaml
storageclass "fast" created
```

## Create PersistentVolumeClain

```bash
$ kubectl create -f 05-pvc.yaml
persistentvolumeclaim "redis" created
```

Verify:

```bash
kubectl get pvc
NAME       STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS   AGE
redis      Bound     pvc-134f88f1-7637-11e7-9753-42010a9a008d   5Gi        RWO           fast           27s
```

## Create Deployment

```bash
$ kubectl create -f 06-deployment.yaml
deployment "redis-deployment" created
```

Verify:

```bash
$ kubectl get pod redis
NAME      READY     STATUS    RESTARTS   AGE
redis     1/1       Running   0          5m
```

## Create Redis Service

```bash
$ kubectl create -f 07-redis-service.yaml
service "redis" created
```

Verify:

```bash
$kubectl get service redis
NAME      CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
redis     10.55.249.70   35.197.243.211   6379:30163/TCP   4m
```

## Run Benchmark

Create a batch job to run the benchmarks:

```bash
$ kubectl create -f 08-benchmark.yaml
job "bench" created
```

```bash
$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
bench     1         1            2m
```

```bash
$kubectl describe job
...
  5m		5m		1	job-controller			Normal		SuccessfulCreate	Created pod: bench-3sw6x
```

Get results:

```bash
kubectl logs bench-3sw6x
```

## Connect to Redis

All commands run from workstation.  Install `redis-cli` using `brew install redis`.

Verify Connection:

```bash
$ redis-cli -h 35.197.243.211 ping
PONG
```

Add data:

```bash
redis-cli -h 35.197.243.211 set date "`date`"
OK
```

Verify:

```bash
redis-cli -h 35.197.243.211 get date
"Thu Aug 17 13:09:16 BST 2017"
```

## Failover test

Get redis pod name:

```bash
kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
redis-deployment-3889241253-6kws3   1/1       Running   0          25m
```

Get node redis is running on:

```bash
$ kubectl describe pod redis-deployment-3889241253-6kws3
Name:		redis-deployment-3889241253-6kws3
Namespace:	default
Node:		gke-demo-storageos-node-default-pool-478bef52-xsdg/10.154.0.6
...
```

```bash
kubectl drain --force --ignore-daemonsets gke-demo-storageos-node-default-pool-478bef52-xsdg
```

```bash
kubectl uncordon gke-demo-storageos-node-default-pool-478bef52-xsdg
```
