---
weight: 20
title: "Terraform实践"
subtitle: ""
date: 2023-03-05T10:23:12+08:00
lastmod: 2023-03-05T10:23:12+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["IaC"]
categories: ["探索与实战"]

toc:
  auto: false
lightgallery: true
license: ""
---
本文简述Terraform HCL语言；提供基于Docker与AWS平台供应基础设施的demo；并对多环境管理进行探索。

<!--more-->

## 模板引擎

### 文件结构

Terraform识别以下几类特殊文件：

- `*.tf` ：资源声明文件，用于声明所要创建的资源。
- `*_override.tf`：资源后处理文件，用于覆盖已存在资源的某些部分。
- `.terraform.lock.hcl`：锁定provider的版本。
- `terraform.tfvars`：给定已声明变量一组特定值。

### HCL语法

[HCL](https://developer.hashicorp.com/terraform/language)全称为**HashiCorp Configure Language**，是由HashiCorp专门为Terraform设计的声明式配置语言，用来描述如何管理一组的基础设施。

HCL通用语法块为：

```
<块类型> "<资源类型>" "<资源名>" {
  # Block body
  <属性名> = <属性值> # Argument
}
```

其中

- 块类型包含以下几种常见类型：

  | 块类型    | 作用           |
    |--------------| ---------------------- |
  | terraform | 声明provider版本及来源 |
  | provider  | 配置provider   |
  | resource  | 定义并配置资源对象    |
  | data      | 获取外部资源       |
  | variable  | 定义模块输入变量     |
  | output    | 定义模块输出变量     |
  | locals    | 定义本地变量       |
  | module    | 定义模块引用       |

- 资源类型及Arguments由provider定义。

- 资源名由用户自定义，用来在引用模块内资源，因而避免相同资源同名的问题。并非所有块类型都需要额外的资源名（如variable、output、provider、locals等）

以创建一个redis docker container为例体验一下HCL：

```
# 声明docker provider
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

# 配置provider，必需，即使没有arguments
provider "docker" {}

# 创建redis docker image resource
resource "docker_image" "redis" {
  name         = "redis:5.0.5-alpine"
  keep_locally = true
}

# 创建redis docker container resource，引用上一步创建的docker image id
resource "docker_container" "redis" {
  name = "storage"
  image = docker_image.redis.image_id
  command = ["--requirepass", "dea1452fe9133ea28e60b25f70fa93c43bcfeca9648d0cb470a473f563c91af6"]
}
```



### 隐式依赖

HCL中通过资源引用来声明隐式依赖，即`R1`的某些输入参数通过引用`R2`资源的输出结果建立起了`R1`对`R2`的隐式依赖。Terraform在进行资源创建时，会先创建`R2`再去创建`R1`，删除顺序与创建顺序相反。

上例中`docker_container.redis.image`引用了`docker_image.redis.image_id`，即`docker_container.redis`依赖于`docker_image.redis`。

```shell
$ terraform apply --auto-approve

# ...... some resource plan output ......
docker_image.redis: Creating...
docker_image.redis: Creation complete after 0s [id=sha256:ed7d2ff5a6232b43bdc89a2220ed989f532c3794422aa2a86823b8bc62e71447redis:5.0.5-alpine]
docker_container.redis: Creating...
docker_container.redis: Creation complete after 1s [id=583c145b78d31bc28d8c993087c9bbd077be2dd6c60f51da6f78299880958ae7]

$ terraform destroy --auto-approve

# ...... some resource plan output ......
docker_container.redis: Destroying... [id=583c145b78d31bc28d8c993087c9bbd077be2dd6c60f51da6f78299880958ae7]
docker_container.redis: Destruction complete after 1s
docker_image.redis: Destroying... [id=sha256:ed7d2ff5a6232b43bdc89a2220ed989f532c3794422aa2a86823b8bc62e71447redis:5.0.5-alpine]
docker_image.redis: Destruction complete after 0s
```

### 模块

为了增强功能块复用，Terraform提供了强大的模块化能力。

模块通常包含一组一起使用的资源，通常在一个目录下定义多个`tf`文件以实现一个模块。

模块通常分为两类：

- 根模块：在工作目录（执行terraform CLI时所在的目录）下的模块。
- 子模块：被其他模块引用的模块，子模块可以是自定义的本地模块，也可以使用registry中第三方提供的模块。模块引用可以嵌套，但嵌套深度不应过深。

模块之间的交互主要有两种方式：

- 父模块通过配置子模块属性向子模块传递input variables。
- 子模块通过output variables向父模块返回相关的配置值供其引用。

## Terraform in Docker

Terraform可以构建、修改及移除Docker基础设施。[Docker provider](https://registry.terraform.io/providers/kreuzwerker/docker/3.0.1)提供了network、volume、image、container等资源，以提供对Docker操作的完整支持。

[Terraform Docker Tutorial](https://developer.hashicorp.com/terraform/tutorials/docker-get-started)详细描述了如何部署一个nginx docker container，配置与上例类似。

我们以部署一个带存储后端（redis）的留言板（guestbook）为例，探索使用Terraform部署两个具有依赖关系的docker container。

该场景资源依赖关系可以描述为下图：

{{< image src="dep.png" caption="docker资源的依赖关系图" width=800 height=400 >}}

具体实现可参考[示例代码](https://github.com/raygecao/terraform-demo/tree/19dad1f72369e72c4bdb5e6ff81107a492dca542/docker)，工作方式与docker compose极为相似。

然而Docker资源属于上层组件，Terraform并不具备管理优势，其核心价值在于公有云基础设施供应。

## Terraform in AWS

通常，云提供商为了满足多样的商业需求，会提供与计算、存储、网络相关的多种功能服务。这些厂商提供了易用的UI界面及api供用户管理相关的资源，但存在以下问题：

- 服务种类众多，每个服务又包含数十种配置，给用户带来大量的学习成本。
- 流程繁杂，用户需要按顺序地配置、启动服务，并进行资源关联，易错且低效。

单从创建一个强隔离性的nginx ec2 instance而言，用户不得不手动创建vpc、subnet、internet gateway、route table等网络服务以及ec2、security group、key-pair等计算资源。构建过程需要对这些资源手动创建并正确配置，清理过程同样需要手动移除各个服务。如果考虑在云上维护多个环境，那管理成本无法估量。

Terraform将我们从繁杂的基础设施管理中解救出来：

- 声明式配置简化我们在界面上点点点的操作，并降低出错的概率。
- 资源按依赖顺序并发构建，大幅提升基础设施的部署速度。
- 模块化供应屏蔽了大量内部细节，比如`aws_vpc` module将vpc、subnet、internet gateway、route table等资源封装起来。

[示例代码](https://github.com/raygecao/terraform-demo/tree/b4aa280b4b8fe34fce418e0c3ca7acecee508641/aws/single)展示了构建一个guestbook ec2 instance的方法，并将相应的公网ip输出出来。

```shell
$ terraform apply --auto-approve

# ..... some resource plan output ......
aws_key_pair.ssh: Creating...
aws_vpc.my-vpc: Creating...
aws_key_pair.ssh: Creation complete after 1s [id=dev_instance_key]
aws_vpc.my-vpc: Creation complete after 3s [id=vpc-04f4f4619c774d27d]
aws_internet_gateway.my-igw: Creating...
aws_subnet.my-subnet: Creating...
aws_security_group.my-sg: Creating...
aws_internet_gateway.my-igw: Creation complete after 0s [id=igw-00d00c50080474895]
aws_subnet.my-subnet: Creation complete after 0s [id=subnet-060ff42389821a4e4]
aws_route_table.my-route-table: Creating...
aws_route_table.my-route-table: Creation complete after 2s [id=rtb-0c150344ed47e3c8b]
aws_route_table_association.a-rtb-subnet: Creating...
aws_route_table_association.a-rtb-subnet: Creation complete after 1s [id=rtbassoc-0c8c6b265e4333216]
aws_security_group.my-sg: Creation complete after 4s [id=sg-0ace2bfadd865994b]
aws_instance.my-server: Creating...
aws_instance.my-server: Still creating... [10s elapsed]
aws_instance.my-server: Still creating... [20s elapsed]
aws_instance.my-server: Still creating... [30s elapsed]
aws_instance.my-server: Creation complete after 35s [id=i-0fee61317795aab3b]

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

server-ip = "13.231.202.24"
```

通过`curl 13.231.202.24:3000`可以访问留言板应用。


## 多环境管理

一般的产品迭代过程分为需求研发、质量测试、功能上线等阶段。在不同的阶段，我们通常需要维护一套独立的环境以达到当前阶段的目的，比如通常我们需要维护dev、stage、prod三个环境。Terraform可以帮助我们简化多环境管理，越来越多的[多环境管理的最佳实践](https://blog.gruntwork.io/how-to-manage-multiple-environments-with-terraform-32c7bc5d692)被提了出来。

### 基于Terraform workspace管理

Terraform workspace用于描述存放在同一backend的state文件。某些backend支持存放多个state文件，因此支持通过多个workspace进行环境管理。

Terraform支持workspace子命令，用以列举、创建、切换workspace。一个workspace对应着一个环境，环境的切换对应于workspace的切换，初始仅存在一个**default** workspace。

为支持多环境使用不同的配置，Terraform提供了特殊的插值表达式`terraform.workspace`，Terraform在解析配置文件时，会将此标识替换成当前的workspace名字。下例展示了两种常见的用法：

```
resource "aws_instance" "my-server" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = "t2.micro"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.my-subnet.id
  vpc_security_group_ids      = [aws_security_group.my-sg.id]
  availability_zone			  = var.available_zone
  key_name = aws_key_pair.ssh.key_name
  tags = {
    # 使用workspace区分实例标识
    Name = "${terraform.workspace}-inst"   
  }
  # 使用workspace切换执行脚本
  user_data = file(terraform.workspace == "dev" ? "guest1.sh" : "guest2.sh") 
}

```

[示例代码](https://github.com/raygecao/terraform-demo/tree/b2da2aebfe625e2cd624945b761724a25aaa6321/aws/single)展示了使用多个workspace创建不同的ec2实例，每个workspace管理的基础设施相互独立。（此例workspace并未提交到VCS中，workspace管理于local backend）

使用workspace对多环境管理可以最大程度上复用配置代码，并且是Terraform官方提供的特性。

{{< admonition warning "workspace的局限性" >}}

workspace管理的多环境必须共用同一backend，并不是所有backend都可以用workspace进行管理。官方提供了[支持多workspace的backend](https://developer.hashicorp.com/terraform/language/state/workspaces#backends-supporting-multiple-workspaces)。此外workspace不适用于存在证书、接入控制隔离要求的环境管理。

{{< /admonition >}}

### 基于多分支管理

基于git branch对多环境管理是十分常见的策略，大量企业使用不同的release分支管理产品的版本。这最大程度上保证了版本的隔离性，避免不同版本的特性相互污染。但另一方面放弃了代码复用的能力，尤其是当维护的分支规模较大时，一个通用的bugfix会引起cherry-pick风暴。

为支持不同环境使用不同的配置，通常将不变的内容以模板的形式存放在Terraform配置文件中，将可变的配置声明在`variable.tf`中，不同分支通过维护一组特殊的配置值（记录在`terraform.tfvars`中）来控制环境的配置。

[示例仓库](https://github.com/raygecao/terraform-demo)创建了[dev分支](https://github.com/raygecao/terraform-demo/tree/dev/docker)与[prod分支](https://github.com/raygecao/terraform-demo/tree/prod/docker)分别管理dev环境与prod环境，二者的区别在于对服务监听端口及应用镜像的tag配置不同。

{{< admonition info "如何提升复用能力？" >}}

使用分支进行多环境管理本质上是对配置文件进行版本控制，可以通过配置与模板分开管理以提升代码复用能力。

{{< /admonition >}}

### 基于Terragrunt管理

Terragrunt是由[**Gruntwork.io**](https://gruntwork.io/)开发的Terraform代码管理工具，其核心宗旨是提供一层thin wrapper实现DRY(Don't Repeat Yourself)。其核心思想是将**不变的配置模板与可变的配置值分开管理**，以维护多环境的特异性配置，在设计上与[kustomize](https://kustomize.io/)异曲同工。

使用Terragrunt通常将配置模板和配置值放到不同的目录进行管理，如分别在modules、live目录下：

```
live
├── dev
│   └── docker
│       └── terragrunt.hcl
└── prod
    └── docker
        └── terragrunt.hcl
modules
└── docker
    ├── main.tf
    ├── output.tf
    └── variable.tf
```

在配置模板中，声明所需的输入变量：

```shell
$ cat modules/docker/variable.tf
variable "external_port" {
  description = "External port for expose guestbook service"
  type = number
}

variable "guestbook_image_tag" {
  description = "The tag for the guestbook image"
  type = string
}
```

在patch文件中，通过维护不同的参数文件来控制环境的配置：

```
terraform {
  source = "../../../modules/docker"
}

inputs = {
  external_port = 14000
  guestbook_image_tag = "v2"
}
```

在相应的配置目录下执行`terragrunt apply`即可实现对应环境的部署。参考完整[示例代码](https://github.com/raygecao/terraform-demo/tree/terragrunt)。



Terragrunt CLI本质是将分离的配置与模板组合在一起，再去执行对应的Terraform CLI，这也是Terragrunt会被定位为Terraform的**thin wrapper**的原因。

Terragrunt 结合了workspace与multi-branch管理的优势并解决了二者存在的问题。但是其作为一个外部工具引入了额外的学习成本，并且目前在Terraform Cloud中无法使用。

### 三种环境管理方式的比较

|                                         | Workspaces | Branches | Terragrunt |
| --------------------------------------- | ---------- | -------- | ---------- |
| Minimize code duplication               | ■■■■■      | □□□□□    | ■■■■□      |
| See and navigate environments           | □□□□□      | ■■■□□    | ■■■■■      |
| Different settings in each environment  | ■■■■■      | ■■■■□    | ■■■■■      |
| Different backends for each environment | □□□□□      | ■■■■□    | ■■■■■      |
| Easy to manage multiple backends        | □□□□□      | ■■■■□    | ■■■■■      |
| Different versions in each environment  | □□□□□      | ■■□□□    | ■■■■■      |
| Share data between modules              | ■■□□□      | ■■□□□    | ■■■■■      |
| Work with multiple modules concurrently | □□□□□      | □□□□□    | ■■■■■      |
| No extra tooling to learn or use        | ■■■■■      | ■■■■■    | □□□□□      |
| Works with Terraform Cloud              | ■■■■■      | ■■■■■    | ■□□□□      |

黑块越多表示越具优势。引自 https://blog.gruntwork.io/how-to-manage-multiple-environments-with-terraform-32c7bc5d692

### 多环境与Devops

在DevOps的实践领域中，一种高度自动化的方式是通过pipeline贯穿多个环境，从而实现**代码直接上线**的能力。

比如下面的[案例](https://medium.com/@kief/https-medium-com-kief-using-pipelines-to-manage-environments-with-infrastructure-as-code-b37285a1cbf5)中，一套代码控制多个环境使用，但通过一条pipeline将环境连接起来，从而实现自动化部署、测试、上线的能力。这依赖了Terraform基础设施构建的一致性。

{{< image src="stages.png" caption="pipeline横贯多个环境" width=800 height=400 >}}

具体的工作流如下：

- 向VCS提交代码。
- 通过CI将VCS中的模块拷贝发布到制品仓库中。
- CD检测到有新版本的制品发布，会应用到测试环境，并执行自动化测试。
- 测试通过后，自动地（或人工触发）应用到线上环境。

{{< image src="workflow.png" caption="infra自动化上线流程" width=800 height=400 >}}

## 使用Terraform Cloud实现自动化部署

Terraform Cloud是Hashicorp为支持Terraform工作流的平台，提供自动化管理基础设施的能力。其具备权限控制、密钥管理、托管状态文件等能力。

Terraform Cloud使用流程如下：

- 接入VCS provider，核心是向VCS中注入TC webhook。
- 创建workspace，可以指定具体的触发条件。
- 配置变量。包括`tf`文件中声明variable，Access Token相关的环境变量。

- 首次运行。[官方](https://developer.hashicorp.com/terraform/cloud-docs/vcs/troubleshooting)明确指出：**A workspace with no runs will not accept new runs from a VCS webhook. You must queue at least one run manually**。

之后， 便可以根据workspace创建的触发条件完成自动化部署。

{{< image src="runs.png" caption="运行状态列表" width=800 height=400 >}}


## 参考文档

- [Terraform config language tutorial](https://developer.hashicorp.com/terraform/tutorials/configuration-language)
- [Terraform docker tutorial](https://developer.hashicorp.com/terraform/tutorials/docker-get-started)
- [Terraform aws tutorial](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)
- [Terraform docker provider](https://registry.terraform.io/providers/kreuzwerker/docker/3.0.1)
- [Terraform aws provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [How to manage multiple environments with Terraform](https://blog.gruntwork.io/how-to-manage-multiple-environments-with-terraform-32c7bc5d692)
- [Terragrunt](https://terragrunt.gruntwork.io/)
- [kubernetes kustomize 初体验](https://zhuanlan.zhihu.com/p/38424955)
- [Using Pipelines to Manage Environments with Infrastructure as Code](https://medium.com/@kief/https-medium-com-kief-using-pipelines-to-manage-environments-with-infrastructure-as-code-b37285a1cbf5)
