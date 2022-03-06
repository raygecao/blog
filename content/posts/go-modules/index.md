---
weight: 20
title: "Go modules要点梳理"
subtitle: ""
date: 2022-03-06T13:20:22+08:00
lastmod: 2022-03-06T13:20:22+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "mvs.png"

tags: ["go"]
categories: ["整理与总结"]

toc:
  auto: false
lightgallery: true
license: ""
---
Modules are how Go manages dependencies.
<!--more-->
## 术语介绍

- 包（**package**）：相同目录下，一起编译的源文件集合，每个包会归属于某个模块
- 模块（**module**）：一组统一release并publish的包的集合
- 主模块（**main module**）：go命令被调用时所在的模块，主模块在当前目录下或者在其父目录下存在`go.mod`文件
- 依赖（**dependency**）
  - 直接依赖（**direct dependency**）：主模块所显式import的包或者模块
  - 间接依赖（**indirect dependency**）：主模块间接引入的包或者模块，未被主模块显式import。在`go.mod`中需要加上`//indirect` suffix
- 模块图（**module graph**）：模块依赖构成的以主模块为root的有向图，每条边对应`go.mod`中一条require命令
- 主版本后缀（**major version suffix**）：一个major version子目录用于定义一个模块的新版本，一般当模块有较大的breaking change时引入
- 模块代理（**module proxy**）：实现GOPROXY标准的webserver，go命令从proxy下载模块相关信息
- 模块缓存（**module cache**）：一个存放下载模块的本地目录，直接serve此目录可以构建一个module proxy
- 最小版本选择算法（**mvs**）：构建时用以决定所有模块版本的算法，基准为**选择满足约束的最小版本**
- 标准版本（**canonical version**）：由字母`v`加`semver`组成，不带meta的（+incompatible除外）版本，只有标准版本可以用于mvs

## Model Path

- 模块路径会在`go.mod`中声明，用于描述如何定位到对应的模块
- 模块路径是其包含的所有包路径的前缀
- 模块可能包含如下三部分：
  - repo root path，即模块所在的vcs仓库
  - repo root path下的一个子目录，当repo root path无法定位到模块时，可能引入了一个子目录后缀
  - 一个major version suffix，如`golang.org/x/somemodule/v2`

## Model Version

- 遵循[semver2.0](https://semver.org/)规范
  - 打破后向兼容的change需要递增major version，并将minor/patch设置为0
  - 未打破后向兼容的change需要递增minor version，并将patch设置为0
  - Bugfix或者小优化可以递增patch version
  - Pre-release会在patch后，使用`-`连接的部分信息。pre-release会比对应release版本号小。当存在release时，latest version不会匹配pre-release，即使此pre-release更晚发布
  - Metadata suffix指patch后，用`+`连接的部分，不用做version排序，只起到标识作用
- `+incompatible`是用于表明兼容性的特殊metadata，标识所引的模块**在迁移到module之前就已经release了`major>1`的版本**，这个metadata表明会从对应版本tag的**非major version suffix目录**中去找模块
  - 模块一定要在repo root directory中，即**module path与repo root directory一致**
  - 不应有`go.mod`文件
- 伪版本(pseudo-versions)
  - 是一种特殊的pre-release形式
  - 伪版本一般用于如下场景
    - 无semver tag的场景，此场景版本base一般为`v0.0.0`
    - 用于标识发布release tag之前的测试版本
    - 指定branch/commit map对应的revision上无semver tag，此场景下版本base为最近的semver tag
  - 伪版本常见形式为`vX.Y.Z-yyyymmddhhmmss-abcdefabcdef`，一般编码如下信息：
    - 下一个待发布的版本`vX.Y.Z`
    - 生成此伪版本的时间戳
    - 模块仓库的revision（如git系统下的commit号）

## MVS

- 操作实体是一个模块图

  - 节点是模块名
  - 边描述对模块**所依赖的最小required version**，由`go.mod`中require命令指定，会被replace/exclude等命令影响
  - mvs从主模块出发遍历图，跟踪每个模块**被依赖的最大required version**，这些模块及相应的版本构成了build list
  {{< image src="mvs.png" caption="MVS版本选择算法" width=800 height=400 >}}

## Workspace

- 功能：在运行mvs时，**将磁盘上的模块添加到主模块中**
- 由`go.work`声明，可以通过`-workfile=off`禁用；也可通过`-workfile=*.work`指定路径，否则会沿着当前目录及其祖先目录寻找`go.work`，核心命令如下：
  - `use`：声明需要加入主模块的本地模块路径
  - `replace`：进行指定模块的替换，**适用于workspace内的所有模块**

## Module Proxy

实现了如下GET api的http server：

| Path                             | Required | Description                                                  |
| :------------------------------- | -------- | ------------------------------------------------------------ |
| `$base/$module/@v/list`          | T        | 返回`$module`的所有已知版本（不包含pseudo-versions）         |
| `$base/$module/@v/$version.info` | T        | 返回包含canonical version相关信息，用于定位模块。`$version`与返回的canonical version并不需要一致，但大多数情况下会保持一致；Time字段可选 |
| `$base/$module/@v/$version.mod`  | T        | 返回模块指定版本的`go.mod`文件，如果没有此文件，则生成一个只有`module`命令的`go.mod` |
| `$base/$module/@v/$version.zip`  | T        | 返回模块指定版本的源码压缩包                                 |
| `$base/$module/@latest`          | F        | 返回latest version的info信息                                 |

- 为了解决uri大小写不敏感的问题，使用`!m`替换`M`
- 环境变量`GOPROXY`用于声明一组proxy url用于定位模块，其默认值为`https://proxy.golang.org,direct`，其中`direct`表明直接从源码仓库VCS去找模块

### 定位包所在的模块

- 根据`GOPROXY`遍历proxy list，对package path进行最长前缀匹配（匹配module path+subdirectory），如果匹配到了module并且包含对应的package，则记录对应的模块
- 如果匹配过程中报了权限等**非404/410相关的错误**则抛出错误（GOPROXY以`|`分隔proxy list则会继续遍历）
- 如果此proxy所有的可能模块路径request均返回404/410，继续遍历proxy list并执行第一步
- 如果proxy list遍历完也未找到对应的模块，直接报错
- 例：`GOPROXY=https://corp.example.com,https://proxy.golang.org`，搜索`golang.org/x/net/html`所在模块：
  - To `https://corp.example.com/`
    - Request for latest version of `golang.org/x/net/html`
    - Request for latest version of `golang.org/x/net`
    - Request for latest version of `golang.org/x`
    - Request for latest version of `golang.org`
  - To `https://proxy.golang.org/`
    - Request for latest version of `golang.org/x/net/html`
    - Request for latest version of `golang.org/x/net`
    - Request for latest version of `golang.org/x`
    - Request for latest version of `golang.org`

## Module Cache

- 存放已下载的模块文件，和build cache没有关系
- cache在本机所有project中共享，接入是并发安全的
- cache的模块是只读的
- 默认path为`$GOPATH/pkg/mod`，可以通过` go clean -modcache`将cache purge掉
- `cache/download`path下的布局是符合Module Proxy标准的，因此可以直接serve `cache/download`作为module proxy

{{< admonition note >}}

module cache并不十分严格符合GOPROXY协议，比如模块cache了pseudo-version的话，`@v/list`会返回pseudo-version。

{{< /admonition >}}

### go get 底层API调用流程

Serve本地module cache作为proxy验证go get的api调用。

在域名为 `web.raygecao.cn`上下载`6.824-test`模块，并serve `cache/download`目录作为proxy：

```shell
# web.raygecao.cn server
$ go get github.com/raygecao/6.824-test
go: finding github.com/raygecao/6.824-test v0.0.1
go: downloading github.com/raygecao/6.824-test v0.0.1
go: extracting github.com/raygecao/6.824-test v0.0.1

$ cd $GOPATH/pkg/mod/cache/download/ && sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

在另一台机器上将`web.raygecao.cn`设置为GOPROXY，并获取此模块：

```shell
# client
$ go mod init test
$ GOPROXY=http://web.raygecao.cn  GOSUMDB=off go get github.com/raygecao/6.824-test
```

因为客户端没有此模块的cache，因此在server端观察到的api调用流程为：

{{< image src="api-without-cache.png" caption="模块未被cache时，server端观测的api调用" width=800 height=400 >}}

此时模块已下载并cache住，再次go get此模块时，api调用流程为：

{{< image src="api-with-cache.png" caption="模块被cache后，server端观测的api调用" width=800 height=400 >}}

**总结完整的api调用流程如下：**

- module path拆分，并发调用`@v/list`api以及最长前缀匹配去定位模块，调用`@v/list` api获取所有release/pre-release，选择latest version
- 【cached】调用`@v/info`api获取canonical version等信息
- 【cached】调用`mod`api获取依赖从而构建build list
- 【cached】调用`zip`api获取模块源码，从而加载相应的package

{{< admonition note "@latest api用途" >}}

由于`@v/list`只会获取release/pre-release，当模块repo里没有release/pre-release tag时，`@v/list`返回结果为空，此时会尝试调用`@latest`endpoint搜索latest pseudo-version，依然找不见则会自动生成。

{{< /admonition >}}

## Module Authentication

- 验证的主体**包括zip文件和mod文件**，可以使得不可信的proxy提供的包变得可信，此外cache提供的模块也需要经过验证
- 对于zip文件，计算hash是顺序无关的，并且不受其他metadata、alignment等影响
- 验证时会优先从`go.sum`中找，如果`go.sum`不存在，会向checksum database query，验证通过后记录到`go.sum`中
- checksum校验失败的一个常见的原因是**人为更改repo tag**

## 相关环境变量

| 环境变量       | 默认值                            | 类型                        | 作用                                                         |
| -------------- | --------------------------------- | --------------------------- | ------------------------------------------------------------ |
| **GOMODCACHE** | `$GOPATH/pkg/mod`                 | **filepath**                | 指定下载的模块及相关文件存放的目录                           |
| **GOINSECURE** | -                                 | 逗号分隔的**glob patterns** | 用于匹配模块前缀，当直接从VCS中fetch匹配的模块时，可以允许其使用insecure manner（`https`=>`http`, `git+ssh://` => `git://`) |
| **GONOPROXY**  | `$GOPRIVATE`                      | 逗号分隔的**glob patterns** | 用于匹配模块前缀，匹配的模块跳过query proxy，直接从VCS中获取 |
| **GONOSUMDB**  | `$GOPRIVATE`                      | 逗号分隔的**glob patterns** | 用于匹配模块前缀，匹配的模块跳过校验流程                     |
| **GOPRIVATE**  | -                                 | 逗号分隔的**glob patterns** | `GONOPROXY`与`GONOSUMDB`的默认值                             |
| **GOPROXY**    | `https://proxy.golang.org,direct` | **url list**                | 声明proxy列表，当前一个proxy返回`404/410`状态码时，follow next（针对逗号分隔的场景） |
| **GOSUMDB**    | `sum.golang.org`                  | **url**                     | 指定checksum database，当`go.sum`不存在且未跳过校验流程时，从此url中获取hash用于校验 |

## Tips

- 主模块要求有`go.mod`文件，但是没有`go.mod`文件的模块可以用作依赖，在寻找此模块时，如果模块路径与repo root path一致时，go command会生成一个只有`module`命令的`go.mod`，以保证依赖方每次构建的确定性

- Go1.17相对于之前的版本在`go.mod`中显式地列出了所有主模块间接导入的包，而之前的版本对于间接依赖，只有在默认mvs选择的版本与被依赖的版本不一致时才会显式列出。这些额外的间接依赖的信息用于**模块图剪枝**及**模块延迟加载**

- retract用法（始于go 1.16）

  - 声明本模块的一些版本不应被依赖，一般用于误发版本/版本存在fatal bug时

  - 被撤销的版本本身会存在，避免破坏已依赖此版本的模块构建

  - 用法示例：当前最新版本为`v0.9.1`，又发布了`v1.0.0`且有fatal bug，则需要发布`v1.0.1`并进行如下声明：

    ```
    retract (
        v1.0.0 // Published accidentally.
        v1.0.1 // Contains retractions only.
    )
    ```

- 一般来讲module path跟repo root path应是一致的（不考虑major version suffix），但有些情况下模块会定义在repo root path下的子目录中，一般用于**monorepo中多个组件需要独立发版**的case，这其中每个组件都应有个`go.mod`文件

## References

- https://go.dev/ref/mod
- https://go.dev/blog/using-go-modules
- https://github.com/golang/go/blob/go1.17.7/src/cmd/go/internal/modfetch/proxy.go
- https://github.com/golang/go/issues/32715
- https://github.com/golang/go/issues/51391
