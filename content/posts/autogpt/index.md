---
weight: 20
title: "AutoGPT初体验"
subtitle: ""
date: 2023-04-20T18:38:39+08:00
lastmod: 2023-04-20T18:38:39+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["tools"]
categories: ["探索与实战"]


toc:
  auto: false
lightgallery: true
license: ""
---

嫌一句句地问ChatGPT太麻烦？试试AutoGPT吧！

<!--more-->


## 背景

近来ChatGPT引爆了生成式AI的浪潮，基于ChatGPT的应用喷涌而出。ChatGPT提供一种交互式的聊天方式回答使用者提出的问题，通常这种答案不够全面，需要提供多轮提示才可以获得满足预期的结果。

这存在几个限制：

- 需要大量的人工介入。
- 非专业人士很难给出专业的提示。
- 很难给出很全面的提示。


为了可以一次性获得更全面的内容，AutoGPT粉墨登场。



## AutoGPT是什么

AutoGPT实际上是个**自主人工智能代理**，它的目标是**make GPT-4 fully autonomous**。

下面总结了对AutoGPT的一些定位：

- 它可以通过自主地链接思维来实现特定的目标，尝试**将目标分解为子任务**，并使用互联网和其他工具进行自动循环以实现它。
- 它可以**自我提示**并生成完成任务所需的每个必要提示。
- 它依赖于AI代理来根据预定义的目标和规则**做出决策并采取行动**。
- 它拥有互联网访问权限，能够读写文件，并且**具备短期和长期记忆力**，以理解已经完成的工作。
- 它最令人印象深刻的一点是其**自主反思和改进行为**的能力。这是通过一个反馈循环实现的，该循环使用计划、批评、行动、阅读反馈和再次计划，就像拥有编码技能的个人教练一样。



## AutoGPT vs ChatGPT

ChatGPT以交互式的方式回答使用者提出的问题，从而实现一个完整目标。而AutoGPT通过对目标进行分解，并不断进行反馈迭代，从而自主地解决整个目标。

{{< image src="chatgpt.png" caption="ChatGPT交互方式" width=600 height=200 >}}

{{< image src="autogpt.png" caption="AutoGPT交互方式" width=600 height=200 >}}


从工作原理上看，**ChatGPT有点类似于DNS迭代查询，而AutoGPT类似于DNS递归查询。**



## AutoGPT的体验

目前基于AutoGPT的产品比较多，主流的是[Auto-GPT](https://github.com/Significant-Gravitas/Auto-GPT)，但其并没有提供一个易用的用户界面，因此使用[cognosys](https://www.cognosys.ai/)进行体验。截止目前（2023.04.19），cognosys可以免费试用。

从体验上来看，AutoGPT主要将目标拆分为若干个子任务，分析返回的结果并进行追问，形成完整的目标结论。

其拆分子任务的策略也具有一定的规律：

- 搜集数据
- 分析数据
- 给出方案

而反馈循环一般是围绕上一轮结果中一些关键点开展，比如风险控制、可行性分析、步骤细化等。

下面将从两个案例分析AutoGPT自学习能力，为节省篇幅，省略回答内容。



### 案例一：如何构建开发者门户？

**第一轮：任务拆解**

1. 研究现有的开发门户并分析它们的特性和功能，以确定最重要和最相关的内部门户。
2. 确定组织对开发门户的具体需求和要求，包括用户角色、访问控制、协作工具以及与现有系统的集成。
3. 设计和开发内部开发门户，使用以用户为中心的方法确保可用性和可访问性，并与具有代表性的用户组一起测试门户以确保满足他们的需求和期望。

**第二轮：方案细化**

1. 创建可集成到内部开发门户以增强其功能和可用性的潜在第三方应用程序列表。
2. 制定培训用户如何有效使用内部开发门户的计划，包括创建用户指南和举办培训课程。
3. 对内部开发门户进行安全审计，以确保其符合数据保护和访问控制的行业标准。

可以看出AutoGPT会按照上述策略拆分成子任务，并根据子任务返回的结果进行反馈追问，使得目标的解决方案更全面。

### 案例二：如何把大象放进冰箱？

**第一轮：任务拆解**

1. 识别和分析一只普通大象的尺寸和尺度，以确定需要多大的冰箱才能容纳一只大象。
2. 制定一个逐步计划，以安全和人道的方式将大象运输到冰箱中，并考虑潜在的障碍和挑战。
3. 为整个过程创建详细的成本分析，包括合适的冰箱、运输以及任何必要的设备或人员的成本。

**第二轮：成本预估**

1. 研究并确定可容纳大象的合适冰箱的最佳选择，并确定其相关成本。
2. 进行市场调查以确定最具成本效益的大象运输方式并确定相关成本。
3. 为运输过程中可能出现的意外费用和不可预见的费用制定预算。

**第三轮：风险评估**

1. 考虑到大象的体型和健康状况，研究并确定安全和人道地镇静运输大象的最佳方法。
2. 识别和分析运输过程中可能出现的潜在障碍和挑战，例如狭窄的道路或桥梁，并制定应急计划来解决这些问题。
3. 进行风险分析以识别潜在危险并制定减轻危险的策略，确保大象和所有参与运输过程的人员的安全。


**第四轮：可行性保障**

1. 进行可行性研究以确定将大象运输到冰箱中的伦理和实际影响，同时考虑动物福利和公众认知。
2. 确定并与合格的专业人员团队合作，包括兽医和运输专家，以确保大象的安全和人道运输。
3. 制定沟通计划，告知利益相关者（包括公共和动物福利组织）运输过程，并解决他们可能有的任何疑虑或问题。</summary>


针对此问题，ChatGPT给出了如下的答案。

1. 打开冰箱门。
2. 把大象的四条腿一只一只地放进冰箱里。
3. 仔细压紧大象的身体，推进冰箱里。
4. 关上冰箱门，确保大象完全被封闭在里面。


给出“标准答案”背后的原因值得我们思考，当然只能简单的猜一猜：

- 它能听理解人类的玩笑话。
- 它觉得这是不可能完成事件所以回答的比较敷衍。
- 他无法详细回答这么宽泛的问题呢。

## 使用总结

AutoGPT以**自主型AI代理**的方式不断进行自强化，减少了人工干预。它进一步将互联网与AI结合起来，从一个应答式的AI工具向提供通用解决方案的AI应用迈进。相信在不久的将来，它会渗透到各行各业。

## 引用

- [Why You Need To Know About AutoGPTs](https://medium.com/the-generator/why-you-need-to-know-about-autogpts-89289c88093f)
- [AUTO-GPT VS CHATGPT: HOW DO THEY DIFFER AND EVERYTHING YOU NEED TO KNOW](https://autogpt.net/auto-gpt-vs-chatgpt-how-do-they-differ-and-everything-you-need-to-know/)
- [AUTOGPT: THE AI THAT CAN SELF-IMPROVE IS SCARY!](https://autogpt.net/autogpt-the-ai-that-can-self-improve-is-scary/)
- [INTRODUCTION TO AUTOGPT](https://autogpt.net/autogpt-step-by-step-full-setup-guide/)
- 封面图引自 [Auto GPT vs ChatGPT: What’s the difference?](https://openaimaster.com/auto-gpt-vs-chatgpt-whats-the-difference/)
