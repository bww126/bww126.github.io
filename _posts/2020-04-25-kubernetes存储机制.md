---
layout:     post
title:      kubernetes存储机制
subtitle:   PV PVC StorageClass
date:       2020-04-25
author:     bww126
header-img: img/bg/gray-bg.PNG
catalog: true
tags:
    - kubernetes
---

## 1. 概述

有状态的容器应用通常需要可靠的存储来保存重要数据，以便pod重启后依然可以使用之前的数据。Kubernetes提供了PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 两种资源来实现对存储的管理。PV是对底层网络共享存储的抽象，而PVC是用户对于存储资源的一个“申请”，会”消费“PV资源。PV和PVC可以将pod与存储卷解耦，pod不需要去关心底层存储，直接使用PVC即可实现数据的持久化。

## 2. PV 介绍

pv － 持久化卷， 支持本地存储和网络存储，demo如下：

```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv0003
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    storageClassName: slow
    nfs:
      path: /tmp
      server: 172.17.0.2
```

##### 存储能力（Capacity）

最新发行版本1.18仅支持对存储空间的设置。

##### 访问模式（Access Modes）

访问模式有以下三种，PV只能使用其中一种，多种访问方式不能同时生效。

- ReadWriteOnce –  可读可写，但只支持被单个 Pod 挂载。
- ReadOnlyMany – 可以以只读的方式被多个 Pod 挂载。
- ReadWriteMany – 这种存储可以以读写的方式被多个 Pod 共享。

不是每一种存储类型都支持三种方式，常见的iSCSI与FC只支持前两种，而NFS与CephFS支持多节点读写，下表描述了不同存储提供者支持的访问模式。

| Volume Plugin        | ReadWriteOnce         | ReadOnlyMany          | ReadWriteMany                      |
| :------------------- | :-------------------- | :-------------------- | :--------------------------------- |
| AWSElasticBlockStore | ✓                     | -                     | -                                  |
| AzureFile            | ✓                     | ✓                     | ✓                                  |
| AzureDisk            | ✓                     | -                     | -                                  |
| CephFS               | ✓                     | ✓                     | ✓                                  |
| Cinder               | ✓                     | -                     | -                                  |
| CSI                  | depends on the driver | depends on the driver | depends on the driver              |
| FC                   | ✓                     | ✓                     | -                                  |
| FlexVolume           | ✓                     | ✓                     | depends on the driver              |
| Flocker              | ✓                     | -                     | -                                  |
| GCEPersistentDisk    | ✓                     | ✓                     | -                                  |
| Glusterfs            | ✓                     | ✓                     | ✓                                  |
| HostPath             | ✓                     | -                     | -                                  |
| iSCSI                | ✓                     | ✓                     | -                                  |
| Quobyte              | ✓                     | ✓                     | ✓                                  |
| NFS                  | ✓                     | ✓                     | ✓                                  |
| RBD                  | ✓                     | ✓                     | -                                  |
| VsphereVolume        | ✓                     | -                     | - (works when Pods are collocated) |
| PortworxVolume       | ✓                     | -                     | ✓                                  |
| ScaleIO              | ✓                     | ✓                     | -                                  |
| StorageOS            | ✓                     | -                     | -                                  |

##### 存储类型（Class）

通过设置storageClassName参数来指定存储的类别，只有请求该类别的PVC才可以进行绑定。

##### 四个阶段（Phase）

- Available：资源可用，尚未被PVC声明绑定 。
- Bound：已被声明绑定。
- Released：释放状态，绑定的申明已被删除，但还未被重新声明。
- Failed：自动回收失败。

##### 回收策略（Reclaim  Policy）

- Retain：保留回收策略，允许手动回收资源，删除PVC时，状态由Bound变为Released。
- Recycle：基本的数据擦除(`rm -rf /thevolume/*`)，删除PVC时，状态由Bound变为Available。
- Delete：对于支持删除回收策略的卷插件，删除操作将从 Kubernetes 中删除 `PersistentVolume` 对象，并删除外部基础架构（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中的关联存储资产。

## 3. PVC 介绍

PVC参数设置主要包括访问模式、卷模式、资源请求、PV选择条件、存储类别等。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

- **accessModes**：可以设置三种访问方式，与PV的设置相同。
- **volumeMode**：声明使用与卷相同的约定，指示将卷作为文件系统或块设备使用。
- **resources**：仅支持requests.storage。
- **selector**：可以使用matchLabels和matchExpressions，只有标签与选择器匹配的卷可以绑定到声明，如果两者都设置了，则需要同时满足才能完成匹配。
- **storageClassName**：指定的 StorageClass的名称来请求特定类别的PV，只有所请求的类与 PVC 具有相同 storageClassName 的 PV 才能绑定到 PVC。

> PVC 不一定要请求类。其 `storageClassName` 设置为 `""` 的 PVC 始终被解释为没有请求类的 PV，因此只能绑定到没有类的 PV（没有注解或 `""`）。没有 `storageClassName` 的 PVC 根据是否打开[`DefaultStorageClass` 准入控制插件](https://kubernetes.io/docs/admin/admission-controllers/#defaultstorageclass)，集群对其进行不同处理。
>
> - 如果打开了准入控制插件，管理员可以指定一个默认的 `StorageClass`。所有没有 `StorageClassName` 的 PVC 将被绑定到该默认的 PV。通过在 `StorageClass` 对象中将注解 `storageclass.kubernetes.io/is-default-class` 设置为 “true” 来指定默认的 `StorageClass`。如果管理员没有指定缺省值，那么集群会响应 PVC 创建，就好像关闭了准入控制插件一样。如果指定了多个默认值，则准入控制插件将禁止所有 PVC 创建。
> - 如果准入控制插件被关闭，则没有默认 `StorageClass` 的概念。所有没有 `storageClassName` 的 PVC 只能绑定到没有类的 PV。在这种情况下，没有 `storageClassName` 的 PVC 的处理方式与 `storageClassName` 设置为 `""` 的 PVC 的处理方式相同。

## 4. StorageClass

动态卷供给是Kubernetes存储机制中一个重要的功能，这一功能允许用户按需创建存储卷。在没有这种能力之前，如果需要使用一个PVC就必须手动去创建一个PV，这在很多使用场景下是不合适的。StorageClass提供了一种描述存储类的方法，不同的存储类可能会映射到不同的服务质量等级和备份策略或其他策略等。

#### 4.1 关键配置参数

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

- **Provisioner（存储分配器**）
provisioner用来指定存储资源的提供者，可以看作是后端存储驱动，该字段为必填。
- **Parameters（参数）**
不同的provisioner具有不同的参数设置，当参数省略时，则会使用默认值。
- **Reclaim Policy（回收策略）**
reclaimPolicy指定创建的PV的回收策略，同 PV 的回收策略。如果 StorageClass 对象被创建时没有指定 reclaimPolicy ，它将默认为 Delete。
- **allowVolumeExpansion（允许卷扩展）**
PersistentVolume 可以配置为可扩展。将此功能设置为 `true` 时，允许用户通过编辑相应的 PVC 对象来调整卷大小。此功能只能用于扩容，不可用于缩小。
- **Mount Options（挂载选项）**
mountOptions在StorageClass和PV上都不会做验证，如果该参数填写无效，后端存储驱动不支持次挂载选项，则会导致PV分配失败。
- **volumeBindingMode（卷绑定模式）**
Immediate模式表示一旦创建PVC也就完成了PV的动态分配与绑定。WaitForFirstConsumer模式将延迟PV的分配和绑定，直达该PVC的pod被创建之后，才会去做Bound。

#### 4.2 nfs示例

要使用StroageClass就需先创建对应的Provisioner，比如对于nfs类型的存储驱动，其自动配置程序为nfs-client。

- **第一步：配置deployment创建nfs-client。**

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.151.30.57
            - name: NFS_PATH
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.151.30.57
            path: /data/k8s
```

当然在部署`nfs-client`之前，需要事先安装nfs服务器，例子中服务地址NFS_SERVER为10.151.30.57，共享数据NFS_PATH为/data/k8s。

- **第二步：创建serviceAccount、ClusterRole、ClusterRoleBinding**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

`nfs-client`使用了名为nfs-client-provisioner的sa，在这一步需要创建该sa，同时给为其绑定了名为nfs-client-provisioner-runner的ClusterRole，此ClusterRole申明了对PV增删改查、PVC操作等权限，具有该权限的sa可以完成对PV的动态创建。

- **第三步：创建StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  archiveOnDelete: "false"
```

需要注意的是provisioner参数要和provisioner的deployment的环境变量PROVISIONER_NAME值保持一致。参数archiveOnDelete字面意思为删除时是否存档，false表示不存档，即删除数据；没有指定archiveOnDelete参数或者值为true，表明需要删除时存档，即将oldPath重命名，命名格式为`"archived-"+pvName`。

- **第四步：使用PVC**

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc1
  annotations:
    volume.beta.kubernetes.io/storage-class: "course-nfs-storage"  
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

或

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc2
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: course-nfs-storage   
```

## 参考文章

[Persistent Volume（持久化卷）](https://jimmysong.io/kubernetes-handbook/concepts/persistent-volume.html)
[存储介绍]([https://github.com/hackstoic/kubernetes_practice/blob/master/%E5%AD%98%E5%82%A8.md](https://github.com/hackstoic/kubernetes_practice/blob/master/存储.md))
[Persistent Volume](https://github.com/feiskyer/kubernetes-handbook/blob/master/concepts/persistent-volume.md)
[从Docker到Kubernetes进阶 ：StorageClass](https://www.qikqiak.com/k8s-book/docs/35.StorageClass.html )