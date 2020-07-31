# OpenEBS cStor install for Free IBM Cloud (for Windows PowerShell user) {ignore=true}

## Objectives {ignore=true}

Ensure to create OpenEBS cStor volumes on a single node with a small capacity environment (without replication).

> ⚠️ This document is written for Windows Powershell users.
You only need Windows to run it. because you may not have the WSL environment.

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Prerequisites](#prerequisites)
- [Deploy OpenEBS operator](#deploy-openebs-operator)
  - [Check running pods.](#check-running-pods)
  - [Check all StorageClass.](#check-all-storageclass)
  - [Check BlockDevice](#check-blockdevice)
  - [Check detail of block devices](#check-detail-of-block-devices)
  - [Block storage capacity](#block-storage-capacity)
  - [Check default StoragePool](#check-default-storagepool)
- [Deploy StoragePoolClaim](#deploy-storagepoolclaim)
- [Create cStor StorageClass](#create-cstor-storageclass)
- [Deploy application](#deploy-application)
  - [Check deployment status](#check-deployment-status)
  - [Check PVC, PV](#check-pvc-pv)
  - [Check mount directories](#check-mount-directories)
- [Delete application](#delete-application)

<!-- /code_chunk_output -->

## Prerequisites

<details><summary>Public/Private Clouds</summary>

  - If you want to use Free IBM Kubernetes cluster,  
    See [Free IBM Cloud Kubernetes hosting for beginners](https://github.com/ymikasa/free-ibm-cloud-kubernetes-hosting-for-beginner)

</details>

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
<details><summary>Name: blockdevice-92db3a5e20e7f4f40e74889670b04ec6</summary>

```yaml
Name:         blockdevice-92db3a5e20e7f4f40e74889670b04ec6
Namespace:    openebs
Labels:       kubernetes.io/hostname=10.144.183.23
              ndm.io/blockdevice-type=blockdevice
              ndm.io/managed=true
Annotations:  <none>
API Version:  openebs.io/v1alpha1
Kind:         BlockDevice
Metadata:
  Creation Timestamp:  2020-07-30T20:39:40Z
  Generation:          3
  Managed Fields:
    API Version:  openebs.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:kubernetes.io/hostname:
          f:ndm.io/blockdevice-type:
          f:ndm.io/managed:
      f:spec:
        .:
        f:capacity:
          .:
          f:logicalSectorSize:
          f:physicalSectorSize:
          f:storage:
        f:details:
          .:
          f:compliance:
          f:deviceType:
          f:driveType:
          f:firmwareRevision:
          f:hardwareSectorSize:
          f:logicalBlockSize:
          f:model:
          f:physicalBlockSize:
          f:serial:
          f:vendor:
        f:devlinks:
        f:filesystem:
        f:nodeAttributes:
          .:
          f:nodeName:
        f:partitioned:
        f:path:
      f:status:
        .:
        f:claimState:
        f:state:
    Manager:         ndm
    Operation:       Update
    Time:            2020-07-30T22:48:21Z
  Resource Version:  131113
  Self Link:         /apis/openebs.io/v1alpha1/namespaces/openebs/blockdevices/blockdevice-92db3a5e20e7f4f40e74889670b04ec6
  UID:               25644baf-e113-4730-82f0-131db07c8671
Spec:
  Capacity:
    Logical Sector Size:   512
    Physical Sector Size:  512
    Storage:               67108864
  Details:
    Compliance:
    Device Type:           disk
    Drive Type:            SSD
    Firmware Revision:
    Hardware Sector Size:  512
    Logical Block Size:    512
    Model:
    Physical Block Size:   512
    Serial:
    Vendor:
  Devlinks:
  Filesystem:
  Node Attributes:
    Node Name:  10.144.183.23
  Partitioned:  No
  Path:         /dev/xvdh
Status:
  Claim State:  Unclaimed
  State:        Active
Events:         <none>
```

</details>

```powershell
kubectl -n openebs describe blockdevice blockdevice-d47004dbc0be5ca02f8edf3307d229cd
```
<details><summary>Name: blockdevice-d47004dbc0be5ca02f8edf3307d229cd</summary>

```yaml
Name:         blockdevice-d47004dbc0be5ca02f8edf3307d229cd
Namespace:    openebs
Labels:       kubernetes.io/hostname=10.144.183.23
              ndm.io/blockdevice-type=blockdevice
              ndm.io/managed=true
Annotations:  <none>
API Version:  openebs.io/v1alpha1
Kind:         BlockDevice
Metadata:
  Creation Timestamp:  2020-07-30T20:39:40Z
  Finalizers:
    openebs.io/bd-protection
  Generation:  4
  Managed Fields:
    API Version:  openebs.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
      f:spec:
        f:claimRef:
          .:
          f:apiVersion:
          f:kind:
          f:name:
          f:namespace:
          f:resourceVersion:
          f:uid:
      f:status:
        f:claimState:
    Manager:      ndo
    Operation:    Update
    Time:         2020-07-30T21:11:44Z
    API Version:  openebs.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:kubernetes.io/hostname:
          f:ndm.io/blockdevice-type:
          f:ndm.io/managed:
      f:spec:
        .:
        f:capacity:
          .:
          f:logicalSectorSize:
          f:physicalSectorSize:
          f:storage:
        f:details:
          .:
          f:compliance:
          f:deviceType:
          f:driveType:
          f:firmwareRevision:
          f:hardwareSectorSize:
          f:logicalBlockSize:
          f:model:
          f:physicalBlockSize:
          f:serial:
          f:vendor:
        f:devlinks:
        f:filesystem:
        f:nodeAttributes:
          .:
          f:nodeName:
        f:partitioned:
        f:path:
      f:status:
        .:
        f:state:
    Manager:         ndm
    Operation:       Update
    Time:            2020-07-30T22:48:21Z
  Resource Version:  131115
  Self Link:         /apis/openebs.io/v1alpha1/namespaces/openebs/blockdevices/blockdevice-d47004dbc0be5ca02f8edf3307d229cd
  UID:               43981f35-7649-46ba-a0cb-d2d40b734928
Spec:
  Capacity:
    Logical Sector Size:   512
    Physical Sector Size:  512
    Storage:               2147483648
  Claim Ref:
    API Version:       openebs.io/v1alpha1
    Kind:              BlockDeviceClaim
    Name:              bdc-43981f35-7649-46ba-a0cb-d2d40b734928
    Namespace:         openebs
    Resource Version:  106084
    UID:               e2e4f66e-c727-4f09-aac7-82d4c3090087
  Details:
    Compliance:
    Device Type:           disk
    Drive Type:            SSD
    Firmware Revision:
    Hardware Sector Size:  512
    Logical Block Size:    512
    Model:
    Physical Block Size:   512
    Serial:
    Vendor:
  Devlinks:
  Filesystem:
  Node Attributes:
    Node Name:  10.144.183.23
  Partitioned:  No
  Path:         /dev/xvdb
Status:
  Claim State:  Claimed
  State:        Active
Events:         <none>
```

</details>

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
Filesystem      Size  Used Avail Use% Mounted on
overlay          97G   11G   87G  11% /
tmpfs            64M     0   64M   0% /dev
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/xvda2       97G   11G   87G  11% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/sdb         45M  810K   43M   2% /var/log/nginx
/dev/sda        1.9G  2.9M  1.9G   1% /usr/share/nginx/html
tmpfs           2.0G   16K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           2.0G     0  2.0G   0% /proc/acpi
tmpfs           2.0G     0  2.0G   0% /proc/scsi
tmpfs           2.0G     0  2.0G   0% /sys/firmware
```
Storage attached /dev/sda, /dev/sdb from the storage pool.

## Delete application

```powershell
kubectl delete -k cstor-example\base\
```
