---
weight: 20
title: "开发利器集锦"
subtitle: ""
date: 2024-02-18T19:26:48+08:00
lastmod: 2024-02-18T19:26:48+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.jpg"

tags: ["tools"]
categories: ["工具党"]


toc:
  auto: false
lightgallery: true
license: ""
---

工欲善其事，必先利其器。

<!--more-->

## Unix 通用配置
- [Oh My Zsh](https://ohmyz.sh/)：专业的 zsh 管理工具，有丰富的插件、主题及好用的 git alias 等。
- [agnoster-zsh-theme](https://github.com/agnoster/agnoster-zsh-theme)：好用的 zsh theme，基于历史记录的命令行即时提示。
- [fzf](https://github.com/junegunn/fzf)：命令行模糊搜索工具，UI 展示出模糊搜索到的历史记录。
- [mosh](https://mosh.org/)：移动的 shell，增强型 ssh。支持间歇式连接，可以避免 ssh 网络切换/超时导致 session 断开的问题。
- [tmux](https://github.com/tmux/tmux/wiki)：终端复用器，在单个 shell 终端上管理多个程序 session，配合 mosh 使用更香。
- [tig](https://jonas.github.io/tig/)：git 的文本模式接口，交互性与易用性大大提升。
- [sshuttle](https://github.com/sshuttle/sshuttle)：ssh 透明代理，浏览器访问专网时时无需设置端口转发，连洞神器。

## 容器相关工具
- [dive](https://github.com/wagoodman/dive)：镜像的 layer 可视化分析工具，展示各层内容以便优化镜像大小。
- [crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md)：镜像的管理工具，主要用来与 registry 进行交互。
- [k9s](https://k9scli.io/)：k8s 可视化管理工具。

## 多媒体相关
- [snipaste](https://www.snipaste.com/)：可以悬浮铺开的截图工具。
- [sqoosh](https://squoosh.app/)：图片压缩工具，写文档/博客必备。
- [licecap](https://www.cockos.com/licecap/)：捕获桌面动作的 gif 生成器，写文档/博客必备。
- 图表
  - [excalidraw](https://excalidraw.com/)：在线手绘风绘图工具。
  - [drawio](https://app.diagrams.net/)：在线流程图绘制工具。
  - [processon](https://www.processon.com/)：在线制作流程图与思维导图。
  - [D2](https://d2lang.com/)：可编程的声明式图表。
- 图标库
  - [fontawesome](https://fontawesome.com/search)
  - [iconfont](https://www.iconfont.cn/)
  - [heroicons](https://heroicons.com/)
  - [terrastruct](https://icons.terrastruct.com/)
  - [mdi icon](https://pictogrammers.com/library/mdi/)
- 图片库
  - [freepik](https://www.freepik.com/)
  - [unsplash](https://unsplash.com/)
- 网页设计
  - [onepagelove](https://onepagelove.com/)
  - [screenlane](https://screenlane.com/)

## 效率工具
- [codimd](https://github.com/hackmdio/codimd?tab=readme-ov-file)：可协作的实时 markdown 编辑器，需私有化部署，支持 markdown slide 制作。
- [Alfred](https://www.alfredapp.com/)：流行的 MAC 搜索工具，可以集成很多实用小工具，如单位转换、翻译、搜索引擎等。
- [Navicat](https://navicat.com/en/)：数据库管理 GUI。
- [ByteBase](https://cn.bytebase.com/)：开源的数据库 DevOps 工具，使数据库与 VCS 集成，实现 Database as Code。
- 正则表达式
  - [正则表达式手册](https://tool.oschina.net/uploads/apidocs/jquery/regexp.html)
  - [regexper](https://regexper.com/)：可视化展示正则表达式。
  - [rubular](https://rubular.com/)：正则表达式匹配校验器。
- [carbon](https://carbon.now.sh/)：代码图片化工具。
- [dot-to-ascii](https://dot-to-ascii.ggerganov.com/)：基于字符的流程标识，常用于高大上的代码注释中。
- [replit](https://replit.com/)：在线 IDE，丰富的代码提示虐杀 go playground。
- [Reactive Resume](https://rxresu.me/)：开源的简历制作工具。

## 网站相关

- [vercel](https://vercel.com/)：个人网页的部署工具。
- [github pages](https://pages.github.com/)：静态网页的部署工具。
- [next.js](https://nextjs.org/)：易上手的的 React web 应用框架。结合 vercel、nextAuth.js 一站式建站。
- [hugo](https://gohugo.io/)：静态网页生成器，搭建个人博客不二之选。
