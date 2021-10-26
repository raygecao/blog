---
weight: 20
title: "私有化交付下的应用打包方案"
subtitle: ""
date: 2021-08-26T20:58:05+08:00
lastmod: 2021-08-26T20:58:05+08:00
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
私有化交付的三要求：可复现性，易操作性和易维护性。
<!--more-->

## 私有化交付

公有云交付和私有化交付是ToB服务交付两种重要手段。公有云交付成本较低，数据安全性及性能的要求不需要太高，**通常允许连接外网**；而私有化交付一定程度上避免的资源的复用与共享，成本较高，但数据安全性高，适合数据敏感度高的企业，这些企业中大部分**不允许连接外网**。综上，私有化交付在数据安全很高的事业单位、银行及政府部门中占据主导地位。

私有化交付的一个重要的应用场景是**如何将一组服务部署到与外部隔绝的内网环境**，这也是我们今天研究的主题。本文不深入探究平台等基建是如何bootstrap出来，毕竟这个topic要溯源的话甚至要问一问服务器是哪来的。我们更专注于应用的打包方案。

显然，在网络受限的环境下，应用相关的所有资源都只能通过人肉搬运。下面的模式是私有化交付场景下常见的模式。

{{< mermaid >}}
graph TB;
home[公司内发布平台]
registry[公网发布中心]
offline[离线存储介质]
scenio[私有化交付现场]
home --打包并发布--> registry
registry --下载应用包-->offline
offline --人肉搬运并上传-->scenio
scenio--部署-->scenio
{{< /mermaid >}}

## 一个应用部署需要啥？

- **镜像**：容器化是私有化交付的关键，而容器化中最重要的组成就是image，image在服务部署中是不可或缺的。
- **编排文件与配置信息（应用包）**：私有化交付的特点是允许较高程度上的定制：不同私有化集群受节点数目、资源条件、特殊需求等因素，对服务的配置有着不同的需求。为了尽可能地自动化，将编排及默认配置打包在应用中是不错的选择，其中比较经典的例子是**helm chart**。编排及配置相当于应用管理工具包，有了它可以便于部署的自动化，但并不像image那么刚需。
- **数据包**：应用中的部分服务可能会依赖于某些数据包，如算法模型、调试工具包等。服务相关的附属物件的都可以认为是一种数据包。是否需要数据包是具体服务确定的。

打包应用，无外乎将应用所需的上述的所有组件pack起来带到现场进行部署。

## 当前交付方案

镜像/编排/数据包各自有着不同的发布流程，并且各自的发布中心独立，比如：

- 镜像的发布中心可以是docker registry。
- 编排包的发布中心可以是chart museum。
- 数据包的发布中心可以是某个s3。

为保证系统的可复现性，意味着现场环境也需要mirror这些infra，结构大致如下：

{{< image src="old-pack.png" caption="当前mirror方案" width=600 height=200 >}}

这个架构存在两个痛点：

- 发布中心分散，无法进行统一管理，增加维护成本。
- 公网中将各个组件打成一个tar包，隔离性太强，很多版本的image/数据包没有变化，存在同一组件打包多份的情况。这增加了打包时长，浪费公网存储资源。

## 优化方向

### 统一制品仓库

为解决第一个问题，我们考虑是否可以将这些发布中心归拢在一起。我们把上面提及的镜像，应用包及数据包统一描述为**制品（artifact）**，我们需要一个统一制品仓库来实现这些制品的发布。而OCI registry是个不错的选择，理由是：

- OCI registry即满足[OCI Distribution Spec](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)的registry，很多开源产品实现了此规范，如docker registry, harbor, nexus等。灵活性较强，可以根据发布规模、运维需求等因素灵活选型。
- OCI registry天然支持docker image的发布。上述制品中，只有image的发布最为复杂，[OCI Image Format Spec](https://github.com/opencontainers/image-spec/blob/main/spec.md)里定义的镜像格式是基于content-addressed的，而一般的存储系统都是location-addressed，使用registry统一制品会相对简单一些。
- 普通制品使用OCI registry存储已存在开源的解决方案[oras](https://github.com/oras-project/oras) (OCI registry as storage)，可以方便地实现content-addressable 特性。

### 公网发布中心改造

公网发布中心面临的最大问题就是存储空间的浪费。由于不同应用间以及同一应用各版本均为彼此隔离地打包上传，这导致了公网发布中心无法对内容做任何复用，经常一份镜像要重复打包成百上千次（比如ubuntu 这种base image），极大的浪费了存储空间。

一个显而易见的优化方向是将公网发布中心向着OCI registry方向改造，这样很多content可以被复用，这会带来一系列的好处：

- 节省了大量存储空间。
- 利用layer cache可以节省打包上传的时间。
- 可以增加layer粒度的diff机制，用户在公网下在新版本的应用时，只需要下载对应的patch即可，提升交付效率。

{{< admonition tip >}}

如果公网也是OCI registry，那么包的上传就有点类似于docker push，当上传的layer在registry已存在时，对应的blob将不会重复上传。只有新增的layer blob会被上传，形式上类似于增加了一层layer cache。

{{< /admonition >}}

### 增量发布

如果app bundle尺寸很大的话，会对交付效率带来较大的影响。

- 下载应用包耗时较长。
- 应用包占用的存储空间较大，并且应用包的跨网离线拷贝耗时较长。
- 现场环境加载应用包耗时较长。

{{< admonition info "交付效率" >}}

交付效率并没有一个官方的定义，我们这里姑且用**将一个应用从公司内网环境mirror到现场环境并成功部署所耗费的时间**来表述。增量发布只会影响mirror的效率，对部署效率不会有影响。

{{< /admonition >}}

初次交付时，这一问题无法避免，因为这时现场是个空的registry，需要mirror应用所需的所有artifacts。但是如果现场因新需求或bugfix等原因需要升级时，就不需要mirror另一个版本的全部content了。我们可以对这两个版本的bundle进行diff，生成一个补丁包来承载**新版本存在，但老版本不存在的blob集合**。如下图所示，app从v1.0.0升级到v1.0.1仅需要将`M2`与`L4`打到补丁包里即可，即其复用了就版本的`L1`与`L2`两个blob。

{{< mermaid >}}
graph TB;
m1["app@v1.0.0<br>[M1]"]
m2["app@v1.0.1<br>[M2]"]
l1((L1))
l2((L2))
l3((L3))
l4((L4))
m1 --> l1
m1 --> l2
m1 --> l3
m2 -.-> l1
m2 -.-> l2

subgraph patch-v1.0.1
m2 -.-> l4
end
{{< /mermaid >}}

增量发布对于小版本bugfix效率很高，改动越小，可以复用的layer越多，补丁包就越小，交付效率也就越高。

## 优化方案

### 现有开源项目参考

- [ORAS](https://github.com/oras-project/oras-go)：使用OCI registry存储artifacts。
  - 可以将若干文件以layer的方式存到某个repository下。
  - push操作只能实现一层关联，即manifest => layers+config。
  - 依赖特定annotation，无法与docker image兼容，即无法pull出docker image。
- [crane](https://github.com/google/go-containerregistry): 与registry交互的go lib。
  - 基本封装了OCI Distribution Spec。
  - 与image format绑定较深，对OCI Artifacts支持不完善，尤其是对index的支持。
- [cnab-to-oci](https://github.com/cnabio/cnab-to-oci)：使用OCI registry来发布应用包（application bundle），其引入了打包的概念，将若干Image打包成一个bundle，并在registry中流转。
  - 突出了app bundle的概念，便于应用整体的管理。
  - 与Docker Image Format Spec强绑定，底层调用moby sdk，无法支持通用的artifacts。

- [helm registry](https://github.com/helm/helm/tree/main/internal/experimental/registry)：chart 支持基于OCI registry的发布。
  - local cache以OCI [image-layout](https://github.com/opencontainers/image-spec/blob/main/image-layout.md)结构组织，content addressable & location addressable。
  - 与registry的交互必须依赖于local cache。
  - 同样仅支持两层镜像结构
- [containered](https://github.com/containerd/containerd) 一个容器运行时的标准，其中`images`pkg包含对OCI image format的处理，`remotes`pkg包含与registry交互的底层sdk。
  - 兼容全部的OCI Spec。
  - 封装度低，对image的处理十分通用，不局限于docker image。

### 组织结构

打包操作无非是将若干的artifacts包在一起，并给其一个用于定位的reference（包名），而其对偶的拆包操作就是根据包名能获取到所有artifacts。

[cnab](https://cnab.io/)给我们提供一个很好的组织方式的参考，bundle reference作为检索的入口，可以关联到所有的artifacts，形式上就类似于一个[image index](https://github.com/opencontainers/image-spec/blob/main/image-index.md)与[image manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md)之间的关系。

{{< image src="bundle-struct.png" caption="cnab's bundle format in OCI registry" width=500 height=200 >}}

简化一下组织结构，本质上就是一颗多叉树。

{{< mermaid >}}
graph TB;
subgraph OCI image index
bundle
pi[multi-platform index]
end
subgraph OCI image manifest
image
ap[应用包]
dp[数据包]
end
subgraph OCI layer
il[rootfs.diff]
chart
data
end

bundle --> pi
bundle --> ap
bundle --> dp

pi --> image

image --> il
ap --> chart
dp --> data
{{< /mermaid >}}

## 实践探索

我们以一个具体的例子来简化一下我们要解决的问题：我**们如何使用helm在私有化k8s集群中部署起来一个nginx服务**。我们要做的是将helm chart与image发布并打包，在现场环境中导入这些artifacts。

结合oras, crane与containerd sdk开发一个小工具**cb**，提供如下核心功能

| cmd      | 作用                               | 实现要点                                                     |
| -------- | ---------------------------------- | ------------------------------------------------------------ |
| `push`   | 上传普通制品（非docker image）     | 对`oras.Push`做简单封装                                      |
| `pull`   | 下载普通制品，是push的对偶操作     | 对`oras.Pull`做简单封装                                      |
| `bundle` | 打包生成bundle，将一组制品关联起来 | 生成index索引artifact list，并将index及artifacts push到registry中特定的repository里 |
| `pack`   | 将registry中的bundle保存到本地     | 以OCI image-layout的形式存储上述结构，相关的reference通过特殊的annotation记录在`index.json`中 |
| `load`   | 从OCI image-layout中恢复bundle     | 加载OCI image-layout中的`index.json`并将关联的content及reference  push到registry |
| `diff`  | 比较并产生target reference相对source reference的patch包 | 形式仍以OCI image-layout存在，只是`blobs`与`index.json`中存在的是两个ref之间的diff，而非全量的引用关系。一般来讲，diff产生的patch包，脱离了source reference是无法加载的 |
| `patch` | 加载patch包 | Provider需要结合source reference与patch包构建完整target reference。另外patch包里记录source与target是一个不错的选择 |


此外**cb**还wrap了**crane**的`manifest`，`catalog`及`list` 功能，另外结合了[go-graphviz](https://github.com/goccy/go-graphviz)实现了OCI layer可视化功能。

### 准备阶段

- 在本地起一个docker registry（确保registry是干净的），并添加dns **myregistry**。

- 在docker hub上下载`ubuntu:21.04`和`nginx:1.21.1`，以及使用helm create创建默认的nginx chart。

### 上传制品

```shell
# push chart
$ cb push myregistry:5000/nginx-chart:1.0.0 --files mychart

# push nginx image
$ docker tag nginx:1.21.1 myregistry:5000/nginx:1.21.1
$ docker push myregistry:5000/nginx:1.21.1

$ cb catalog myregistry:5000 # 列出registry中的repositories
NO.	NAME
0  	nginx
1  	nginx-chart
```

Push仅仅包装了oras cli，nginx chart的manifest如下

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.unknown.config.v1+json",
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
    "size": 2
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:64f02409c5583265a67390256055c95902696345bdfb46a566b14ea9eac7c306",
      "size": 3943,
      "annotations": {
        "io.deis.oras.content.digest": "sha256:f2c5bf294ce3a1fcd249c00df5f916135059d107cc3f35c92c86189b7113d74e",
        "io.deis.oras.content.unpack": "true",
        "org.opencontainers.image.title": "mychart"
      }
    }
  ]
}
```

### 打包bundle

定义nginx-app bundle，tag为v1.0.0，包含nginx镜像与nginx chart。bundle.yaml简化如下

```yaml
name: myregistry:5000/nginx-app
tag: v1.0.0
artifacts:
  - name: "myregistry:5000/nginx:1.21.1"
  - name: "myregistry:5000/nginx-chart:1.0.0"
```

```shell
$ cb bundle nginx-bundle.yaml # 打包nginx-app
```

`nginx-app:v1.0.0`的index如下，reference记录在`org.opencontainers.image.ref.name` annotation里，这个reference（tag）需要被记录并在现场加载时恢复。

```json
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:053598290cc6fad47d9af8f98baa939d6ebb92f672f7a1871e29cb55a85f2964",
      "size": 257
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:5e95e5eb8be4322e3b3652d737371705e56809ed8b307ad68ec59ddebaaf60e4",
      "size": 1570,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx:1.21.1"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:7ca59b0f5387479b242896fa0c507d8c29dfa5292100a5576f8663a546e41611",
      "size": 602,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx-chart:1.0.0"
      }
    }
  ]
}
```

### 下载bundle

```shell
# 将nginx-app bundle下载到nginx-app目录下
$ cb pack myregistry:5000/nginx-app:v1.0.0 -o nginx-app 

# 验证nginx-app中的结构符合OCI Image Layer Spec
$ tree nginx-app
nginx-app
├── blobs
│   └── sha256
│       ├── 053598290cc6fad47d9af8f98baa939d6ebb92f672f7a1871e29cb55a85f2964
│       ├── 12455f71a9b5e0c207a601fb32bcf7f10a933d7193574d968409bbc5c2d89fe0
│       ├── 2a53fa598ee20ad436f2f9da7c0a21cce583bd236f47828895d771fb2e8795e1
│       ├── 44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a
│       ├── 572061c855037851b6384e6bba08cb7d48f71e74631865641518644ba1469e32
│       ├── 5e95e5eb8be4322e3b3652d737371705e56809ed8b307ad68ec59ddebaaf60e4
│       ├── 64f02409c5583265a67390256055c95902696345bdfb46a566b14ea9eac7c306
│       ├── 7ca59b0f5387479b242896fa0c507d8c29dfa5292100a5576f8663a546e41611
│       ├── 9e324aa228dbd3c1b80ea7c20b6d63b897605fa92f168ccce976a2df42375e77
│       ├── b86f2ba62d17b165964516228297d3ba669d60b6a283b5fd7779b27d7ec33871
│       ├── dd34e67e3371dc2d1328790c3157ee42dfcae74afffd86b297459ed87a98c0fb
│       ├── e1acddbe380c63f0de4b77d3f287b7c81cd9d89563a230692378126b46ea6546
│       ├── e21006f71c6fb784a76159590b6ba8ab3fb22e5026f67abcf5feb8e4231837d6
│       └── f3341cc17e586daa9660abf087f13b2eba247bcf6646ee972e85d4cbaf18dbae
├── index.json
├── ingest
└── oci-layout

# 查看index内容，相当于把nginx-app在全局index中拍平
$ cat nginx-app/index.json| jq .
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.index.v1+json",
      "digest": "sha256:572061c855037851b6384e6bba08cb7d48f71e74631865641518644ba1469e32",
      "size": 674,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx-app:v1.0.0"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:5e95e5eb8be4322e3b3652d737371705e56809ed8b307ad68ec59ddebaaf60e4",
      "size": 1570,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx:1.21.1"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:7ca59b0f5387479b242896fa0c507d8c29dfa5292100a5576f8663a546e41611",
      "size": 602,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx-chart:1.0.0"
      }
    }
  ]
}
```

### 加载bundle

删除掉registry中的内容并重启，将nginx-app加载到registry中，可以恢复完整的nginx-app bundle。

```shell
$ cb load nginx-app
```

### 多层结构

当前打包策略实现了OCI Artifacts的高扩展性，结构树可以灵活地向上延展，比如bundle里嵌套bundle。

```yaml
name: myregistry:5000/ubuntu-nginx-app
tag: v1.0.0
artifacts:
  - name: "myregistry:5000/nginx-app:v1.0.0" # 这是一个bundle
  - name: "myregistry:5000/ubuntu:21.04"  # 这是一个image
```

打包后，`ubuntu-nginx-app:v1.0.0`的index如下：

```json
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:52e0a43cc3025f0814510022bc2bc7f64135a9f4c93944ea0b82df344cc798c0",
      "size": 257
    },
    {
      "mediaType": "application/vnd.oci.image.index.v1+json",
      "digest": "sha256:572061c855037851b6384e6bba08cb7d48f71e74631865641518644ba1469e32",
      "size": 674,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/nginx-app:v1.0.0"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0",
      "size": 529,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/ubuntu:21.04"
      }
    }
  ]
}
```

对应的层序结构为：
{{< image src="bundle-in-bundle.svg" caption="ubuntu-nginx-app:v1.0.0镜像的层序结构" width=600 height=400 >}}

下载与加载均可以正常工作。

### 补丁机制

通过`ubuntu-nginx-app:v1.0.0`与`nginx-app:v1.0.0`两个bundle验证一下补丁机制的可用性。从上述层序结构图中可以看到后者是前者的一个子集，所以预想中patch包应包含**根节点及其左右两个子分支全部节点对应的blob，及ubuntu-nginx-app:v1.0.0和ubuntu:21.04两个reference**。

```shell
$ cb diff myregistry:5000/ubuntu-nginx-app:v1.0.0 myregistry:5000/nginx-app:v1.0.0 -o patch
INFO[0000] get 6 diffs
```

```shell
$ tree patch
patch
├── blobs
│   └── sha256
│       ├── 4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1
│       ├── 52e0a43cc3025f0814510022bc2bc7f64135a9f4c93944ea0b82df344cc798c0
│       ├── a2723fc64a92418869862e0a14d8e913641ba6e4bca78cf43d0db4be4c3c14fa
│       ├── bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481
│       ├── ee951c00fa8985cc5f5b49f0d7fe5e456697198f24fed85142b4869083b7085b
│       └── ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0
├── index.json
├── ingest
└── oci-layout

3 directories, 8 files
```

```shell
$ cat patch/index.json | jq .
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.index.v1+json",
      "digest": "sha256:a2723fc64a92418869862e0a14d8e913641ba6e4bca78cf43d0db4be4c3c14fa",
      "size": 669,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/ubuntu-nginx-app:v1.0.0"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "digest": "sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0",
      "size": 529,
      "annotations": {
        "org.opencontainers.image.ref.name": "myregistry:5000/ubuntu:21.04"
      }
    }
  ]
}
```

上述结果表明patch包中`blobs`及`index.json`中的引用均符合我们预期。

清空registry并将`nginx-app:v1.0.0`load到registry中以验证应用patch的正确性。

```shell
$ cb load nginx-app
INFO[0000] successfully push ref myregistry:5000/nginx-app:v1.0.0
INFO[0001] successfully push ref myregistry:5000/nginx-chart:1.0.0
$ cb patch patch
INFO[0000] successfully push ref myregistry:5000/ubuntu:21.04
INFO[0001] successfully push ref myregistry:5000/ubuntu-nginx-app:v1.0.0
```

## 总结

本文主要介绍了私有化交付下的应用打包方案，简单介绍了私有化交付的模式，提出了现有的打包方案及基于layer cache的一些优化方向。

本文要解决的问题是如何将一组内容从一个registry离线搬运到另一个registry中。问题的核心是如何将一组artifacts关联起来，这些内容应以什么组织形式在公司与现场之间流转。

通过bundle的概念对应用进行打包，便于对应用及版本进行管理。而OCI image layout是将bundle本地化的一个不错的解决思路，此架构可以与[helm OCI](https://helm.sh/docs/topics/registries/)完全兼容。
