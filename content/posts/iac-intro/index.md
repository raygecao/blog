---
weight: 20
title: "基础设施即代码 (IaC)"
subtitle: ""
date: 2023-03-03T22:22:39+08:00
lastmod: 2023-03-03T22:22:39+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["iac"]
categories: ["整理与总结"]


toc:
  auto: false
lightgallery: true
license: ""
---
*Infrastructure as code is the process of managing and provisioning computer data centers through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.*
<!--more-->

## 传统基础设施管理方案

传统的基础设施构建及运维通常由IT运维团队手动维护，这存在以下问题：

- 交互式管理方式会带来与人相关的问题。
  - 容易引入human error，基础设施在整个系统的最底层，一些误配可能会**降低系统的可用性**。
  - 手动管理基础设施步骤较多，人工**操作效率低**。
  - 不同运维人员有着不同的经验背景、细心程度，**难以保证基础设施一致性**。
- 手动管理**缺乏合适/统一的审计手段**。当基础设施出现故障时，很难定位到当前环境经过了哪些操作、该问题是由哪一步操作引起的。
- 在云原生的环境下，基础设施的规模会极为庞大，手动管理基础设施会**增加运维团队的负担与成本**。
- 基础设施会成为开发团队与IT运维团队的壁垒，研发团队无法快速构建出一个应用部署环境，必须依托于运维团队去构建；而运维团队又会付出大量的时间去应付各个研发团队基础设施运维需求。团队耦合度高，**降低了产品迭代效率**。

为了解决基础设施构建难、配置难、追溯难等问题，IaC将基础设施代码化，以替代传统的手动构建方式。

## IaC 概述

IaC利用human-readable配置文件管理基础设施，将基础设施的管理转化为代码的维护。IaC保证了基础设施供应的一致性，大大减少了人工成本及人为错误操作带来的风险。

IaC的强大体现在以下几个方面。

### 参数化

在产品迭代的不同阶段通常会**维护多个环境**，比如在开发阶段、QA阶段、上线阶段分别对应着dev、stage、prod等多个环境。这些环境上的基础设施是极为相似的，但通常略有不同。

比如dev环境一般会使用较新的版本、较新的技术、以及较为少量的资源Quota；而prod环境一般会使用更为稳定的版本以及更多的资源Quota。

我们当然可以维护多个配置文件与各个环境一一对应。但这一方面引入了大量的管理开销，另一方面由于存在大量代码冗余，每个通用的修改要复制很多次。

为了提供更灵活的配置能力与更强的复用能力，IaC通常会设计成参数化形式。即**将不变的、安全的部分做成模板，将可变的、不安全的部分作为配置值渲染到模板中**。这将多个配置文件的管理转换成单个模板文件及多个参数文件的管理。

使用参数化的IaC，可以将研发团队与运维团队进一步解耦。运维团队只需要提供一份经过充分测试及高度文档化的配置模板，研发团队可以对其按需进行参数化配置构建基础设施，这大大提升协作效率。

### IaC on VCS

VCS（git）为IaC提供版本管理的能力，具备如下优势：

- 支持多人协作。
- 便于追溯配置变更。
- 对基础设施进行版本化管理，易于实现不可变基础设施。
- 是实现DevOps、GitOps工作流的基础。
- 利用VCS的特性降低基础设施管理的复杂度。如基础设施操作权限映射到VCS的权限控制、使用ChatOps实现相关的审批流程等。

{{< admonition info "不可变基础设施" >}}

基础设施的升级采用**安装新版本->切换流量->卸载旧版本**的方式，避免原地升级不完全引起的状态不一致。

{{< /admonition >}}

### IaC in DevOps

IaC使得基础设施的构建与运维变得自动化，结合CI/CD/Pipeline可以对基础设施进行自动化构建、校验、发布与部署，减少人工干预，加速产品迭代速度。

> With the rise of DevOps and SRE approaches, infrastructure management becomes codified, automatable, and software development best practices gain their place around infrastructure management too. On one hand, the daily tasks of classical operations people changed and are more similar to traditional software development. On the other hand, software engineers are more likely to control their whole DevOps lifecycle, including deployments and delivery.    —— gitlab

> IaC makes the environment part of a software release, managed as part of the CI/CD pipeline —— auqa



## 使用场景举例

- IT运维团队提供一份通用的配置模板，DEV团队可以快速在测试环境上构建出一套基础设施。
- DEV团队可以以提交MR的形式对基础设施变更（如扩容）进行申请，由IT运维团队进行审批。

- 基础设施在不同环境（dev/stage/prod）下的迭代演进。如在dev环境下进行研发，经stage环境验证，通过后部署到prod环境。IaC保证同一版本基础设施在不同环境下部署的一致性。



## Terraform

[Terraform](https://www.terraform.io/) 是HashiCorp推出的一款IaC产品，并具备以下特性：

- 定义一种HCL声明式语言，用以定义基础设施供应方式。
- 提供多种provider对接多个云平台，provider通过集成云平台的api将code转化为infrastructure。
- 定义了**写配置 -> 构建执行计划 -> 执行部署**工作流

{{< image src="arch.png" caption="Terraform架构图，引自[terraform intro](https://developer.hashicorp.com/terraform/intro)" width=800 height=400 >}}


Terraform具备以下优势：

- **支持模块化**，以提升代码复用率。模块的版本化有助于实现不可变基础设施，减少基础设施升级的复杂度。
- **声明式配置语言**屏蔽了基础设施部署细节，保证了部署的一致性。
- 依赖管理机制可以保证基础设施**按正确顺序部署**。
- 使用VCS track配置的变更，通过state文件track实际部署的变化。
- Terraform内部维护资源的依赖关系，可以**并发进行资源部署**，提升基础设施安装速度。
- 即可以管理低级组件（计算实例、存储、网络等），又可管理高级组件（DNS、SaaS等）。
- 插件式provider**便于扩展**，易于将terraform扩展到自研平台。

为强化基础设施的管理，HashiCorp还提出了**terraform cloud**，用以支持：

- 基础设施自动化供应
- 安全性管理
  - 密钥管理
  - 远程状态管理
  - 私有化registry
  - RBAC
- 审计、成本预估、大规模团队协作等能力

## 参考文档

- [Infrastructure as code wikipedia](https://en.wikipedia.org/wiki/Infrastructure_as_code)
- [What is Terraform?](https://developer.hashicorp.com/terraform/intro)
- [Infrastructure as Code and DevOps: DevOps Automation Reloaded](https://www.aquasec.com/cloud-native-academy/devsecops/infrastructure-as-code-devops/)
- [Gitlab Infrastructure management](https://docs.gitlab.com/ee/user/infrastructure/index.html)

- 封面图引自 https://www.cisco.com/c/en/us/solutions/cloud/what-is-iac.html
