# OpenEBS cStor install for Free IBM Cloud (for Windows PowerShell user)

## Objectives

Ensure to create OpenEBS cStor volumes on a single node with a small capacity environment (without replication).

> ⚠️ This document is written for Windows Powershell users.
You only need Windows to run it. because you may not have the WSL environment.

## Prerequisites

- Public/Private Clouds
  - If you want to use Free IBM Kubernetes cluster,  
    See [Free IBM Cloud Kubernetes hosting for beginners](https://github.com/ymikasa/free-ibm-cloud-kubernetes-hosting-for-beginner)

## Deploy OpenEBS operator

```powershell
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

### Check running pods.

```powershell
kubectl -n openebs get po -w
```
```text
NAME                                         READY STATUS  RESTARTS AGE
maya-apiserver-68f79c6c8-r45lb               1/1   Running 3        9h
openebs-admission-server-6fdbbff64c-hxh4f    1/1   Running 0        9h
openebs-localpv-provisioner-6648755679-ptv8x 1/1   Running 0        9h
openebs-ndm-operator-585d97db4b-qp8sg        1/1   Running 1        9h
openebs-ndm-z586v                            1/1   Running 0        6h51m
openebs-provisioner-5d8fccf8cc-w94p9         1/1   Running 0        9h
openebs-snapshot-operator-658ccc99b7-v99th   2/2   Running 0        9h
```

### Check all StorageClass.

```powershell
kubectl get sc
```
```text
NAME                      PROVISIONER                                              RECLAIMPOLICY VOLUMEBINDINGMODE    ALLOWVOLUMEEXPANSION AGE
openebs-device            openebs.io/local                                         Delete        WaitForFirstConsumer false                104s
openebs-hostpath          openebs.io/local                                         Delete        WaitForFirstConsumer false                104s
openebs-jiva-default      openebs.io/provisioner-iscsi                             Delete        Immediate            false                106s
openebs-snapshot-promoter volumesnapshot.external-storage.k8s.io/snapshot-promoter Delete        Immediate            false                105s
```

### Check BlockDevice

2 block devices attached to a worker node.

```powershell
kubectl -n openebs get blockdevice
```
```text
NAME                                         NODENAME      SIZE       CLAIMSTATE STATUS AGE
blockdevice-92db3a5e20e7f4f40e74889670b04ec6 10.144.183.23 67108864   Unclaimed  Active 7m12s
blockdevice-d47004dbc0be5ca02f8edf3307d229cd 10.144.183.23 2147483648 Unclaimed  Active 7m12s
```

### Check detail of block devices

```powershell
kubectl -n openebs describe blockdevice blockdevice-92db3a5e20e7f4f40e74889670b04ec6
```
```yaml
Name: blockdevice-92db3a5e20e7f4f40e74889670b04ec6
...
Metadata:
Spec:
  Capacity:
    Storage: 67108864
  Details:
    Device Type: disk
    Drive Type: SSD
  Node Attributes:
    Node Name: 10.144.183.23
  Path: /dev/xvdh
...
```

```powershell
kubectl -n openebs describe blockdevice blockdevice-d47004dbc0be5ca02f8edf3307d229cd
```
```yaml
Name: blockdevice-d47004dbc0be5ca02f8edf3307d229cd
...
Metadata:
Spec:
  Capacity:
    Storage: 2147483648
  Details:
    Device Type: disk
    Drive Type: SSD
  Path: /dev/xvdb
...
```

### Block storage capacity

| Device | Capacity |
| - | - |
|/dev/xvdh | 64MB |
|/dev/xvdb | 2GB |

> ⚠️ Free IBM Kubernetes Cluster node has only 2GB attached storage...  
> ⚠️ Free IBM Kubernetes Cluster node can't any more attach storage.

Use block device /dev/xvdb

```powershell
$blockdevice="blockdevice-d47004dbc0be5ca02f8edf3307d229cd"
```

### Check default StoragePool

```poweshell
kubectl get sp
```
```text
NAME    AGE
default 7m36s
```

## Deploy StoragePoolClaim

Reduce memory requirements without replicas for a free IBM Cloud cluster node.

```powershell
@"
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk-pool
  annotations:
    cas.openebs.io/config: |
      - name: ReplicaCount
        value: "1"
      - name: PoolResourceRequests
        value: |-
            memory: 250Mi
      - name: PoolResourceLimits
        value: |-
            memory: 250Mi
spec:
  name: cstor-disk-pool
  type: disk
  poolSpec:
    poolType: striped
  blockDevices:
    blockDeviceList:
    - $blockdevice
"@ | Set-Content cstor-pool-config.yaml
kubectl apply -f cstor-pool-config.yaml
```

```powershell
kubectl -n openebs get po
```
```text
NAME                                         READY STATUS   RESTARTS AGE
cstor-disk-pool-fw3x-8446689d5-sjbh6         3/3   Running  0        8h
maya-apiserver-68f79c6c8-r45lb               1/1   Running  3        9h
openebs-admission-server-6fdbbff64c-hxh4f    1/1   Running  0        9h
openebs-localpv-provisioner-6648755679-ptv8x 1/1   Running  0        9h
openebs-ndm-operator-585d97db4b-qp8sg        1/1   Running  1        9h
openebs-ndm-z586v                            1/1   Running  0        6h51m
openebs-provisioner-5d8fccf8cc-w94p9         1/1   Running  0        9h
openebs-snapshot-operator-658ccc99b7-v99th   2/2   Running  0        9h
```

Pool pod "cstor-disk-pool" started.

## Create cStor StorageClass

Deploy StorageClass without replicas for a free IBM Cloud cluster node.

```powershell
@"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-sparse-sc
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-disk-pool"
      - name: ReplicaCount
        value: "1"
provisioner: openebs.io/provisioner-iscsi
"@ | Set-Content openebs-sparse-sc.yaml
kubectl apply -f openebs-sparse-sc.yaml
```

## Deploy application

The deployment will 2 volumes (PVC) requesting for nginx directories.  
[Deployment YAML](./cstor-example/base/cstor-example-deployment.yaml)

| Mount Path | Capaity (MB) |
|-|-|
| /usr/share/nginx/html | 1950 |
| /var/log/nginx | 50 |

```powershell
kubectl apply -k cstor-example\base\
```

### Check deployment status

```powershell
kubectl -n cstor-example describe po -l app=cstor-example
```
```text
...
Events:
  Type    Reason                 Age               From                    Message
  ----    ------                 ----              ----                    -------
  Warning FailedScheduling       16m (x2 over 16m) default-scheduler       persistentvolumeclaim "cstor-vol1" not found
  Normal  Scheduled              16m               default-scheduler       Successfully assigned cstor-example/cstor-example-55d5c67bdb-mpz2v to 10.144.183.23
  Normal  SuccessfulAttachVolume 16m               attachdetach-controller AttachVolume.Attach succeeded for volume "pvc-5542c90a-b1fd-4c8c-bfa8-e3ae9eebf105"
  Normal  SuccessfulAttachVolume 16m               attachdetach-controller AttachVolume.Attach succeeded for volume "pvc-def6f0a2-6a1f-4122-8ddc-6c40f2b14c6d"
  Normal  Pulling                14m               kubelet, 10.144.183.23  Pulling image "nginx"
  Normal  Pulled                 14m               kubelet, 10.144.183.23  Successfully pulled image "nginx"
  Normal  Created                14m               kubelet, 10.144.183.23  Created container nginx
  Normal  Started                14m               kubelet, 10.144.183.23  Started container nginx
```

> ⚠️ If you overcommit capacity size, it will fail at attaching volume.

### Check PVC, PV

PVC defined by [Deployment YAML](./cstor-example/base/cstor-example-deployment.yaml)

```powershell
kubectl -n cstor-example get pvc,pv
```
```text
NAME                             STATUS VOLUME                                   CAPACITY ACCESS MODES STORAGECLASS      AGE
persistentvolumeclaim/cstor-vol1 Bound  pvc-cb1fee9c-62ba-481e-b6dd-0c5c5f2d4cfe 1950Mi   RWO          openebs-sparse-sc 5h57m
persistentvolumeclaim/cstor-vol2 Bound  pvc-6c9d5a9a-cb82-4c4b-b4ca-c0227f243b8a 50Mi     RWO          openebs-sparse-sc 5h57m

NAME                                                      CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM                    STORAGECLASS      REASON AGE
persistentvolume/pvc-6c9d5a9a-cb82-4c4b-b4ca-c0227f243b8a 50Mi     RWO          Delete         Bound  cstor-example/cstor-vol2 openebs-sparse-sc        5h57m
persistentvolume/pvc-cb1fee9c-62ba-481e-b6dd-0c5c5f2d4cfe 1950Mi   RWO          Delete         Bound  cstor-example/cstor-vol1 openebs-sparse-sc        5h57m
```

### Check mount directories

```powershell
kubectl -n cstor-example exec -it (kubectl -n cstor-example get po -l app=cstor-example -o jsonpath='{.items[0].metadata.name}') -- df -h
```
```text
Filesystem Size Used Avail Use% Mounted on
overlay    97G  9.7G   87G  10% /
tmpfs      64M     0   64M   0% /dev
tmpfs      2.0G    0  2.0G   0% /sys/fs/cgroup
/dev/xvda2 97G  9.7G   87G  10% /etc/hosts
shm        64M     0   64M   0% /dev/shm
/dev/sda   976M 1.3M  959M   1% /var/log/nginx
/dev/sdb   976M 1.3M  959M   1% /usr/share/nginx/html
tmpfs      2.0G  16K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs      2.0G    0  2.0G   0% /proc/acpi
tmpfs      2.0G    0  2.0G   0% /proc/scsi
tmpfs      2.0G    0  2.0G   0% /sys/firmware
```
Storage attached /dev/sda, /dev/sdb from the storage pool.

## Delete application

```powershell
kubectl delete -k cstor-example\base\
```
