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

​		有状态的容器应用通常需要可靠的存储来保存重要数据，以便pod重启后依然可以使用之前的数据。Kubernetes提供了PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 两种资源来实现对存储的管理。PV是对底层网络共享存储的抽象，而PVC是用户对于存储资源的一个“申请”，会”消费“PV资源。PV和PVC可以将pod与存储卷解耦，pod不需要去关心底层存储，直接使用PVC即可实现数据的持久化。

## 2. PV 介绍

​		pv － 持久化卷， 支持本地存储和网络存储，demo如下：

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

​		最新发行版本1.18仅支持对存储空间的设置。

##### 访问模式（Access Modes）

​        访问模式有以下三种，PV只能使用其中一种，多种访问方式不能同时生效。

- ReadWriteOnce –  可读可写，但只支持被单个 Pod 挂载
- ReadOnlyMany – 可以以只读的方式被多个 Pod 挂载
- ReadWriteMany – 这种存储可以以读写的方式被多个 Pod 共享

​		不是每一种存储类型都支持三种方式，常见的iSCSI与FC只支持前两种，而NFS与CephFS支持多节点读写，下表描述了不同存储提供者支持的访问模式。

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

​		通过设置storageClassName参数来指定存储的类别，只有请求该类别的PVC才可以进行绑定，未配置类别的PVC只能绑定参数storageClassName未设置的PV。

##### 四个阶段（Phase）

- Available – 资源可用，尚未被PVC声明绑定 
- Bound – 已被声明绑定
- Released – 释放状态，绑定的申明已被删除，但还未被重新声明
- Failed – 自动回收失败

##### 回收策略（Reclaim  Policy）

- Retain – 保留回收策略，允许手动回收资源，删除PVC时，状态由Bound变为Released
- Recycle – 基本的数据擦除(`rm -rf /thevolume/*`)，删除PVC时，状态由Bound变为Available
- Delete – 对于支持删除回收策略的卷插件，删除操作将从 Kubernetes 中删除 `PersistentVolume` 对象，并删除外部基础架构（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中的关联存储资产

## 3. PVC 介绍

​		PVC参数设置主要包括访问模式、卷模式、资源请求、PV选择条件、存储类别。

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

- accessModes：可以设置的三种访问方式与PV的设置相同
- volumeMode：声明使用与卷相同的约定，指示将卷作为文件系统或块设备使用
- resources：仅支持requests.storage
- selector：可以使用matchLabels和matchExpressions，只有标签与选择器匹配的卷可以绑定到声明，如果两者都设置了，则需要同时满足才能完成匹配
- storageClassName：指定的 StorageClass的名称来请求特定类别的PV，只有所请求的类与 PVC 具有相同 storageClassName 的 PV 才能绑定到 PVC。

> PVC 不一定要请求类。其 `storageClassName` 设置为 `""` 的 PVC 始终被解释为没有请求类的 PV，因此只能绑定到没有类的 PV（没有注解或 `""`）。没有 `storageClassName` 的 PVC 根据是否打开[`DefaultStorageClass` 准入控制插件](https://kubernetes.io/docs/admin/admission-controllers/#defaultstorageclass)，集群对其进行不同处理。
>
> - 如果打开了准入控制插件，管理员可以指定一个默认的 `StorageClass`。所有没有 `StorageClassName` 的 PVC 将被绑定到该默认的 PV。通过在 `StorageClass` 对象中将注解 `storageclass.kubernetes.io/is-default-class` 设置为 “true” 来指定默认的 `StorageClass`。如果管理员没有指定缺省值，那么集群会响应 PVC 创建，就好像关闭了准入控制插件一样。如果指定了多个默认值，则准入控制插件将禁止所有 PVC 创建。
> - 如果准入控制插件被关闭，则没有默认 `StorageClass` 的概念。所有没有 `storageClassName` 的 PVC 只能绑定到没有类的 PV。在这种情况下，没有 `storageClassName` 的 PVC 的处理方式与 `storageClassName` 设置为 `""` 的 PVC 的处理方式相同。

## 4. StorageClass（待更新）



## 参考文章

[Persistent Volume（持久化卷）](https://jimmysong.io/kubernetes-handbook/concepts/persistent-volume.html)

[存储介绍]([https://github.com/hackstoic/kubernetes_practice/blob/master/%E5%AD%98%E5%82%A8.md](https://github.com/hackstoic/kubernetes_practice/blob/master/存储.md))

[Persistent Volume](https://github.com/feiskyer/kubernetes-handbook/blob/master/concepts/persistent-volume.md)