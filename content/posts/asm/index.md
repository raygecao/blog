---
weight: 20
title: "汇编入门速记"
subtitle: ""
date: 2021-06-27T14:25:41+08:00
lastmod: 2021-06-27T14:25:41+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["汇编"]
categories: ["整理与总结"]


toc:
  auto: false
lightgallery: true
license: ""
---

参考王爽[《汇编语言》](https://book.douban.com/subject/25726019/)

<!--more-->

- CPU读写外部设备需要如下交互信息（分别对应三大逻辑总线）

  - **地址信息**：存储单元的地址。
  - **控制信息**：器件的选择，读或写的命令。
  - **数据信息**：读或写的数据。

- 寄存器的作用

  - CPU可直接读写的部件，可以通过改变寄存器的内容实现对CPU的控制。

- 寄存器cheat sheet

  | 名称  | 类型       | 作用                                                         |
  | ----- | ---------- | ------------------------------------------------------------ |
  | AX    | 通用寄存器 | 用于一般的数据传递                                           |
  | BX    | 通用寄存器 | 用于表示偏移地址，如`mov ax [bx]`将ds:[bx]中的内容送入ax     |
  | CX    | 通用寄存器 | 用于存储循环变量，cx中的值会影响loop指定的执行结果，cx为零时会跳出循环 |
  | SI/DI |            | 配合BX指明内存单元的偏址，如`[bx+si+idata]`                  |
  | SP    |            | 存放栈顶的偏移地址                                           |
  | IP    |            | 指令指针寄存器，存放代码段的偏址                             |
  | BP    |            | 栈内偏址，主要用于指向/寻找栈内的某个地址的元素              |
  | DS    | 段寄存器   | 存放数据段的寄存器，配合[addr]做段内寻址                     |
  | SS    | 段寄存器   | 存放栈的段地址，配合SP/BP做段内寻址                          |
  | CS    | 段寄存器   | 存放代码段的寄存器，配合IP做段内寻址                         |
  | Flag  | 标志寄存器 | 用来存储相关指令的某些执行结果，从而为CPU相关指令提供行为依据 |

  | 标志寄存器标志位 | 作用                                                         |
  | ---------------- | ------------------------------------------------------------ |
  | ZF               | 零标志位，记录指令执行结果是否为0                            |
  | PF               | 奇偶标志位，记录相关指令执行后，所有bit位中1的个数是否为偶数 |
  | SF               | 符号标志位，记录相关指令执行后，结果是否为负                 |
  | CF               | 进位标志位，在无符号数运算时，记录了运算结果的最高有效位的进位值 |
  | OF               | 溢出标志位，用于标识指令执行结果是否溢出                     |
  | DF               | 方向标识位，在串处理指令中，控制每次操作后si, di的增减       |
  | TF               | 调试标志位，TF=1时，表示进入单步执行                         |
  | IF               | 中断允许标志位，表示能否接收外部中断请求，为1时能响应外部中断，反之屏蔽 |

- 8086CPU是16位机，但却有20位地址总线（1MB的寻址能力），无法做到简单的一一映射，其合成物理地址的方式如下

  - 通过两个地址16位地址合成一个20位地址：**物理地址=基址+偏址**。
  - 在8086CPU中物理地址合成公式为：**物理地址=段地址*16+偏移地址**。
    - 内存并没有分段，段的划分来自CPU。
    - 将若干的地址连续的内存单元看做一个段，用段地址*16定位段的基址，用偏移地址定位段中的内存单元。

- CPU如何区分出内存中存放的是数据还是指令呢？
  - 数据和指定在内存中的存储形式是一致的，都是二进制信息
  - CPU将**CS:IP**指向的内存单元中的内容看做指令，将DS:[addr]指向的内存单元中的内容看做数据
- **SS:SP**用于指向栈顶元素，需要注意栈顶越界的问题。
- 内存单元的描述
  - 完整的描述一个内存单元需要两种信息：内存单元的地址和内存单元的长度。
  - `mov ax, [0]`：将ds:[0]的内容送入ax中，内存单元的长度为2字节（即一个字单元，因为是送入到ax）中。
  - `move al, [0]`：将ds:[0]的内容送入al中，内存单元的长度为1字节（即一个字节单元）。
- 转移指令：修改CS或IP的指令成为转移指令。只修改IP时称为段内转移（近转移），同时修改CS和IP的指令称为段间转移（远转移）。
  - 无条件转移指令，如`jmp`。
    - `jmp`不需要转移的目的地址，而是通过相对位移来跳转。
  - 条件转移指令。
  - 循环指令，如`loop`。
  - 过程
    - call & ret
  - 中断
    - CPU在执行完当前正在执行的指令后，检测到从外部或者内部产生的特殊信息（中断信息），便转去处理这个特殊信息
    - 内中断
      - 除法溢出
      - 单步执行（debug）
      - 执行into
      - 执行int指令
    - 外中断
      - 可屏蔽中断
      - 不可屏蔽中断

- 中断处理程序入口通过中断向量表（中断类型码）来索引到（存储中断程序地址），对于8086而言，中断向量表存储在内存地址0处。中断向量表中，一个表项占两个字节，高字节存放段地址，低字节存放偏移地址
- 中断过程
  - 从中断信息中获取中断类型码
  - 标志寄存器入栈
  - 设置标志位TF和IF为0
  - CS内容入栈
  - IP内容入栈
  - 根据中断类型码定位到中断程序的入口
