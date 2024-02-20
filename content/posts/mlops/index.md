---
weight: 20
title: "MLOps 科普"
subtitle: ""
date: 2022-07-20T19:59:58+08:00
lastmod: 2022-07-20T19:59:58+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "arch.jpg"

tags: []
categories: []


toc:
  auto: false
lightgallery: true
license: ""
---

DevOps in Machine Learning.

<!--more-->

## ML 面临的问题

{{< image src="environ.jpg" caption="MLOps 工具链" width=800 height=400 >}}

- 利用机器学习解决问题的完整系统，关于模型训练的代码其实只占很少一部分。
- 为了系统中各个模块合作，各类胶水代码会有很多反模式设计，很难维护，留下很多隐藏的技术债。

## 什么是 MLOps？

MLOps 是机器学习时代的 DevOps。它的主要作用就是**连接模型构建团队、业务团队和运维团队，建立起一个标准化的模型开发、部署与运维流程**，使得企业组织能更好的利用机器学习的能力来促进业务增长。

MLOps 与传统 DevOps 最大不同是管理的最基本要素从**代码**维度延展到**数据**与**模型**维度。

{{< image src="element.png" caption="MLOps 中的三大元素" width=800 height=400 >}}

三者之间的信息流转一般会牵扯到不同角色。

{{< image src="role.png" caption="MLOps 中角色交互" width=800 height=400 >}}

如何建立自动化流程来打破角色、组织之间的边界，并提供可持续交付能力是 MLOps 要解决的问题。

{{< image src="process.png" caption="MLOps 流程闭环" width=800 height=400 >}}

**MLOps 概念参考链接**
- https://ml-ops.org/
- https://en.wikipedia.org/wiki/MLOps
- [ MLOps：机器学习中的持续交付和自动化流水线](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning?hl=zh-cn)
  - Google将MLOps流水线按照自动化程度分了3个Level，可以参考。
- [Machine learning operations (MLOps) Resource Center from Azure](https://azure.microsoft.com/en-us/services/machine-learning/mlops/#resources)
- [ 吴恩达：从以模型为中心到以数据为中心的AI](https://www.bilibili.com/video/BV1WB4y1P7Hw)
  - 吴恩达同时也有一门关于[MLOps的课程](https://www.bilibili.com/video/BV1ji4y197pv?spm_id_from=333.999.0.0)，讲解如何E2E训练、部署、监控模型。

## MLOps 概念辨析

DevOps 以自动化的方式将开发与部署流程结合起来，打破传统模式下各部门间的相互制约， 提升了软件交付效率。此后，越来越多的跨职能协作自动化方案被提出，形成 XOps 趋势。

{{< image src="xops.png" caption="XOps 功能" width=800 height=400 >}}


- **DevOps**：更快地交付软件。
  - 一系列旨在消除开发和运维团队之间障碍的实践，以便更快地构建和部署软件。它通常会被工程团队所采用，包括 DevOps 工程师、基础设施工程师、软件工程师、站点可靠性工程师和数据工程师。

- **DataOps**：更快地交付数据。
  - 一系列旨在提高数据分析质量并缩短分析周期的实践。DataOps 的主要任务包括数据标记、数据测试、数据管道编排、数据版本控制和数据监控。分析和大数据团队是 DataOps 的主要操作者，但是任何生成和使用数据的人都应该采用良好的 DataOps 实践。这包括数据分析师、BI 分析师、数据科学家、数据工程师，有时还包括软件工程师。

- **MLOps**：更快地交付机器学习模型。
  - 一系列设计、构建和管理可重现、可测试和可持续的基于 ML 的软件实践。对于大数据 / 机器学习团队，MLOps 包含了大多数 DataOps 的任务以及其他特定于 ML 的任务，例如模型版本控制、测试、验证和监控。

- **AIOps**：利用 AI 的功能增强 DevOps。
  - 有时人们错误地将 MLOps 称为 AIOps，但它们是完全不同的。AIOps 平台利用大数据、现代机器学习以及其他先进的分析技术，直接或间接地增强 IT 运维（监控、自动化和服务台），具有前瞻性、个性化以及动态洞察力。因此，AIOps 通常是利用 AI 技术来增强服务产品的 DevOps 工具。AWS Cloud Watch 提供的报警和异常检测是 AIOps 的一个很好的例子。

{{< admonition warning "AIOps 命名陷阱" >}}

AIOps 并不符合 XOps 跨职能协作的语义。

{{< /admonition >}}


## MLOps 的实现
MLOps 的涉及面十分广泛，涵盖机器学习全链路的所有工具链。但一般来说，MLOps 主要实现方式有两种：
- 规范流：和 DevOps 类似，MLOps 并不具体指某个单一的系统，而是强调各组织之间合作的规范性。
- 平台流：在 MLOps 这个方向，已经有很多开源或者创业团队在做，很多实现了一个统一的平台入口，进行数据、模型、部署的管理。

下面列举几个 MLOps 开源方案。

**[Kubeflow](https://www.kubeflow.org/docs/started/architecture/)**

{{< image src="kubeflow.png" caption="kubeflow 架构图" width=800 height=400 >}}

> The Kubeflow project is dedicated to making deployments of machine learning (ML) workflows on Kubernetes simple, portable and scalable. Our goal is not to recreate other services, but to provide a straightforward way to deploy best-of-breed open-source systems for ML to diverse infrastructures. Anywhere you are running Kubernetes, you should be able to run Kubeflow.

- 以 Google 为首，目前 star 数最多，是影响比较大的 MLOps 云原生平台。
- 在 K8s 云原生基础上，集成必要的开源组件，来管理模型开发的流程。
- 以 Operator 形式来管理不同训练架构在 K8s 上执行的训练任务。

**[ClearML](https://github.com/allegroai/clearml)**

{{< image src="clearml.png" caption="clearML 架构图" width=800 height=400 >}}

> ClearML is a ML/DL development and production suite, it contains FOUR main modules:
>* [Experiment Manager](https://github.com/allegroai/clearml#clearml-experiment-manager) - Automagical experiment tracking, environments and results
>* [MLOps](https://github.com/allegroai/clearml-agent) - Orchestration, Automation & Pipelines solution for ML/DL jobs (K8s / Cloud / bare-metal)
>* [Data-Management](https://github.com/allegroai/clearml/blob/master/docs/datasets.md) - Fully differentiable data management & version control solution on top of object-storage (S3 / GS / Azure / NAS)
>* [Model-Serving](https://github.com/allegroai/clearml-serving) - *cloud-ready* Scalable model serving solution!
   ✨ **Deploy new model endpoints in under 5 minutes** ✨
   💪 includes optimized GPU serving support backed by Nvidia-Triton 🦾
   📊 **with out-of-the-box Model Monitoring** 😱


更多 MLOps 平台可以参考：[Top 10 Open Source MLOps Tools](https://thechief.io/c/editorial/top-10-open-source-mlops-tools/)。


### 模型服务

模型服务(Model Serving)是 MLOps 流程里的最后阶段。通常包含模型 PPL 搭建，封装微服务，部署到目标集群等。这一部分与运维相关性最大，也是最灵活，通用性要求最高的部分。

下面列举两个开源的解决方案。

**[BentoML](https://github.com/bentoml/BentoML)**

{{< image src="bentoml.jpg" caption="BentoML 工作内容" width=800 height=400 >}}

- BentoML 可以通过简单的方法、注解定义将模型的推理行为对外暴露成 REST 接口，并其它基础库封装成一个镜像制品，称之为 Bento。
- [Yatai](https://github.com/bentoml/yatai) 提供了简单的 UI 控制台，将所有模型、Bento 制品管理起来，支持选择对应版本部署到K8s集群上，并提供对应的日志、监控、跟踪等运维工具。

**[KServe](https://kserve.github.io/website/0.8/)**

{{< image src="kserve.jpg" caption="Kserve 架构图" width=800 height=400 >}}

- KServe 之前叫 KFServing，是 Kubeflow 生态中的一部分，现做为独立项目开发。
- KServe 利用的开源技术栈比较多，比如 Knative + Istio，之前比较重，现在已经变成可选。
- KServe 利用 Knative 实现了模型服务的自动扩缩容，实现了 Serverless 的能力。
- KServe 为了解决多模型的联动的问题，提了一个新的概念：[ModelMesh](https://kserve.github.io/website/0.8/modelserving/mms/modelmesh/overview/)


**Model Serving 相关文章**

- [Seldon Core: Blazing Fast, Industry-Ready ML](https://docs.seldon.io/projects/seldon-core/en/stable/workflow/github-readme.html)
- [Machine Learning Model Serving Overview (Seldon Core, KFServing, BentoML, MLFlow)](https://medium.com/everything-full-stack/machine-learning-model-serving-overview-c01a6aa3e823)
- [Paddle Serving](https://github.com/PaddlePaddle/Serving/blob/v0.8.3/README_CN.md)


## 参考文章

- [Hidden Technical Debt in Machine Learning Systems](https://papers.nips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf)
- [Hidden Technical Debt in Machine Learning Systems 阅读笔记](https://zhuanlan.zhihu.com/p/111694069)
- [Why you Might Want to use Machine Learning ](https://ml-ops.org/content/motivation)
- [MLOps Principles](https://ml-ops.org/content/mlops-principles)
- [从小作坊到智能中枢: MLOps简介](https://zhuanlan.zhihu.com/p/357897337)
- [Continuous Delivery for Machine Learning](https://martinfowler.com/articles/cd4ml.html)
- [MLOps 选型指导](https://ml-ops.org/content/mlops-stack-canvas)
- [ MLOps: 数据编程时代的工具链创业机会](https://zhuanlan.zhihu.com/p/375745901)

