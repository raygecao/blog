---
weight: 20
title: "Containers from scratch探究"
subtitle: ""
date: 2021-09-27T20:38:46+08:00
lastmod: 2021-09-27T20:38:46+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["container"]
categories: ["探索与实战"]


toc:
  auto: false
lightgallery: true
license: ""
---
通过隔离namespace与cgroup构建出一个小型容器，项目来源：[containers-from-scratch](https://github.com/lizrice/containers-from-scratch)
<!--more-->

## 概述

本项目以几十行代码搭建起了一个最简单的container，包含如下特点：

- Mount namespace隔离，通过`chroot`将container的文件系统隔离到宿主机的单个目录层次结构中。
- Pid namespace隔离，保证container与host的pid相互独立。
- 使用独立的cgroup限制容器内的资源使用。

## 代码分析

```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"syscall"
)

// go run main.go run <cmd> <args>
func main() {
	switch os.Args[1] {
	case "run":
		run()
	case "child":
		child()
	default:
		panic("help")
	}
}

func run() {
	fmt.Printf("Running %v \n", os.Args[2:])

	cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags:   syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
		Unshareflags: syscall.CLONE_NEWNS,
	}

	must(cmd.Run())
}

func child() {
	fmt.Printf("Running %v \n", os.Args[2:])

	cg()

	cmd := exec.Command(os.Args[2], os.Args[3:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	must(syscall.Sethostname([]byte("container")))
	must(syscall.Chroot("/home/liz/ubuntufs"))
	must(os.Chdir("/"))
	must(syscall.Mount("proc", "proc", "proc", 0, ""))
	must(syscall.Mount("thing", "mytemp", "tmpfs", 0, ""))

	must(cmd.Run())

	must(syscall.Unmount("proc", 0))
	must(syscall.Unmount("thing", 0))
}

func cg() {
	cgroups := "/sys/fs/cgroup/"
	pids := filepath.Join(cgroups, "pids")
	os.Mkdir(filepath.Join(pids, "liz"), 0755)
	must(ioutil.WriteFile(filepath.Join(pids, "liz/pids.max"), []byte("20"), 0700))
	// Removes the new cgroup in place after the container exits
	must(ioutil.WriteFile(filepath.Join(pids, "liz/notify_on_release"), []byte("1"), 0700))
	must(ioutil.WriteFile(filepath.Join(pids, "liz/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700))
}

func must(err error) {
	if err != nil {
		panic(err)
	}
}

```

- run函数中主要有两个工作：
  - **Fork出子进程并调用child函数**：`/proc/self/exe` 表明当前的程序，即fork出一份子进程执行当前程序的child命令。
  - **设置Clone隔离属性**：`Cloneflags`通过设置`syscall.CLONE_NEWUTS`,`syscall.CLONE_NEWPID`,`syscall.CLONE_NEWNS`分别隔离了uts, pid及mount namespace。`Unshareflags`设置了`syscall.CLONE_NEWNS`用以禁用**挂载传播**。
- child函数中主要做了四件事：
  - **为子进程设置cgroup**，设置当前cgroup总的进程数上限为20。
  - **更新子进程的hostname**，用以验证uts namespace隔离。
  - **更新子进程的root目录**，将container文件系统隔离到宿主机中的单个目录中。
  - **挂载proc及tmpfs**，用以验证pid隔离以及mount隔离。

## 视频讲解

{{< youtube MHv6cWjvQjM >}}

视频基本上将代码的核心模块全部手敲了一遍，核心内容解释的较为清晰，但存在如下问题：

- 视频中run方法中并未设置`cmd.SysProcAttr.Unshareflags=syscall.CLONE_NEWNS`，在实践中发现，未指定此flag会导致容器内的挂载会泄露到宿主机上，这与挂载传播相关，笔者系统上默认使用**shared**传播类型，猜测Liz的系统默认使用**private**传播类型。

- 视频中在Liz退出容器时会触发panic，报错内容为**No such file or directory**，探索后发现是在第59行卸载thing时出错，看一下syscall关系Mount和Unmount的api：

- ```go
  func Mount(source string, target string, fstype string, flags uintptr, data string) (err error)
  
  func Unmount(target string, flags int) (err error)
  ```

  尽管`umount`命令支持使用设备文件或者挂载点，但从报错信息及syscall api上看到，比较稳妥的方式是通过挂载点卸载，因此`Line:59`建议改成`must(syscall.Unmount("mytemp", 0))`。

## 实践探究

### 准备ubuntufs

实现所需的ubuntufs是container的rootfs，为其提供必要的指令。笔者采用从ubuntu container中将整个文件系统拷贝出来。

```shell
docker run -it --rm ubuntu:21.04

docker cp ${UBUNTU_IMAGE_ID}:/ ~/ubuntufs
```

准备完需要更新`Line:51`chroot的path为ubuntufs所在的路径。

### 运行容器并验证namespace隔离情况

```shell
# 容器内
root@container:/# ls -l /proc/self/ns/
total 0
lrwxrwxrwx 1 root root 0 Sep 24 05:57 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 mnt -> 'mnt:[4026536295]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 net -> 'net:[4026531969]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 pid -> 'pid:[4026536294]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 24 05:57 uts -> 'uts:[4026536293]'

# host
$ ll /proc/self/ns
total 0
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 net -> net:[4026531969]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 pid -> pid:[4026531836]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 user -> user:[4026531837]
lrwxrwxrwx 1 ubuntu ubuntu 0 Sep 24 13:57 uts -> uts:[4026531838]
```

由此可知，container内的mnt, pid, uts namespace与host的均不同。

### 验证Cgourp设置

Cgroups的设置也生效，在后台起若干个sleep进程后，新起的进程被cgroup限制了。

{{< image src="cgroup.png" caption="Cgroup限制最大进程数" width=600 height=200 >}}

Liz在测试时使用了fork bomb `:(){ :|:& };:`，这种形式本质上是shell实现的一个自身指数递归调用，其简化形式为：

```shell
bomb() { 
 bomb | bomb &
}; bomb
```

- `:()`定义了一个名字叫`:`的函数。
- `{}`声明了函数体。
- `:|:`表示函数递归调用并pipe到自身，从而实现进程数指数增长。
- `&`表示进程在后台运行。
- `;`表示函数调用结束。
- `:`运行此函数，触发fork bomb。

## 功能扩展

### 支持bind mount

**Bind mount**将宿主目录映射到容器内，实现上比较简单，即在`chroot jail`前进行bind即可。示例代码如下：

```go
        // bind mount
        testBindPath := filepath.Join(rootPath, "test")
        os.Mkdir(testBindPath, 0755)
        must(syscall.Mount(fmt.Sprintf("%s/test", homePath), testBindPath, "", syscall.MS_BIND, ""))
```

上例将家目录下的test目录bind mount到rootfs的test目录，从而在容器内部可见。

### Cgroup资源扩展（Memory/CPU）

在memory和cpu下创建两个cgroup，设置好限制内容，并将container pid加入到这两个cgroup中：

```go
        // Add cpu limitation for 0.3 core
        cpu := filepath.Join(cgroups, "cpu")
        os.Mkdir(filepath.Join(cpu, "liz"), 0755)
        must(ioutil.WriteFile(filepath.Join(cpu, "liz/cpu.cfs_period_us"), []byte("100000"), 0700))
        must(ioutil.WriteFile(filepath.Join(cpu, "liz/cpu.cfs_quota_us"), []byte("30000"), 0700))
        must(ioutil.WriteFile(filepath.Join(cpu, "liz/notify_on_release"), []byte("1"), 0700))
        must(ioutil.WriteFile(filepath.Join(cpu, "liz/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700))

        // Add memory limitation for 100M
        mem := filepath.Join(cgroups, "memory")
        os.Mkdir(filepath.Join(mem, "liz"), 0755)
        must(ioutil.WriteFile(filepath.Join(mem, "liz/memory.limit_in_bytes"), []byte("100M"), 0700))
        must(ioutil.WriteFile(filepath.Join(mem, "liz/memory.swappiness"), []byte("0"), 0700))
        must(ioutil.WriteFile(filepath.Join(mem, "liz/notify_on_release"), []byte("1"), 0700))
        must(ioutil.WriteFile(filepath.Join(mem, "liz/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700))
```

上例分别对cpu与memory进行了限制：
- **CPU**: 限制container最大使用核数为0.3
- **Memory**: 限制物理内存上限为100M，且禁用swap，即内容使用超过了100M的话立刻触发OOM。

验证cgroup隔离效果：

```shell
root@container:/# cat /proc/self/cgroup
12:perf_event:/
11:cpuset:/
10:devices:/user.slice
9:net_cls,net_prio:/
8:pids:/liz
7:blkio:/user.slice
6:memory:/liz
5:rdma:/
4:hugetlb:/
3:freezer:/
2:cpu,cpuacct:/liz
1:name=systemd:/user.slice/user-1000.slice/session-12.scope
0::/user.slice/user-1000.slice/session-12.scope
```

使用`while : ; do : ; done` 压测cpu limitation：

{{< image src="cpu-limit.png" caption="CPU使用被限制在0.3Core" width=1000 height=200 >}}

验证memory limitation：

{{< image src="mem-limit.png" caption="分配400M内存导致OOM" width=1000 height=200 >}}

### 使用pivot_root系统调用替换chroot

pivot_root与chroot作用类似，都是将rootfs jail到一个目录上。区别在于**前者更改此mount namespace下的所有进程的rootfs，后者仅更改当前进程的rootfs**。

pivot_root核心思想是将root mount更改为**new_root**，并且原root mount会移到**put_old**中。其定义了一系列限制，列举如下：

```text
       -  new_root and put_old must be directories.

       -  new_root and put_old must not be on the same mount as the current root.

       -  put_old must be at or underneath new_root; that is, adding some nonnegative number of "/.." prefixes to the pathname
          pointed to by put_old must yield the same directory as new_root.

       -  new_root  must  be  a path to a mount point, but can't be "/".  A path that is not already a mount point can be con‐
          verted into one by bind mounting the path onto itself.

       -  The propagation type of the parent mount of new_root and the parent mount of the current root directory must not  be
          MS_SHARED;  similarly, if put_old is an existing mount point, its propagation type must not be MS_SHARED.  These re‐
          strictions ensure that pivot_root() never propagates any changes to another mount namespace.

       -  The current root directory must be a mount point.
```

参考 [runc pivotRoot func](https://github.com/opencontainers/runc/blob/v1.0.2/libcontainer/rootfs_linux.go#L817) 对代码进行修改：

```go
        must(syscall.Mount(rootPath, rootPath, "bind", syscall.MS_BIND, ""))
        // jail rootfs with pivot_root syscall
        // ref: https://github.com/opencontainers/runc/blob/v1.0.2/libcontainer/rootfs_linux.go#L817
        putOldPath := filepath.Join(rootPath, "put_old")
        os.Mkdir(putOldPath, 0755)
        must(syscall.PivotRoot(rootPath, putOldPath))
        // lazy unmount
        must(syscall.Unmount("/put_old", syscall.MNT_DETACH))
        if err := os.Remove("/put_old"); err != nil{
                panic(err)
        }
        //must(syscall.Chroot("fmt.Sprintf("%s/ubuntufs", homePath))
        must(os.Chdir("/"))
```
- 为满足第二条限制，`new_root`以bind mount形式脱离current root filesystem。
- 由于存在process使用原root mount下的文件，因此无法直接unmount掉`put_old`。这里使用**lazy unmount**（通过**syscall.MNT_DETACH** flag）的方式卸载掉。Lazy unmount使新的进程看不到此挂载点（隐藏掉），并且当接入此mount的进程全部退出后将其真正卸载掉。

{{< admonition note >}}
**上述扩展的完整代码参考[这里](https://github.com/raygecao/containers-from-scratch/pull/1/files)。**
{{< /admonition >}}
## 知识延伸

为探究`Line:34` Unshareflags对挂载的影响，我们了解一下挂载传播。

### 挂载传播

Mount namespace有时会因提供太强的隔离性导致便捷性降低的问题，比如将一个新磁盘加载到一个光驱驱动器中，当有多个mount namespace时，需要将该磁盘挂载到每个namespace中。为了**只使用一次挂载命令就可以将磁盘挂载到所有的mount namespace**，Linux 从2.6.15起引入了**共享子树特性（Shared Subtrees）**，即**允许在namespace之间自动、可控地传播挂载和卸载事件**。

此特性下，每个挂载点都有一个**传播类型**，此类型决定在此挂载点下创建/删除的挂载点能否传播到其他挂载点下，传播类型有如下四种：

| 传播类型        | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| `MS_SHARED`     | 该挂载点下同一对等组中的挂载点**双向共享**挂载和卸载事件   |
| `MS_SLAVE`      | 该挂载点下同一对等组中的主挂载点可以将挂载或卸载事件**单向传播**到从属挂载点 |
| `MS_PRIVATE`    | 挂载点**不会将事件传播给任何对等方**，同时也不会接收事件     |
| `MS_UNBINDABLE` | 在`MS_PRIVATE`基础上**不能作为绑定挂载操作的源**             |

判断挂载点的默认类型基本方法如下：

- 如果挂载点非根挂载点，且其父节点传播类型是`MS_SHARED`，则新挂载点的传播类型也是`MS_SHARED`。
- 否则挂载点的传播类型是`MS_PRIVATE`。

有了这个概念，我们验证一下这个`Unshareflag`加与不加的区别：

添加`Unshareflags`：

```shell
# 容器中
root@container:/# cat /proc/self/mountinfo
4153 4751 0:610 / /proc rw,relatime - proc proc rw
4223 4751 0:642 / /mytemp rw,relatime - tmpfs thing rw
```

未添加`Unshareflags`：

```shell
# 容器中
root@container:/# cat /proc/self/mountinfo
4750 4223 0:610 / /proc rw,relatime shared:304 - proc proc rw
4753 4223 0:642 / /mytemp rw,relatime shared:312 - tmpfs thing rw

# 容器外
$ cat /proc/self/mountinfo | grep ubuntufs
4751 25 0:610 / ~/ubuntufs/proc rw,relatime shared:304 - proc proc rw
4754 25 0:642 / ~/ubuntufs/mytemp rw,relatime shared:312 - tmpfs thing rw
```

由于实验系统根挂载点是`MS_SHARED`类型，新建子挂载点的默认类型为`MS_SHARED`，因此container内的挂载会共享到host。

解决办法有两种:

- 如代码展示那样，加入`Unshareflags=syscall.CLONE_NEWNS` 参考 [issue-38471](https://go-review.googlesource.com/c/go/+/38471)。
- 指定根挂载为`MS_PRIVATE`类型，如`must(syscall.Mount("", "/", "", syscall.MS_PRIVATE|syscall.MS_REC, ""))`。

### Unshare

Linux `unshare` 指令可以新建namespace，并在namespace中运行程序，支持的命名空间类型有：

- Mount namespace，默认使用**private**传播类型。
- UTS namespace
- IPC namespace
- Network namespace
- Pid namespace
- User namespace

欲达到上述的隔离效果，可以通过如下命令来进行：

```shell
$ sudo unshare --fork --pid --mount-proc --uts /bin/bash
```

- `--pid`声明了pid namespace隔离。
- `--mount-proc`挂载了proc，并声明了mount namespace隔离。
- `--uts`声明了uts namespace隔离。

## 参考文献
- [containers-from-scratch](https://github.com/lizrice/containers-from-scratch)
- [挂载命名空间和共享子树](https://cloud.tencent.com/developer/article/1531989)
- [unshare(1)](https://man7.org/linux/man-pages/man1/unshare.1.html)
