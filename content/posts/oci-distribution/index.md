---
weight: 20
title: "OCI Distribution Spec探索与实践"
subtitle: ""
date: 2021-08-26T20:57:33+08:00
lastmod: 2021-08-26T20:57:33+08:00
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

OCI Distribution Spec 定义了内容分发的一组标准API。
<!--more-->

## 术语介绍

- **Registry**：存储/发布artifact的中心，提供OCI Distribution Spec规范的api。
- **Blob**：存储在registry中二进制形式的内容，通过digest寻址到。
- **Manifest**：定义一个artifact的json文档，用于定位artifact的具体内容。
- **Digest**: 对blob内容加密后的唯一标识符。
- **Artifact**：以blob形式存储的概念化的内容，一般是由一个manifest定址若干blobs构成。
- **Tag**：用于便于人类阅读、定位manifest的引用。



## Registry基本要求

- `Pull`，从registry中拉取conent。
- `Push`，向registry中发布content。
- `Content Discovery`，从registry中获取content列表项。
- `Content Managerment`，控制registry中content的完整生命周期。

## 标准API汇总

| Method   | API Path                                                     | Status code       | 用途                                                         |
| -------- | ------------------------------------------------------------ | ----------------- | ------------------------------------------------------------ |
| `GET`    | `/v2/`                                                       | `200/400/401`     | 用于判断registry是否实现OCI Distribution Spec                |
| `HEAD`   | `/v2/<name>/blobs/<digest>`                                  | `200/404`         | 用于判断指定的blob是否存在                                   |
| `GET`    | `/v2/<name>/blobs/<digest>`                                  | `200/404`         | 用于获取指定的blob                                           |
| `HEAD`   | `/v2/<name>/manifests/<reference>`                           | `200/404`         | 用于判断指定的manifest是否存在                               |
| `GET`    | `/v2/<name>/manifests/<reference>`                           | `200/404`         | 用于获取指定的blob                                           |
| `POST`   | `/v2/<name>/blobs/uploads/`                                  | `202/404`         | 获取上传blob的sessionID，为后续**PUT/PATCH**操作提供locator      |
| `POST`   | `/v2/<name>/blobs/uploads/?digest=<digest>`                  | `201/202/404/400` | 直接通过POST上传blob，optional                        |
| `PATCH`  | `/v2/<name>/blobs/uploads/<reference>`                       | `202/404/416`     | 分片上传blob chunks                                          |
| `PUT`    | `/v2/<name>/blobs/uploads/<reference>?digest=<digest>`       | `201/404/400`     | 上传blob。reference为之前POST请求获取的id                    |
| `PUT`    | `/v2/manifests/`                                             | `201/404`         | 上传一个manifest                                             |
| `GET`    | `/v2/<name>/tags/list?n=<integer>&last=<integer>`            | `200/404`         | 获取某个repository下的所有tag，可以通过**list，last** query进行分页 |
| `DELETE` | `/v2/<name>/manifests/<reference>`                           | `202/404/400/405` | 删除某个manifest                                             |
| `DELETE` | `/v2/<name>/blobs/<digest>`                                  | `202/404/405`     | 删除某个blob                                                 |
| `POST`   | `/v2/<name>/blobs/uploads/?mount=<digest>&from=<other_name>` | `201/202/404`     | 如果某个blob在其他repository上存在，此API可以将blob挂载到同一registry下的不同repository |

{{< admonition note "reference与digest区别" >}}

digest指的是content的唯一标识，使用digest可以标识anything，因此**digest可以用作reference**。但使用digest的问题是不容易记忆及索引，从而引入tag来便于记忆，因此**tag是通常意义上的reference**。需要注意的是**一般只有manifest可以使用reference寻址，而blob仅支持digest寻址**。原因是对于经典分层结构，blob对外不会单独使用，需要通过manifest定位到。即**面向用户的manifest才需要用户可理解的reference(location)去寻址，底层存储blob通过manifest记录的digest(content)定位即可**。

{{< /admonition >}}



{{< admonition Info "有关于content discovery的扩展" >}}

OCI Distribution Spec中关于content discovery仅定义了**获取某个repository的tags** api，但仍有**获取到某个registry的repositories**的需求，目前大部分registry都实现了此api及其分页形式：`GET /v2/_catalog?n=<integer>&last=<string>`

{{< /admonition >}}

## 探索与实践

下面以ubuntu:21.04发布为例探索一下上述api的使用。

先看一下ubuntu:21.04的manifest，其中config与layers中的每个descriptor均为一个blob。

```json
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1462,
      "digest": "sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 31837572,
         "digest": "sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1"
      }
   ]
}
```

ubuntu manifest自身的digest为：`ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0`

### Push流程

参考[oras](https://github.com/oras-project/oras-go)封装了一下containerd的`remotes.PushContent`方法将`myregistry:5000/ubuntu:21.04` push到registry。将log-level设置成debug模式，我们可以看到containerd中相关api调用的track如下（仅以layer blob为例，省略掉config blob的日志信息）。

- Step1：Client端发起一个HEAD blob请求，server端返回404表明当前blob不存在。

```verilog
DEBU[0000] do request                                    digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json request.header.accept="application/vnd.docker.container.image.v1+json, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=HEAD size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"

DEBU[0000] fetch response received                       digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json response.header.content-length=157 response.header.content-type=application/json response.header.date="Sat, 21 Aug 2021 04:48:26 GMT" response.header.docker-distribution-api-version=registry/2.0 response.header.x-content-type-options=nosniff response.status="404 Not Found" size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"
```

- Step2：Client端调用POST blob获取upload session id，server端返回一个locator。

```verilog
DEBU[0000] do request                                    digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json request.header.user-agent=containerd/1.5.2+unknown request.method=POST size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/uploads/"

DEBU[0000] fetch response received                       digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json response.header.content-length=0 response.header.date="Sat, 21 Aug 2021 04:48:26 GMT" response.header.docker-distribution-api-version=registry/2.0 response.header.docker-upload-uuid=10729918-5b28-4a0b-b793-319d048a3b66 response.header.location="http://myregistry:5000/v2/ubuntu/blobs/uploads/10729918-5b28-4a0b-b793-319d048a3b66?_state=vQM2ysmno9_Yvwoz6NK0qgEKVG1aCm3GqKFHYoQEX6F7Ik5hbWUiOiJ1YnVudHUiLCJVVUlEIjoiMTA3Mjk5MTgtNWIyOC00YTBiLWI3OTMtMzE5ZDA0OGEzYjY2IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTA4LTIxVDA0OjQ4OjI2Ljc3MTY5MloifQ%3D%3D" response.header.range=0-0 response.header.x-content-type-options=nosniff response.status="202 Accepted" size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/uploads/"
```

- Step3：Client端调用PUT blob上传blob，server端返回上传成功。

```verilog
DEBU[0000] do request                                    digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json request.header.content-type=application/octet-stream request.header.user-agent=containerd/1.5.2+unknown request.method=PUT size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/uploads/10729918-5b28-4a0b-b793-319d048a3b66?_state=vQM2ysmno9_Yvwoz6NK0qgEKVG1aCm3GqKFHYoQEX6F7Ik5hbWUiOiJ1YnVudHUiLCJVVUlEIjoiMTA3Mjk5MTgtNWIyOC00YTBiLWI3OTMtMzE5ZDA0OGEzYjY2IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTA4LTIxVDA0OjQ4OjI2Ljc3MTY5MloifQ%3D%3D&digest=sha256%3Abf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"

DEBU[0000] fetch response received                       digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json response.header.content-length=0 response.header.date="Sat, 21 Aug 2021 04:48:26 GMT" response.header.docker-content-digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" response.header.docker-distribution-api-version=registry/2.0 response.header.location="http://myregistry:5000/v2/ubuntu/blobs/sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" response.header.x-content-type-options=nosniff response.status="201 Created" size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/uploads/10729918-5b28-4a0b-b793-319d048a3b66?_state=vQM2ysmno9_Yvwoz6NK0qgEKVG1aCm3GqKFHYoQEX6F7Ik5hbWUiOiJ1YnVudHUiLCJVVUlEIjoiMTA3Mjk5MTgtNWIyOC00YTBiLWI3OTMtMzE5ZDA0OGEzYjY2IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTA4LTIxVDA0OjQ4OjI2Ljc3MTY5MloifQ%3D%3D&digest=sha256%3Abf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"
```

- Step4：Client端调用HEAD manifest探测manifest的存在性，server端返回404表明当前manifest不存在。

```verilog
DEBU[0001] do request                                    digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json request.header.accept="application/vnd.docker.distribution.manifest.v2+json, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=HEAD size=529 url="http://myregistry:5000/v2/ubuntu/manifests/21.04"

DEBU[0001] fetch response received                       digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json response.header.content-length=95 response.header.content-type=application/json response.header.date="Sat, 21 Aug 2021 04:48:27 GMT" response.header.docker-distribution-api-version=registry/2.0 response.header.x-content-type-options=nosniff response.status="404 Not Found" size=529 url="http://myregistry:5000/v2/ubuntu/manifests/21.04"
```

- Step5：Client端调用PUT manifest上传，server端返回上传成功。

```verilog
DEBU[0001] do request                                    digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json request.header.content-type=application/vnd.docker.distribution.manifest.v2+json request.header.user-agent=containerd/1.5.2+unknown request.method=PUT size=529 url="http://myregistry:5000/v2/ubuntu/manifests/21.04"

DEBU[0001] fetch response received                       digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json response.header.content-length=0 response.header.date="Sat, 21 Aug 2021 04:48:27 GMT" response.header.docker-content-digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" response.header.docker-distribution-api-version=registry/2.0 response.header.location="http://myregistry:5000/v2/ubuntu/manifests/sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" response.header.x-content-type-options=nosniff response.status="201 Created" size=529 url="http://myregistry:5000/v2/ubuntu/manifests/21.04"
```

{{< admonition  note  "registry layer cache" >}}

OCI Distribution Spec仅定义了一组api规范，并没规定registry需要实现layer cache。只是docker registry实现了layer cache，这样往多个repository push相同的blob只有第一次需要真正的上传blob数据，而后续的上传的开销仅是若干个http请求及内部创建一个该blob digest到此repository的引用。

{{< /admonition >}}

总之，push流程会先上传blob，再上传manifest。

### Pull流程

封装了containerd `remotes.FetchHandler`与`images.ChildrenHandler`，以将`myregistry:5000/ubuntu:21.04` pull到本地为例展示一下api调用track。

- Step1：Client端HEAD一下ubuntu:21.04 manifest，server端返回相关的digest。

```verilog
DEBU[0000] do request                                    host="myregistry:5000" request.header.accept="application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=HEAD url="http://myregistry:5000/v2/ubuntu/manifests/21.04"
  
DEBU[0000] fetch response received                       host="myregistry:5000" response.header.content-length=529 response.header.content-type=application/vnd.docker.distribution.manifest.v2+json response.header.date="Sat, 21 Aug 2021 06:57:20 GMT" response.header.docker-content-digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" response.header.docker-distribution-api-version=registry/2.0 response.header.etag="\"sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0\"" response.header.x-content-type-options=nosniff response.status="200 OK" url="http://myregistry:5000/v2/ubuntu/manifests/21.04"
```

- Step2：Client端获取到ubuntu:21.04的manifest，server端返回manifest的内容、

```verilog
DEBU[0000] do request                                    digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json request.header.accept="application/vnd.docker.distribution.manifest.v2+json, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=GET size=529 url="http://myregistry:5000/v2/ubuntu/manifests/sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0"
  
DEBU[0000] fetch response received                       digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" mediatype=application/vnd.docker.distribution.manifest.v2+json response.header.content-length=529 response.header.content-type=application/vnd.docker.distribution.manifest.v2+json response.header.date="Sat, 21 Aug 2021 06:57:20 GMT" response.header.docker-content-digest="sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0" response.header.docker-distribution-api-version=registry/2.0 response.header.etag="\"sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0\"" response.header.x-content-type-options=nosniff response.status="200 OK" size=529 url="http://myregistry:5000/v2/ubuntu/manifests/sha256:ef8ee90cfa9cfc7c218586dea9daa6a8d1d191b3c73be143f4120fe140dae3d0"
```

- Step3：Client端从manifest解析出blobs，并发送GET blob请求，server端将blob的内容返回。

```verilog
------Requests for 2 blobs------
DEBU[0000] do request                                    digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json request.header.accept="application/vnd.docker.container.image.v1+json, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=GET size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"

DEBU[0000] do request                                    digest="sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1" mediatype=application/vnd.docker.image.rootfs.diff.tar.gzip request.header.accept="application/vnd.docker.image.rootfs.diff.tar.gzip, */*" request.header.user-agent=containerd/1.5.2+unknown request.method=GET size=31837572 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1"


------Responses for 2 blobs------

DEBU[0000] fetch response received                       digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" mediatype=application/vnd.docker.container.image.v1+json response.header.accept-ranges=bytes response.header.cache-control="max-age=31536000" response.header.content-length=1462 response.header.content-type=application/octet-stream response.header.date="Sat, 21 Aug 2021 06:57:20 GMT" response.header.docker-content-digest="sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481" response.header.docker-distribution-api-version=registry/2.0 response.header.etag="\"sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481\"" response.header.x-content-type-options=nosniff response.status="200 OK" size=1462 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:bf70ebd2c444440ae068c5ccea80e2087906a825ff1019a9f6d6cbb229e33481"

DEBU[0000] fetch response received                       digest="sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1" mediatype=application/vnd.docker.image.rootfs.diff.tar.gzip response.header.accept-ranges=bytes response.header.cache-control="max-age=31536000" response.header.content-length=31837572 response.header.content-type=application/octet-stream response.header.date="Sat, 21 Aug 2021 06:57:20 GMT" response.header.docker-content-digest="sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1" response.header.docker-distribution-api-version=registry/2.0 response.header.etag="\"sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1\"" response.header.x-content-type-options=nosniff response.status="200 OK" size=31837572 url="http://myregistry:5000/v2/ubuntu/blobs/sha256:4451f5c7eb7af74432585f5ebfbeb01bbfc87ec4a74dc93703bdd89330559cd1"
```

与push流程相反，pull操作先去获取manifest，解析出blobs后再去拉取对应的的blob。

## 参考文献

- [Oras](https://github.com/oras-project/oras-go)
- [Containerd remotes](https://github.com/containerd/containerd/tree/main/remotes)
- [Containerd images](https://github.com/containerd/containerd/tree/main/images)
- [OCI Distribution Spec](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
