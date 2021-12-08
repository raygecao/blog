---
weight: 20
title: "浅谈Docker Storage Driver"
subtitle: ""
date: 2021-10-16T12:23:06+08:00
lastmod: 2021-10-16T12:23:06+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["container"]
categories: ["探索与实战"]


toc:
  auto: false
lightgallery: true
license: ""
---
Docker 通过storage driver来**存储镜像层，并且将数据存储到容器层**。本文主要介绍storage driver与volume的区别，以及对经典的driver：`aufs`, `overlay`, `overlay2`进行简单的探索。
<!--more-->

## Storage Driver vs Docker Volume

- **Storage Driver**
  - 用于存储image layer。
  - 用于在容器层（writeable layer）存数据。
  - 容器层在容器销毁后丢失，无法做到持久化，无法在多个容器中共享用户数据。
  - 一般使用**CoW**（copy on write）机制写容器层，初次写数据时需要从镜像层将数据copy-up到容器层进行写操作。
  - 容器可以复用镜像层，由于CoW机制，每创建一个容器仅多创建一个很薄的容器层，可以充分提升空间效率。
- **Docker Volume**
  - 持久化容器产生的数据，与容器生命周期无关。
  - 数据可以在容器间共享。
  - 写volume性能远比写容器层的性能好，volume适合于写密集的场景。

## Aufs

### 理论

- 判断内核是否支持aufs driver，结果输出如下表示支持：

  ```shell
  $ grep aufs /proc/filesystems
  
  nodev aufs
  ```

- Aufs是一个联合文件系统，其采用**union mount**将linux系统上的多个目录**堆叠**成一个目录，一个目录代表一个branch（在docker术语中对应为layer）。

  {{< image src="aufs-layers.png" caption="aufs layer组织形式" width=800 height=400 >}}

- **存储结构**

  - `diff/`：每一层的内容，每层以一个独立的子目录存储。
  - `layers/`：存放layers的元信息以标识镜像层如何堆叠，每层以一个文件表示。
  - `mnt/`：挂载点，每层一个，用于向容器组装/挂载联合文件系统。

- **容器读文件**
  - 仅在容器层存在：直接从容器层读出。
  - 仅在镜像层存在：沿着layer stack寻找文件，找到后读出。
  - 在容器层和镜像层均存在：从容器层读出，镜像层相应的文件被容器层所遮盖。
- **容器写文件**
  - 从容器层查找文件，如果存在则直接修改。
  - 如果不存在，沿着镜像layer stack查找文件，如果文件存在则copy-up到容器层进行修改。
  - 如果镜像层也不存在，则直接在容器层创建文件。
- **容器删除文件/目录**
  - 删除文件：在容器层创建一个`whiteout file`，避免向下层继续寻找。
  - 删除目录：在容器层创建一个`opaque file`（实测依然是whiteout file）。
- **Aufs性能**
  - 对容器密集型场景友好，因其能有效利用运行中的容器image，使得容器启动更迅速，减少磁盘空间使用。
  - 能有效使用page cache。
  - 定位文件开销大，需要沿着layer stack逐层定位。  
  - 首次写操作开销大，尤其是文件在镜像层存在时，需要copy-up至容器层。由于**aufs底层存储是文件级别而非块级别**，对于文件的修改需要将整个文件复制到容器层。因此文件越大，写性能越差。
- **最佳实践**
  - 使用ssd盘，速度远高于旋转式磁盘。
  - 对于write-heavy负载使用**volume**，减少IO开销的同时可以将容器数据持久化，并且可以在多个容器中共享。

### 实践

使用ubuntu:16.04 image对aufs driver进行探究，首先启动一个container：
```shell
$ docker run -it --rm ubuntu:16.04 bash
```

Docker存储目录结构如下：

```shell
$ tree -L 2 /var/lib/docker/aufs

/var/lib/docker/aufs
├── diff
│   ├── 1492027998d17f2f422a0d46ed6e41a9ef59911bb13357765aeb0dc9f150ea76
│   ├── 72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a
│   ├── 8065b4da5339d297dfb53f4bd7edfeba97b5c0089eda635c412943c4d7e55e81
│   ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227
│   ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227-init
│   └── aff7006186a6867de9cc7a75f1d31c90eb52753b9d96d6c85a3f66e78ccd465b
├── layers
│   ├── 1492027998d17f2f422a0d46ed6e41a9ef59911bb13357765aeb0dc9f150ea76
│   ├── 72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a
│   ├── 8065b4da5339d297dfb53f4bd7edfeba97b5c0089eda635c412943c4d7e55e81
│   ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227
│   ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227-init
│   └── aff7006186a6867de9cc7a75f1d31c90eb52753b9d96d6c85a3f66e78ccd465b
└── mnt
    ├── 1492027998d17f2f422a0d46ed6e41a9ef59911bb13357765aeb0dc9f150ea76
    ├── 72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a
    ├── 8065b4da5339d297dfb53f4bd7edfeba97b5c0089eda635c412943c4d7e55e81
    ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227
    ├── a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227-init
    └── aff7006186a6867de9cc7a75f1d31c90eb52753b9d96d6c85a3f66e78ccd465b

15 directories, 6 files
```

可以看到存在一个`-init`后缀目录项，因此可以断定`a5cad43`是容器层，其他均为镜像层。

从挂载信息中可以验证这一点，aufs仅需挂载upperdir：

```shell
$ mount | grep aufs

none on /var/lib/docker/aufs/mnt/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227 type aufs (rw,relatime,si=b3be613c19d0e57c,dio,dirperm1)
```

可以看到此挂载source内容即为ubuntu rootfs的内容：

```shell
$ ls /var/lib/docker/aufs/mnt/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227

bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

此镜像中layer stack又是如何组织的呢？

```shell
$ cat /var/lib/docker/aufs/layers/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227

a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227-init
aff7006186a6867de9cc7a75f1d31c90eb52753b9d96d6c85a3f66e78ccd465b
8065b4da5339d297dfb53f4bd7edfeba97b5c0089eda635c412943c4d7e55e81
1492027998d17f2f422a0d46ed6e41a9ef59911bb13357765aeb0dc9f150ea76
72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a
```

可以从容器层的layers内容中看到layer stack自顶向下的组织形式为 **a5 -> a5-init -> af -> 80 -> 14 -> 72**。

接下来验证一下容器内更改文件系统对aufs存储有什么影响。

#### 创建文件

```shell
# 容器内
$ touch kkk

# 宿主机
$ find /var/lib/docker/aufs -name kkk

/var/lib/docker/aufs/mnt/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227/kkk
/var/lib/docker/aufs/diff/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227/kkk
```

可以看见文件的创建仅发生在容器层。同理，更改文件内容也是如此，只是涉及到从镜像层到容器层的copy-up。

#### 删除文件

```shell
# 容器内
$ rm /bin/zegrep

# 宿主机
$ find  /var/lib/docker/aufs -name *zegrep*

/var/lib/docker/aufs/mnt/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227/usr/share/man/man1/zegrep.1.gz
/var/lib/docker/aufs/diff/a5cad43027e71dcd8efaa9287a4c502a59631c108d22b5e861d9ca1878525227/bin/.wh.zegrep
/var/lib/docker/aufs/diff/72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a/bin/zegrep
/var/lib/docker/aufs/diff/72c62493358ebb6ae7d47717a491b7f3ff402fb338a34d9df8529c166223737a/usr/share/man/man1/zegrep.1.gz
```

可以看到容器中`/bin/zegrep`位于最底层的`72c6249`镜像层，容器内删除此文件并不会在镜像层删除，而是在容器层`a5cad43`产生一个whiteout文件——`/bin/.wh.zegrep`，通过whiteout文件避免沿着layer stack向下查找。如果容器内误删了某个文件，可以在对应容器层diff目录中将whiteout文件删除以将其恢复。

## OverlayFS

- OverlayFS是一种类似aufs的联合文件系统，但是速度更快，更简单。Docker基于overlayFS提供了两种存储驱动：`overlay`和`overlay2`

- 将两个目录以**union mount**的形式组织成单个目录，镜像层目录为`lowerdir`，容器层目录为`upperdir`，对外呈现一致的目录视图（容器内看到的文件系统）叫做`merged`。
  
  {{< image src="overlay-layers.png" caption="overlayFS layer组织形式" width=800 height=400 >}}

### Overlay2

- Overlay2 driver每一层均以一个`patch`的形式表示，基于内核overlayFS的multiple lower layers特性实现，不再需要硬链接，**lowerdir是将镜像层所有的layer overlay起来组成**。

  容器层挂载信息：

  ```shell
  # mount | grep overlay
  overlay on /var/lib/docker/overlay2/23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/XRV25KXVQWJ42GBK5UJD7ICFVV:/var/lib/docker/overlay2/l/ZVAQQEPIP4VXNQK2LZM24JXP7H:/var/lib/docker/overlay2/l/7YTDWL6UA7JEAI5KPCZEUDIUUF:/var/lib/docker/overlay2/l/OER42RBQQTIFRQ7LZRR4IPRQWU:/var/lib/docker/overlay2/l/2MR2ULDESC4VPFHAYTSOAVXLFF,upperdir=/var/lib/docker/overlay2/23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b/diff,workdir=/var/lib/docker/overlay2/23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b/work,xino=off)
  ```

  层间组织关系简单示意：
  
  {{< image src="overlay2-driver.png" caption="overlay2 driver层堆叠示意" width=800 height=400 >}}


- **存储结构**

  - `diff/`：本层的内容。
  - `link`：当前层的短标识。
  - `lower`：当前层的父layers，按层序排列，除最底层外有此文件。
  - `work/`：overlayFS内部使用的文件，除最底层外有此目录。
  - `merged/`：其自身及其父layer的联合目录结构，只有容器层有此目录。
  - `l/`：此目录存放短id的符号链接。

  ```shell
  # tree -L 2 /var/lib/docker/overlay2
  /var/lib/docker/overlay2
  ├── 013299735d8abd2c0cc5cbad32f011be44fad600a624bc6c2f1f94b9bffa64c2
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   └── work
  ├── 23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   ├── merged
  │   └── work
  ├── 23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b-init
  │   ├── committed
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   └── work
  ├── 36bf33266ffa01007cdc8db601c77fb6bd5aa5d2dc8dd35fe873e1580073db82
  │   ├── committed
  │   ├── diff
  │   └── link
  ├── 3862629a36e3286f03a2d9234633fdb0e9301a33a7195087afd3b3f92c1464fb
  │   ├── committed
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   └── work
  ├── 6d1ca51490fa03a0ebf3767e0ddc93508593fa380b542af3e5187f0ec5e2629c
  │   ├── committed
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   └── work
  ├── 74e31d6f05b337350eb841d01d9728852f6fb0c69a007957dcf04cdb570d8b06
  │   ├── committed
  │   ├── diff
  │   ├── link
  │   ├── lower
  │   └── work
  └── l
      ├── 2MR2ULDESC4VPFHAYTSOAVXLFF -> ../36bf33266ffa01007cdc8db601c77fb6bd5aa5d2dc8dd35fe873e1580073db82/diff
      ├── 7YTDWL6UA7JEAI5KPCZEUDIUUF -> ../74e31d6f05b337350eb841d01d9728852f6fb0c69a007957dcf04cdb570d8b06/diff
      ├── EYKZCXLZJEZJW55DB7S5ID2HKW -> ../23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b/diff
      ├── OER42RBQQTIFRQ7LZRR4IPRQWU -> ../3862629a36e3286f03a2d9234633fdb0e9301a33a7195087afd3b3f92c1464fb/diff
      ├── TQQ75355OL5U7VFEC6XG7YBWE5 -> ../013299735d8abd2c0cc5cbad32f011be44fad600a624bc6c2f1f94b9bffa64c2/diff
      ├── XRV25KXVQWJ42GBK5UJD7ICFVV -> ../23ae9a63ab9f4f4c36af29f78fb64d4e943c7af9f241b696e7870d37dca0130b-init/diff
      └── ZVAQQEPIP4VXNQK2LZM24JXP7H -> ../6d1ca51490fa03a0ebf3767e0ddc93508593fa380b542af3e5187f0ec5e2629c/diff
  ```

  {{< admonition note >}}
  
  OverlayFS中删除文件仍然使用**whiteout文件**来阻止向容器层以下查找文件，只是在aufs中，whiteout文件以`.wh.{FILENAME}`命名的**普通文件**呈现；而在overlayFS中，whiteout文件以原文件命名的**字符设备文件**呈现。
  
  例如，从附录ubuntu:16.04的镜像分析中可以看到第三层删除了`/var/lib/apt/lists`下的所有文件，对应layer的diff中可以看到，这些文件都变成了字符设备文件：
  
  ```shell
  $ ll -a /var/lib/docker/overlay2/74e31d6f05b337350eb841d01d9728852f6fb0c69a007957dcf04cdb570d8b06/diff/var/lib/apt/lists/
  
  total 8
  drwxr-xr-x 2 root root 4096 Aug 31 09:21 ./
  drwxr-xr-x 3 root root 4096 Aug  5 03:01 ../
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial_InRelease
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial_main_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial_main_i18n_Translation-en
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial_restricted_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial_restricted_i18n_Translation-en
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial-updates_InReleasec--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial-updates_main_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial-updates_main_i18n_Translation-en
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial-updates_restricted_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 archive.ubuntu.com_ubuntu_dists_xenial-updates_restricted_i18n_Translation-en
  c--------- 1 root root 0, 0 Oct  7 13:55 lock
  c--------- 1 root root 0, 0 Oct  7 13:55 partial
  c--------- 1 root root 0, 0 Oct  7 13:55 security.ubuntu.com_ubuntu_dists_xenial-security_InRelease
  c--------- 1 root root 0, 0 Oct  7 13:55 security.ubuntu.com_ubuntu_dists_xenial-security_main_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 security.ubuntu.com_ubuntu_dists_xenial-security_main_i18n_Translation-en
  c--------- 1 root root 0, 0 Oct  7 13:55 security.ubuntu.com_ubuntu_dists_xenial-security_restricted_binary-amd64_Packages
  c--------- 1 root root 0, 0 Oct  7 13:55 security.ubuntu.com_ubuntu_dists_xenial-security_restricted_i18n_Translation-en
  ```
  
  {{< /admonition >}}

### Overlay

- Overlay driver每一层都构筑成完整的镜像，即每一层都是从最底层到当前层overlay出的完整结构，下一层的文件以**硬链接**的方式出现在它的上一层，**lowerdir只由镜像层的top layer组成**。

  容器层挂载信息：

  ```shell
  # 镜像层每层以硬链接形式共享文件
  $ cd /var/lib/docker/overlay; ls -i 07236efba039eb5cb4e0b6ec010218aefd17293f8297d43c04be4a5b0fd59ab7/root/bin/ls  09195a984aeac5c05ac3c487b1bb2ffb4d57465a3dc575c13e8d4610484ae0b2/root/bin/ls
  
  655184 07236efba039eb5cb4e0b6ec010218aefd17293f8297d43c04be4a5b0fd59ab7/root/bin/ls
  655184 09195a984aeac5c05ac3c487b1bb2ffb4d57465a3dc575c13e8d4610484ae0b2/root/bin/ls
  
  # 容器层仅overlay mount了镜像层最顶层的root目录
  $ mount | grep overlay
  
  overlay on /var/lib/docker/overlay/4d0852eca897d746e415bffb325ad3661f8371987082f6b6c8e9d5b9fda9abc5/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay/07236efba039eb5cb4e0b6ec010218aefd17293f8297d43c04be4a5b0fd59ab7/root,upperdir=/var/lib/docker/overlay/4d0852eca897d746e415bffb325ad3661f8371987082f6b6c8e9d5b9fda9abc5/upper,workdir=/var/lib/docker/overlay/4d0852eca897d746e415bffb325ad3661f8371987082f6b6c8e9d5b9fda9abc5/work,xino=off)
  ```

  层间组织关系简单示意：

  {{< image src="overlay-driver.png" caption="overlay driver层堆叠示意" width=800 height=400 >}}

- **存储结构**

  - `root/`：本层完整的目录结构，下层文件以硬链的形式出现在本层，所有镜像层有且仅有此目录。
  - `merged/`： 其自身及其父layer的联合目录结构，只有容器层有此目录。
  - `upper/`：本层的内容，类似于overlay2中的**diff**目录，但只有容器层有此目录。
  - `work/`：overlayFS内部使用的文件，只有容器层有此目录。
  - `lower-id`：lowerdir（镜像层顶层）的id。

  ```shell
  $ tree -L 2 /var/lib/docker/overlay
  
  /var/lib/docker/overlay
  ├── 07236efba039eb5cb4e0b6ec010218aefd17293f8297d43c04be4a5b0fd59ab7
  │   └── root
  ├── 09195a984aeac5c05ac3c487b1bb2ffb4d57465a3dc575c13e8d4610484ae0b2
  │   └── root
  ├── 15afdc0d3161ffd9002a5d7714bb3566a658bfa286de82ff95a49368769e5072
  │   └── root
  ├── 4d0852eca897d746e415bffb325ad3661f8371987082f6b6c8e9d5b9fda9abc5
  │   ├── lower-id
  │   ├── merged
  │   ├── upper
  │   └── work
  ├── 4d0852eca897d746e415bffb325ad3661f8371987082f6b6c8e9d5b9fda9abc5-init
  │   ├── lower-id
  │   ├── upper
  │   └── work
  └── f6bdc701b8624ef39632441ad84efde79de50ae2bc3c637eabf0827084d547ec
      └── root
  ```

  {{< admonition note "init layer">}}
  
  在上述driver中，存储目录中都有一个`-init`后缀的目录，此为**init layer**，位于容器层与镜像层之间，**只读**。其主要包含了docker为容器准备的一些文件：
  
  ```shell
  $ tree /var/lib/docker/overlay/2178b65434aae442f3e71937bc6f775063a9993a3684625e982c7195052e9289-init/
  
  /var/lib/docker/overlay/2178b65434aae442f3e71937bc6f775063a9993a3684625e982c7195052e9289-init/
  ├── lower-id
  ├── upper
  │   ├── dev
  │   │   └── console
  │   └── etc
  │       ├── hostname
  │       ├── hosts
  │       ├── mtab -> /proc/mounts
  │       └── resolv.conf
  └── work
      └── work
  ```
  
  upper dir中除了`mtab`是指向`/proc/mounts`的软链接之外，其他都是空的普通文件。这些文件都是Linux runtime必须的文件，如果缺少会导致某些程序或库出现异常。**init layer主要是用于占坑，避免系统因缺少特殊文件而崩溃，具体内容后续进行bind mount**。
  
  由于init layer很薄并且只读，上述讨论将其予以忽略，其本身可以帮助我们快速定位到容器层的id。
  
  {{< /admonition >}}

## Aufs, overlay与overlay2比较

- **相同点**
  - 划分容器层与镜像层，镜像层只读可以复用；容器层可写，采用CoW机制。
  - 底层基于**file-level**，而非block-level，可有效利用内存，但写操作开销大，CoW效率低。当文件很大时，对其修改需要全部copy-up到容器层，即便是仅仅进行了很小的修改。
- **不同点**
  - Aufs driver是按照layer stack组织镜像层的，即在定位文件时需要沿着layer stack一层一层地去定位。因此其性能相较于overlayFS差，尤其是镜像层数比较深时。
  - Overlay driver每层都构筑完整的镜像目录结构，通过硬链接的形式复用底层镜像层的文件。在镜像层数较深时，定位文件及写容器层的性能要略好于overlay2。但是由于每层都是完整的镜像目录结构，**各级子目录会占用大量的inode**，尤其当层数很深时，inode易被耗尽。
  - Overlay2 driver每层仅包含当前层的增量内容，通过overlay multiple lower layers形式构筑lowerdir，解决了overlay driver中消耗大量inode的问题，也是docker官方推荐的storage driver。

## 附录

使用dive对ubuntu:16.04进行镜像分析：
```shell
$ dive ubuntu:16.04
```

{{< image src="ubuntu-image.png" caption="ubuntu:16.04的镜像分析" width=600 height=200 >}}

## 参考文献
- [About storage drivers](https://docs.docker.com/storage/storagedriver/)
- [Use the AUFS storage driver](https://docs.docker.com/storage/storagedriver/aufs-driver/)
- [Use the OverlayFS storage driver](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)
- [DOCKER基础技术：AUFS](https://coolshell.cn/articles/17061.html)
