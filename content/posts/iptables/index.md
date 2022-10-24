---
weight: 20
title: "Iptables概要整理"
subtitle: ""
date: 2021-12-08T20:44:15+08:00
lastmod: 2021-12-08T20:44:15+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["网络"]
categories: ["整理与总结"]


toc:
  auto: false
lightgallery: true
license: ""
---
**Iptables** is the userspace command line program used to configure the Linux 2.4.x and later packet filtering ruleset. It is targeted towards system administrators.
<!--more-->

## 简介

Netfilter/Iptables是unix/linux自带的防火墙工具，netfilter是内核中**数据包处理模块**，定义了一组hook点，埋在网络协议栈中以对不同阶段的包进行处理，而Iptables是**操纵netfilter的命令行工具**。netfilter工作在**二、三、四层**，其匹配规则涵盖网卡，IP，端口等。

- netfilter的作用
  - NAT（网络地址转换）
  - 数据包内容修改
  - 数据包过滤

### Iptables 四表五链

- **Iptables四张表**

  - **filter表**：负责过滤数据包；
  - **nat表**：网络地址转换，如SNAT、DNAT；
  - **mangle表**：修改报文，解包->修改->封包；
  - **raw表**：关闭nat及数据包链接追踪机制；

- **Iptables五个默认链**

  - **PREROUTING**：刚进入网络层的数据包会先经由PREROUTING链，并根据路由判断目标地址是否为本机。由于此链为接收包的第一站，因此**DNAT通常在此链上配置**；
  - **INPUT**：经由PREROUTING链的数据包发现目标地址为本机，会经由INPUT链进入本机，**入站过滤规则通常在此链配置**；
  - **FORWARD**：经由PREROUTING链的数据报发现目标地址不是本机，会经由FORWARD链进行转发，通常**网络防火墙会在此链配置**；
  - **OUTPUT**：从本机产生的数据包向外发送时会先经过OUTPUT链，通常**出站过滤规则在此链配置**。
  - **POSTROUTING**：经由OUTPUT/FORWARD链中流出的数据包最终经路由后会到达POSTROUTING链，以从对应的接口流出，POSTROUTING为数据包流出的最后一站，**SNAT通常在此链上配置**；

  每条链上会有多个规则，按顺序匹配，符合规则的触发action，并停止后续的规则匹配。 
  
  {{< image src="iptables.png" caption="iptables的默认链及拥有的功能表，引自[从零开始认识 iptables](https://morven.life/posts/the_knowledge_of_iptables/)" width=500 height=300 >}}

  - 数据包由外部发往本主机进程流向为：`PREROUTING -> INPUT`
  - 数据包由本主机发往外部进程流向为：`OUTPUT -> POSTROUTING`
  - 数据包由本主机进行转发的流向为：`PREROUTING -> FORWARD -> POSTROUTING`

- **表与链的区别**：表主要是按功能进行分类；而链是协议栈中某处提供的钩子函数。

- **默认策略**：每条默认链都会有一个默认策略，**在没有规则被匹配时会执行默认策略中的动作**，而与之相关的是防火墙的两种通行策略：

  - **黑名单机制**：只有匹配到规则的数据包才可能被拒绝，没有匹配到的数据包都会被接受。**将默认策略设置为ACCEPT即可实现黑名单机制**；
  - **白名单机制**：只有匹配到规则的数据包才可能被接收，没有匹配到的数据包都会被拒绝。**将默认策略设置为DROP即可实现白名单机制**；

  {{< admonition note >}}

  黑白名单机制并非只能靠调整默认策略实现，设置默认策略只是实现此机制的一种方案，也可以通过其他方案来实现黑白名单机制，比如**将默认策略设置ACCEPT，在链的尾部加一个DROP ALL的规则可以实现白名单机制**。通常**不建议将默认策略更改为DROP/REJECT**，因为在此默认策略下，误flush了链会导致所有请求被拒绝。

  {{< /admonition >}}

- **同一链上各表执行优先级**：`raw > mangle > nat > filter`



### 匹配条件

- 常见匹配条件

| 类型     | iptables输出 | 备注                                            |
| -------- | ---------------- | ----------------------------------------------- |
| 源IP     | source           | IP报头中记录，可以使单个IP地址或网段，`-s`指定  |
| 目的IP   | destination      | IP报头中记录，可以使单个IP地址或网段， `-d`指定 |
| 流入网卡 | in               | `-i`指定                                        |
| 流出网卡 | out              | `-o`指定                                        |
| 协议类型 | prot             | `-p`指定，支持tcp/udp/icmp等                    |

- 扩展匹配条件

| 类型     | 表示        | 备注                                                         |
| -------- | ----------- | ------------------------------------------------------------ |
| 源端口   | tcp spt:22  | 导入协议模块，同时指定`--sport`指定目的端口                  |
| 目的端口 | tcp  dpt:22 | 导入协议模块，同时指定`--dport` 指定目的端口                 |
| 字符串   | STRING      | 导入string模块，同时指定`--string`指定匹配文本内容           |
| 连接数   | connlimit   | 需导入connlimit模块，并设置连接数规则                        |
| 报文速率 | limit       | 需导入limit模块，并设置速率限制规则                          |
| 状态     | state       | 需导入state模块，通过`--state`对状态进行设置，通常用来判断报文是对方主动发送的还是对方回复的响应报文 |

使用`!`可以对匹配条件进行取反，扩展匹配条件通常需要通过`-m`导入特定的模块才可以使用。

### 常用的处理动作

| 动作       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| ACCEPT     | 接受此数据包                                                 |
| DROP       | 直接丢弃此数据包                                             |
| REJECT     | 拒绝数据包，并返回相应的通知                                 |
| SNAT       | 源端网络地址转换，即将数据包的源IP改写为指定IP               |
| MASQUERADE | 动态的SNAT，无需手动指定源IP，iptables会将数据包的源IP改写成out网卡的IP，适用于动态生成IP的场景 |
| DNAT       | 目的端网络地址转换，即将数据包的目的地址改写为指定IP         |
| REDIRECT   | 本机端口映射，即将数据包导向另一个端口                       |
| LOG        | 将数据包记录在内核日志中，然后继续后续规则匹配               |
| RETURN     | 结束在当前链中的规则匹配，返回到父链继续匹配                   |

### CheatSheet

| 选项      | 作用                   | 备注                                                  |
| --------- | ---------------------- | ----------------------------------------------------- |
| `-L`      | 获取规则列表           | 可接链名，不接默认为所有链，一般使用`-nvL --line`列举 |
| `-I`/`-A` | 在特定行/尾行插入      | `-I`可以指定行号，默认为首行                          |
| `-t`      | 指定表名               | 默认为filter表                                        |
| `-D`      | 删除某条规则           | 可以指定行号，也可以指定匹配条件+动作                 |
| `-R`      | 更新规则               | **须指定匹配条件和动作**，建议使用**删除+添加**的形式 |
| `-F`      | 清空某条链全部规则     | 慎用                                                  |
| `-P`      | 修改某条链上的默认规则 | 初始为ACCEPT，**不建议设置为DROP/REJECT**           |
| `-N`      | 创建一条自定义链       | 规则模块化，便于管理                                  |

### 增删改查基本指令示例

```shell
$ iptables -t filter -nvL INPUT --line # 查询filter表，INPUT链的所有规则并展示行号
$ iptables -t filter -I INPUT 2 -s 10.1.0.1 -j DROP # 在filter表，INPUT链第二行添加拒绝源IP为10.1.0.1的DROP规则
$ iptables -t filter -D INPUT 2 # 在filter表，INPUT链删除第二条规则
$ iptables -D INPUT 2 && iptable -I INPUT 2 -s 10.1.0.2 -j REJECT # 将filter表，INPUT链第二条规则修改为拒绝10.122.0.2
$ iptables -t filter -P FORWARD DROP # 将filter表的FORWARD链默认动作设置为DROP(默认禁止转发)
```

### 自定义链

上述描述的五条链是Iptables的默认链，除此之外，iptables还支持自定义链，自定义链与默认链的操作方式相同，但不是netfilter标准hook点，因此**自定义链需要附加在某个默认链上**。默认链在匹配到自定义链时会将其展开进行规则匹配，类似于内联函数，我们可以类比自定义函数来看一下自定义链的优势。

- 函数一般需要满足单一性原则，自定义链也是**将满足同一功能/归属统一业务的规则集中在一条链上管理**，这样可以有效避免默认链上规则无限制的增长，便于管理与维护。
- 函数是可复用的，自定义链也是**可以复用**的，一些通用的规则可以放在一条自定义链上被多个默认链/其他自定义链引用，对复用链的更新会传播到引用链中，对同一条规则的修改无需手动操作多条链。

## Docker iptables分析

为了将docker中iptables的作用展示的更透彻，示例环境启动了两个容器，详情如下：

```bash
# docker ps
CONTAINER ID   IMAGE                 COMMAND         CREATED       STATUS       PORTS                     NAMES
f36770eceff5   ibmcom/guestbook:v2   "./guestbook"   3 weeks ago   Up 3 weeks   0.0.0.0:19002->3000/tcp   jolly_noether
d8e30286aa69   ibmcom/guestbook:v1   "./guestbook"   3 weeks ago   Up 3 weeks   0.0.0.0:19001->3000/tcp   eager_dubinsky
```

均在默认的bridge网络下，两个容器的3000端口分别publish到19002与19001端口。

### filter表

Iptables的INPUT链，OUTPUT链均为空，FORWARD设置了默认策略是DROP的白名单模式，这里着重看一下**FORWARD**链：

{{< image src="forward-filter.png" caption="filter表中forward链" width=800 height=200 >}}

- Rule1引用了一个自定义链`DOCKER-USER`，此链只有一个RETURN动作，如果需要在此docker环境下手动添加iptables rules可以选择在此处添加。

- Rule2引用了`DOCKER-ISOLATION-STAGE-1`链，相继引用了`DOCKER-ISOLATION-STAGE-2`链，这两条链也都没有实质性规则，所有的转发数据包会继续向下匹配。
- Rule3表示接受从docker0流出的所有响应报文。
- Rule4表示从docker0流出的数据包需要走DOCKER链，DOCKER链我们稍后分析。
- Rule5表示接受从docker0流入，但不是从docker0流出的数据包，**容器访问宿主机/外网等非容器网络时匹配此规则**。
- Rule6表示接受从docker0流入，同时从docker0流出的数据报，在默认bridge网络上的**容器间通信匹配此规则**。

再来看一下**DOCKER**链，容器相关的规则都放在DOCKER链上，便于管理及查询。

{{< image src="docker-filter.png" caption="filter表中自定义DOCKER链" width=800 height=200 >}}

- Rule1表示接受不是从docker0流入，但从docker0流出，并且访问的目的IP为**172.17.0.2**， 访问目的port为**3000**的tcp报文。
- Rule2表示接受不是从docker0流入，但从docker0流出，并且访问的目的IP为**172.17.0.3**， 访问目的port为**3000**的tcp报文。

上述规则都没有匹配的报文，那是因为无论我们从容器内还是宿主机访问容器，都是从docker0流入、从docker0流出。如果将某个一条路由的节点添加一条路由规则，将172.17.0.0/24的destination gateway设置为本机，这样对`172.17.0.3:3000`的访问会经物理网卡流入，经docker0流出到容器内，会匹配上此链上的规则。说白了，其意图是**只要你能将请求路由到本机，我就允许你访问默认bridge网络中容器提供的服务**。

### nat表

下面来分析一下nat表，OUTPUT链和PREROUTING链会引用DOCKER自定义链，其他链上规则基本为空或者都不会匹配到。我们主要关注一下POSTROUTING链和DOKCER链，先看一下**POSTROUTING**链：

{{< image src="postrouting-nat.png" caption="nat表中PREROUTING链" width=800 height=200 >}}

- Rule1表示不是从docker0流出的，且源网段为`172.17.0.0/16`的数据包会被进行动态SNAT，将数据包的源IP改为流出网卡的地址，若从容器内访问外网，源IP会被更新为宿主机物理网卡的IP地址。
- Rule2，Rule3乍一看像是自己访问自己的3000端口时会采用MASQ。但从容器内路由规则可以看到，容器内访问容器网络是无需路由到docker0，而直接在二层网络上转发。而容器内通过容器IP访问自身服务是无需经由docker0的，因此第2、3条的作用不明确。

再看一下**DOCKER**链：

{{< image src="docker-nat.png" caption="nat表中自定义DOCKER链" width=800 height=200 >}}

- Rule1表示从docker0流入的数据包直接跳过此链，因为容器网络间的访问是不需要NAT的。
- Rule2, Rule3分别表示两个容器publish了3000端口到19001端口，3000端口到19002端口。那些不是从docker0流入的数据包想要访问容器内的服务时，访问对应网卡上的**19001**端口的tcp请求就转发到**172.17.0.2:3000**；而访问**19002**端口的tcp请求就转发到**172.17.0.3:3000**。

## Tips

- 由于iptables是按顺序匹配的，因此应该**将更容易匹配的规则放在前面**，以提升防火墙的匹配性能。
- 当存在多个匹配条件时，各条件之间为与的关系，即**所有匹配条件均满足时才会执行对应的action**。
- 配置白名单时，一般不会将默认链策略设置为DROP，这是因为如果误刷了该默认链（`-F`）将导致所有数据包都会被拒绝，会导致我们无法连接到服务器。**一般白名单机制会将默认链策略设置为ACCEPT，并在链的最后设置DROP ALL的规则**。
- 为提升iptables线性匹配算法，一些ACL优化算法被提出，[eBPF 技术实践：高性能 ACL](https://www.infoq.cn/article/Tc5Bugo5vBAkyaRb5CCU) 采用根据匹配规则倒排，并利用bitmap来快速匹配的算法，以达到近O(1)的匹配性能。

## 参考文献

- [朱双印的博客iptables部分](https://www.zsythink.net/archives/category/%e8%bf%90%e7%bb%b4%e7%9b%b8%e5%85%b3/iptables)
- [从零开始认识 iptables](https://morven.life/posts/the_knowledge_of_iptables/)
- [eBPF 技术实践：高性能 ACL](https://www.infoq.cn/article/Tc5Bugo5vBAkyaRb5CCU)
