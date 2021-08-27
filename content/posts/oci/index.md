---
weight: 20
title: "OCI 标准探究"
subtitle: ""
date: 2021-08-26T20:57:13+08:00
lastmod: 2021-08-26T20:57:13+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.jpeg"

tags: ["container"]
categories: ["整理与总结"]


toc:
  auto: false
lightgallery: true
license: ""
---
> The **Open Container Initiative** is an open governance structure for the express purpose of creating open industry standards around container formats and runtimes.
<!--more-->

## 内容概览

OCI定义了如下几个标准：

### OCI Runtime-Spec

定义容器**配置**，**运行**环境及**生命周期**规范。

### OCI Image-Spec

定义image格式规范。

- manifest
- filesystem layers
- image index

### OCI Distribution-Spec

定义内容分发的一组标准API。

### OCI Artifacts

OCI Image-Spec的通用扩展，将普通artifact按照image形式发布到distribution上。



## 对标准的认识

标准是什么？字面上的意思是指**衡量、区别事物的准则**。遵循某一标准的事物可以达到某种程度上的统一。

最初对标准的认识就是两个字：**抽象**。似乎人们天生对于抽象的东西的初印象都是十分抗拒的，而具体的东西更有助于人们去理解新事物。随着具有相同功能的产品越来越多，人们对这些产品的选择也越来越多，但是当我们从一个产品的使用切换到另一个产品的使用上却犯了难：**完成同样的事，操作或行为却大相径庭**。而标准正是为了统一这类产品而生，即我们只需要了解相应的标准即可完成想要的功能。完成此功能的产品可以很多，但他们需要满足这个标准，而一旦他们满足了这个标准，我们就可以方便地在这些产品中进行切换。

上面的描述还是过于抽象，举个生活中最简单的例子：**数据线接口**。如果没有标准，每个设备的接口的样式可以五花八门，试想一下某款鼠标为了适配几千种笔记本的接口形状，也需要提供成千上万种的数据线，这简直就是灾难。而通过制定一组标准接口(VGA, HDMI, USB, TypeC)，所有厂商制造笔记本时受限于这些标准接口，这样外设便可以做到很通用，即便存在为数不多的几种不同标准，一个绿联就可以轻松搞定。

总之标准就是为了提升通用性，就好比多态一样，**实现可以多种多样，但都实现统一的接口，而这接口便是完成某个功能的最短路径**。

在工作中受益于标准化的案例就是**OpenTracing**，这是一套分布式追中的标准规范，定义了一组接口规范。在开发过程中，无需陷于产品的选型，直接使用这组规范接口，而只要是实现了此规范的产品（jaeger, skywalking, zipkin等）都可以在项目中轻松切换。

有了OCI Spec，**我们可以使用contained来替换docker，也可以将镜像仓库从docker registry迁移到harbor**。OCI让容器化变得更通用。

## 系列文章
- [OCI Distribution Spec探索与实践](../oci-distribution/)
- [OCI Image Format Spec探索与实践](../oci-image/)
- [私有化交付下的应用打包方案](../oras/)
