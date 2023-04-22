---
weight: 20
title: "安利一款好用的ChatGPT Web"
subtitle: ""
date: 2023-04-22T12:47:02+08:00
lastmod: 2023-04-22T12:47:02+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["tools"]
categories: ["工具党"]


toc:
  auto: false
lightgallery: true
license: ""
---

无需科学上网，可以复用OpenAI账号的Chat Web。
<!--more-->

## 项目介绍

近来发现一个好用的[ChatGPT网页应用](https://github.com/Yidadaa/ChatGPT-Next-Web)。

具有以下优势：

- 可以使用[vercel](https://vercel.com/)一键部署一个chat页面，也可以一条命令在本地部署，操作十分方便。
- 无需连vpn即可访问，便于与不会科学上网、没OpenAI账号的广大亲朋好友分享使用。
- 可以通过密码认证的方式共享构建者的账号，也可以自己配置OpenAI API Key复用此UI。
- 以markdown方式导出聊天记录，无需额外的浏览器插件。


示例：以Markdown导出聊天内容。

```markdown
# 开发门户分析工具


## 来自你的消息:

开发者门户中，scorecard有什么用？

## 来自 ChatGPT 的消息:

开发者门户中的scorecard可以用来衡量和监控开发者在开发过程中的整体表现。它可以跟踪开发者在某个时间段内的代码提交数量、错误率、代码质量、代码审查通过率等指标，帮助团队管理者或上级领导更好地评估开发者的表现和绩效，并及时发现和解决问题。通过scorecard的监控和评估，开发团队可以更好地协同工作，提高项目交付质量和效率。

## 来自你的消息:

如何开发一个scorecard?

## 来自 ChatGPT 的消息:

开发一个scorecard需要以下几个步骤：

1. 确定需要衡量的指标。这些指标应该与团队或项目的目标相关，并可以帮助衡量开发者的绩效。例如，代码质量、代码提交数量、错误率等。

2. 确定指标的权重。根据指标的重要性来赋予不同的权重，在计算总分时需要考虑这些权重。

3. 设计评分标准。根据指标的权重和分值范围来设计评分标准。例如，代码质量评分可以根据静态代码分析的结果，分为A级、B级、C级等，每个级别赋予不同的分值。

4. 开发评估工具。可以通过开发评估工具来自动化收集和评估指标。例如，代码质量可以通过静态代码分析工具来评估；代码提交数量可以通过版本控制工具来统计。

5. 计算总分并生成scorecard。根据指标的权重和评分标准，计算每个开发者的总分，并生成scorecard。scorecard可以是一个表格或图表，可视化地展示开发者的绩效。
```

## 注意事项

- 密码谨慎泄露，避免太多人消耗你的token。
- 本地搭建时，需要指定OpenAI proxy避免无法直连，使用docker本地部署可以使用如下命令：
```shell
docker run -it --rm -p 3000:3000 \
   -e OPENAI_API_KEY=<YOUR OPENAI KEY> \
   -e CODE=<YOUR PASSWARD> \
   -e BASE_URL="api.openai-proxy.com"\
   yidadaa/chatgpt-next-web
```

## Demo site
- 官方：[https://chat-gpt-next-web.vercel.app](https://chat-gpt-next-web.vercel.app/)
- 墙内：[https://chat.raygecao.cn](https://chat.raygecao.cn/)


