---
title: "Persistent Storage Configuration"
keywords: 'kubernetes, docker, kubesphere, storage, volume, PVC'
description: 'Persistent Storage Configuration'

linkTitle: "Persistent Storage Configuration"
weight: 2140
---
## Overview
Persistence volume is **Must** for Kubesphere. So before installation of Kubesphere,  one **default** 
[StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) and corresponding storage plugin should be installed on the Kubernetes cluster.
As different users may choose different storage plugin, [KubeKey](https://github.com/kubesphere/kubekey) supports to install storage plugin by the way of 
[add-on](https://github.com/kubesphere/kubekey/blob/v1.0.0/docs/addons.md). This passage will introduce add-on configuration for some mainly used storage plugin.

## QingCloud-CSI
[QingCloud-CSI](https://github.com/yunify/qingcloud-csi) plugin implements an interface between CSI-enabled Container Orchestrator (CO) and the disk of QingCloud. 
Here is a helm-chart example of installing by KubeKey add-on. 
```bash
addons:
- name: csi-qingcloud
  namespace: kube-system
  sources:
    chart:
      name: csi-qingcloud
      repo: https://charts.kubesphere.io/test
      values:
      - config.qy_access_key_id=SHOULD_BE_REPLACED
      - config.qy_secret_access_key=SHOULD_BE_REPLACED
      - config.zone=SHOULD_BE_REPLACED
      - sc.isDefaultClass=true
```
For more information about QingCloud, see [QingCloud](https://www.qingcloud.com/). 
For more chart values, see [configuration](https://github.com/kubesphere/helm-charts/tree/master/src/test/csi-qingcloud#configuration).

## NFS-client
The [nfs-client-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client) is an automatic provisioner for Kubernetes that uses your 
*already configured* NFS server, dynamically creating Persistent Volumes.
Hear is a helm-chart example of installing by KubeKey add-on.
```yaml
addons:
- name: nfs-client
  namespace: kube-system
  sources:
    chart:
      name: nfs-client-provisioner
      repo: https://charts.kubesphere.io/main
      values:
      - nfs.server=SHOULD_BE_REPLACED
      - nfs.path=SHOULD_BE_REPLACED
      - storageClass.defaultClass=true
```
For more chart values, see [configuration](https://github.com/kubesphere/helm-charts/tree/master/src/main/csi-nfs-provisioner#configuration)

## Ceph RBD
Ceph RBD is an in-tree storage plugin on Kubernetes. As **hyperkube** images were [deprecated since 1.17](https://github.com/kubernetes/kubernetes/pull/85094), 
**KubeKey** will never use **hyperkube** images. So in-tree Ceph RBD may not work on Kubernetes installed by **KubeKey**. 
If you work with 14.0.0(Nautilus)+ Ceph Cluster, we appreciate you to use [Ceph CSI](#Ceph CSI). 
Meanwhile you could use [rbd provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd) as substitute, which is same format with in-tree Ceph RBD. 
Here is an example of rbd-provisioner. 
```yaml
- name: rbd-provisioner
  namespace: kube-system
  sources:
    chart:
      name: rbd-provisioner
      repo: https://charts.kubesphere.io/test
      values:
      - ceph.mon=SHOULD_BE_REPLACED # like 192.168.0.10:6789
      - ceph.adminKey=SHOULD_BE_REPLACED
      - ceph.userKey=SHOULD_BE_REPLACED
      - sc.isDefault=true
```
For more values, see [configuration](https://github.com/kubesphere/helm-charts/tree/master/src/test/rbd-provisioner#configuration)

## Ceph CSI
[Ceph-CSI](https://github.com/ceph/ceph-csi) contains Ceph Container Storage Interface (CSI) driver for RBD, CephFS. It will be substitute for [Ceph-RBD](#Ceph RBD) in the future.
Ceph CSI should be installed on v1.14.0+ Kubernetes, and work with 14.0.0(Nautilus)+ Ceph Cluster.
For details about compatibility, see [support matrix](https://github.com/ceph/ceph-csi#support-matrix).	Here is an example of installing ceph-csi-rbd by **KubeKey** add-on.
```yaml
csiConfig:
  - clusterID: "cluster1"
    monitors:
      - SHOULD_BE_REPLACED 
```
Save the YAML file of ceph config in local, **/root/ceph-csi-config.yaml** for example. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: kube-system
stringData:
  userID: admin
  userKey: SHOULD_BE_REPLACED
  encryptionPassphrase: test_passphrase
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
   annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
     storageclass.kubesphere.io/supported-access-modes: '["ReadWriteOnce","ReadOnlyMany","ReadWriteMany"]'
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: "cluster1"
   pool: rbd
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: kube-system
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: kube-system
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: kube-system
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
```
Save the YAML file of StorageClass in local, **/root/ceph-csi-rbd-sc.yaml** for example. The add-on configuration could be set like:

```yaml
addons: 
- name: ceph-csi-rbd
  namespace: kube-system
  sources:
    chart:
      name: ceph-csi-rbd
      repo: https://ceph.github.io/csi-charts
      values: /root/ceph-csi-config.yaml
- name: ceph-csi-rbd-sc
  sources:
    yaml:
      path:
      - /root/ceph-csi-rbd-sc.yaml
```
For more information, see [chart for ceph-csi-rbd](https://github.com/ceph/ceph-csi/tree/master/charts/ceph-csi-rbd)


## Glusterfs
Glusterfs is an in-tree storage plugin on Kubernetes, only StorageClass is need to been installed. 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
type: kubernetes.io/glusterfs
data:
  key: SHOULD_BE_REPLACED
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/supported-access-modes: '["ReadWriteOnce","ReadOnlyMany","ReadWriteMany"]'
  name: glusterfs
parameters:
  clusterid: SHOULD_BE_REPLACED 
  gidMax: "50000"
  gidMin: "40000"
  restauthenabled: "true"
  resturl: SHOULD_BE_REPLCADED # like "http://192.168.0.14:8080"
  restuser: admin
  secretName: heketi-secret
  secretNamespace: kube-system
  volumetype: SHOULD_BE_REPLACED # like replicate:2
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```
For detailed information, see [configuration](https://kubernetes.io/docs/concepts/storage/storage-classes/#glusterfs) 

Save the YAML file of StorageClass in local, **/root/glusterfs-sc.yaml** for example. The add-on configuration could be set like:
```bash
- addon
- name: glusterfs
  sources:
    yaml:
      path:
      - /root/glusterfs-sc.yaml
```

## OpenEBS/LocalVolumes
[OpenEBS](https://github.com/openebs/openebs) Dynamic Local PV provisioner can create Kubernetes Local Persistent Volumes using a unique 
HostPath (directory) on the node to persist data. It's very convenient for experience KubeSphere when you has no special storage system.
If no default StorageClass configured of **KubeKey** add-on, OpenEBS/LocalVolumes will be installed.

## Multi-Storage
If you intend to install more than one storage plugins, remind to set only one to be default. 
Otherwise [ks-installer](https://github.com/kubesphere/ks-installer) will be confused about which StorageClass to use. 
