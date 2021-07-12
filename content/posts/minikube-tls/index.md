---
weight: 20
title: "Minikube中的TLS认证探秘"
subtitle: ""
date: 2021-07-11T22:14:32+08:00
lastmod: 2021-07-11T22:14:32+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["k8s", "tls"]
categories: ["探索与实战"]


toc:
  auto: false
lightgallery: true
license: ""
---
诸多巧合凑在一起就是完美的偏差
<!--more-->

## 缘起

对TLS的认知最初应该在学习计算机网络中的https协议，该协议通过TLS层对传输数据进行**加密解密**。为了防止**中间人攻击**，需要第三方证书机构进行**认证**，认证的方式是通过**数字签名**。基本上这几点就涵盖了所有的考纲内容。然而当我在学习或者在工作中真的遇到TLS引起的认证问题时，我发现我所理解的十分笼统，无法给我提供任何有价值的排查思路，因此准备稍微深挖一下TLS的认证机制。

可是载体为什么是minikube？最近在跟随大佬熟悉一些**operator**相关的机制，在开始前搭了一套minikube环境，想着能有更好的开发体验，打算在本地去连服务器的minikube集群，后面有机会再去研究一下**telepresence**以便在本地调试。连接服务器的k8s集群相关的文档也有很多，核心的解决方案就是**将minikube集群中的kubeconfig及对应的证书文件拷贝到本机**。顺着这个思路一顿操作猛如虎，结果在使用kubectl获取资源时，出现了x509证书认证错误。

{{< admonition failure "kubectl get pods">}}
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "minikubeCA")
{{< /admonition >}}
在google没有找到合适的诱因及解决方案，因此打算自己探秘一番。有关TLS认证，x509证书以及openssl相关的命令我会记录在另一篇文章中，本文主要记录k8s的TLS双向认证的过程。

## 初识

k8s在权限管理上做的格外细致，尤其是在k8s api的访问上，更是细分为**认证、鉴权和准入**等阶段。我们这里只对tls双向认证做粗略的描述。

什么是双向认证呢？简单来说就是服务器需要验明客户端的身份，同时客户端也需要验明服务器的身份。k8s的认证载体为x509证书。在minikube集群中有一个统一的CA（默认路径为`.minikube/ca.crt`）来对证书进行验签。

双向认证发生在任何两个进行通信的组件中，下面列举一些例子：

- **etcd集群内peer间通信**是双向的，etcd peer既充当客户端，又充当服务端。因此既需要持有客户端证书，又需要持有服务端证书进行认证。
- **kube-apiserver与etcd之间的通信**是单向的，kube-apiserver充当客户端，etcd充当服务端。因此kube-apiserver需持有客户端证书，etcd需持有服务端证书。
- **kubectl与apiserver的通信**是单向的，kubectl充当客户端，etcd充当服务端，kubectl需要持有客户端证书，apiserver需要持有服务端证书。

- ......

每个控制面组件在启动时会指定所使用到的证书，下例列举出apiserver的启动参数

```shell
# 获取kube-apiserver 相关配置
$ kubectl get pods/kube-apiserver-minikube -o yaml --namespace kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.17.0.40
    - ......
    # 用于验证访问apiserver客户端证书的CA根证书
    - --client-ca-file=/var/lib/minikube/certs/ca.crt       
    - ....
    # 与客户端通信的服务端证书即私钥
    - --tls-cert-file=/var/lib/minikube/certs/apiserver.crt
    - --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
    # 用于验证服务器证书的CA根证书
    - --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt
    # 用于访问etcd的客户端证书
    - --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt
    # 与etcd通信使用的私钥
    - --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key
    # kubelet相关证书及私钥
    - --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt
    - --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key
    # kube-proxy相关证书及私钥
    - --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt
    - --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key
    - --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt
    - ....
```

从上面不难看出，对于一条通信链路的服务端和客户端，都需要提供**一个证书，一个私钥**，和**一个验证对端证书的CA根证书**。

再来看看我们的kubeconfig，里面有kubectl的客户端证书及秘钥，以及验证apiserver服务端证书的CA根证书。

```shell
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    # 用于验证apiserver服务端证书的根证书
    certificate-authority: $HOME/.minikube/ca.crt
    server: https://172.17.0.40:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    # kubectl持有的客户端证书及私钥
    client-certificate: $HOME/.minikube/profiles/minikube/client.crt
    client-key: $HOME/.minikube/profiles/minikube/client.key
```

## 寻幽

下面我们来验证一下kubectl与apiserver的证书及双向认证。

从上一节可以推断出kubectl中CA `$HOME/.minikube/ca.crt`可以验证apiserver的服务端证书 `/var/lib/minikube/certs/apiserver.crt`，apiserver的CA `/var/lib/minikube/certs/ca.crt `可以验证kubectl的客户端证书`$HOME/.minikube/profiles/minikube/client.crt`，验证结果如下：

```shell
# 服务端证书验证
$ openssl verify -CAfile ~/.minikube/ca.crt /var/lib/minikube/certs/apiserver.crt

/var/lib/minikube/certs/apiserver.crt: O = system:masters, CN = minikube
error 7 at 0 depth lookup:certificate signature failure
139668477843096:error:0407006A:rsa routines:RSA_padding_check_PKCS1_type_1:block type is not 01:rsa_pk1.c:103:
139668477843096:error:04067072:rsa routines:RSA_EAY_PUBLIC_DECRYPT:padding check failed:rsa_eay.c:705:
139668477843096:error:0D0C5006:asn1 encoding routines:ASN1_item_verify:EVP lib:a_verify.c:218:

# 客户端证书验证
$ openssl verify -CAfile  /var/lib/minikube/certs/ca.cr ~/.minikube/profiles/minikube/client.crt

$HOME/minikube/profiles/minikube/client.crt: O = system:masters, CN = minikube-user
error 7 at 0 depth lookup:certificate signature failure
139864367138456:error:0407006A:rsa routines:RSA_padding_check_PKCS1_type_1:block type is not 01:rsa_pk1.c:103:
139864367138456:error:04067072:rsa routines:RSA_EAY_PUBLIC_DECRYPT:padding check failed:rsa_eay.c:705:
139864367138456:error:0D0C5006:asn1 encoding routines:ASN1_item_verify:EVP lib:a_verify.c:218:
```

双方验证均未通过，难道是根证书有问题？

```shell
# 查看apiserver客户端证书
$ openssl x509 -text -in /var/lib/minikube/certs/apiserver.crt -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=minikubeCA
        Validity
            Not Before: Aug  8 09:07:42 2020 GMT
            Not After : Aug  9 09:07:42 2021 GMT
        Subject: O=system:masters, CN=minikube
        Subject Public Key Info:
        ......
    X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Alternative Name:
            ......
    ......
    
# 查看kubectl ca根证书
$ openssl x509 -text -in /var/lib/minikube/certs/ca.crt -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=minikubeCA
        Validity
            Not Before: Aug  8 09:07:41 2020 GMT
            Not After : Aug  7 09:07:41 2030 GMT
        Subject: CN=minikubeCA
......
```

apiserver服务端证书和kubectl的CA证书输出也符合预期，前者的Issuer与后者的Subject相同，说明**前者的证书是后者颁发的**；后者Issuer与Subject相同，说明**其是一个合法的自签名证书**。

我是谁，我在哪...

明明服务器上kubectl可以访问到apiserver，证书的颁发关系也没问题，而且TLS双向认证原理也是无懈可击，为什么openssl证书验证就是有问题的呢？



## 溯源

上述结论存在一个致命的错误
{{< admonition bug >}} 
**证书A的Issuer与证书B的Subject相同，说明A的证书是B颁发的**
{{< /admonition >}}
这个错误犯得自己都想笑，x509证书中Issuer代表**颁发机构**，而Subject代表**证书所有者**，A的Issuer与B的Subject相同，只能说明A的证书颁发机构与B的所有者重名，并不能说明A一定是B颁发的。**这就好比小明的妈妈叫小丽，但是叫小丽的不一定是小明的妈妈**。证书认证本质上是通过**数字签名**进行的，签名的私钥是唯一的，但机构名称并不唯一，因此验签失败并不与实际矛盾。

但是事出反常必有妖，**叫小丽的可能有很多，但是一个房子里就两个人，并且他们都叫小丽就很不寻常**。这就让我开始怀疑环境中是不是有另一套minikubeCA根证书，**即存在不止一个minikube**？

在验证kubectl客户端证书的时候留意到`~/.minikube`的配置目录中也有个apiserver.crt

```shell
$ tree ~/.minikube
minikube
├── addons
├── ca.crt
├── ca.key
├── ca.pem
├── cert.pem
├── certs
│   ├── ca-key.pem
│   ├── ca.pem
│   ├── cert.pem
│   └── key.pem
├── config
│   └── config.json
├── files
├── key.pem
├── last_update_check
├── logs
├── machines
│   ├── minikube
│   │   ├── config.json
│   │   ├── id_rsa
│   │   └── id_rsa.pub
│   ├── server-key.pem
│   └── server.pem
├── profiles
│   └── minikube
│       ├── apiserver.crt
│       ├── apiserver.crt.49553cf0
│       ├── apiserver.key
│       ├── apiserver.key.49553cf0
│       ├── client.crt
│       ├── client.key
│       ├── config.json
│       ├── events.json
│       ├── proxy-client.crt
│       └── proxy-client.key
├── proxy-client-ca.crt
└── proxy-client-ca.key
```

使用openssl使用Kubectl的服务端验证根证书验证一下这个证书，验证通过。

```shell
$ openssl verify -CAfile  ~/.minikube/ca.crt ~/.minikube/profiles/minikube/apiserver.crt
$HOME/.minikube/profiles/minikube/apiserver.crt: OK
```

这时候另一个疑惑就涌上来了：为什么**kubectl客户端证书和apiserver服务端证书会放在完全不相关的两个目录下**？又挖一挖config文件，发现有auth path相关的路径配置。

```shell
$ cat ~/.minikube/machines/minikube/config.json
{
  ....
      {
        "AuthOptions": {
            "CertDir": "$HOME/.minikube",
            "CaCertPath": "$HOME/.minikube/certs/ca.pem",
            "CaPrivateKeyPath": "$HOME/.minikube/certs/ca-key.pem",
            "CaCertRemotePath": "",
            "ServerCertPath": "$HOME/.minikube/machines/server.pem",
            "ServerKeyPath": "$HOME/.minikube/machines/server-key.pem",
            "ClientKeyPath": "$HOME/.minikube/certs/key.pem",
            "ServerCertRemotePath": "",
            "ServerKeyRemotePath": "",
            "ClientCertPath": "$HOME/.minikube/certs/cert.pem",
            "ServerCertSANs": null,
            "StorePath": "$HOME/.minikube"
        }
}
```

尽管没看过minikube的源码，也没在官方文档上找到很明确的说法，但是基本上可以确定minikube启动的所有控制面组件使用的证书是在这里配置的，也就是完全存放在`~/.minikube`path中。而pod中挂载的`/var/lib/minikube/certs`应该并非挂是宿主机对应的path上，毕竟minikube是运行在VM中的，挂载路径的灵活性便可想而知。
{{< admonition info "minikube的定位" >}}
Minikube is a tool that makes it easy to run Kubernetes locally. Minikube runs a single-node Kubernetes cluster inside a Virtual Machine (VM) on your laptop for users looking to try out Kubernetes or develop with it day-to-day.
{{< /admonition >}}
## 破疑

接下来的就是之前使我们步入泥淖的两个问题：

- 在本机执行kubectl为什么会出现x509证书认证失败？

- 宿主机上为什么会有/var/lib/minikube/certs一系列证书？

这两个问题有着很强的相关性，一个大胆的猜想就是**本机连的apiserver是在/var/lib/minikube/certs中认证的**。

首先来看一下apiserver的配置

```shell
# 查看apiserver的配置
$ kubectl get pods/kube-apiserver-minikube -o yaml --namespace kube-system

spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.17.0.40
    - --secure-port=8443
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/var/lib/minikube/certs/ca.crt
...
  podIP: 172.17.0.40
    podIPs:
    - ip: 172.17.0.40
```

从apiserver的配置中我们可以看到对应的podIP并不是宿主机的ip，而是个虚拟机的ip。这个再结合**minikube在虚拟机中运行k8s**很容易理解。那么也就是说minikube并没有暴露在宿主机的8443端口。可是我们执行kubectl**并不是返回端口connection refused，而是返回的x509认证失败**，这也就是宿主机的8443端口被监听。使用ps查看apiserver的进程，结果居然存在2个进程。

这样之前的灵异现象都能串起来了：**宿主机还有一个额外的minikube集群，并且对外暴露了宿主机8443端口，使用/var/lib/minikube/certs进行认证**。

经过一番查证与折腾，发现在/etc/kubernetes下有一些kubeconfig file以及控制面组件的部署文件。

```shell
$ tree /etc/kubernetes
/etc/kubernetes
├── addons
│   ├── dashboard-clusterrolebinding.yaml
│   ├── dashboard-clusterrole.yaml
│   ├── dashboard-configmap.yaml
│   ├── dashboard-dp.yaml
│   ├── dashboard-ns.yaml
│   ├── dashboard-rolebinding.yaml
│   ├── dashboard-role.yaml
│   ├── dashboard-sa.yaml
│   ├── dashboard-secret.yaml
│   ├── dashboard-svc.yaml
│   ├── ingress-configmap.yaml
│   ├── ingress-dp.yaml
│   ├── ingress-rbac.yaml
│   ├── storageclass.yaml
│   └── storage-provisioner.yaml
├── admin.conf
├── controller-manager.conf
├── kubelet.conf
├── manifests
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml
│   └── kube-scheduler.yaml
└── scheduler.conf
```

然后在服务器上指定kubeconfig为`admin.conf`执行kubectl，果然结果来自另外的一个集群。这个集群应该是我在最早接触k8s时使用的版本比较低的minikube搭建的，至于为啥部署形态会成这样我也不清楚至今仍然未解。

观察这个Minikube部署的apiserver，的确对外暴露的宿主机的8443端口。

```shell
$ sudo KUBECONFIG=/etc/kubernetes/admin.conf kubectl get pods/kube-apiserver-XX --namespace kube-system -o yaml
......
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.122.101.148
    - --secure-port=8443
    - --client-ca-file=/var/lib/minikube/certs/ca.crt
    - --tls-cert-file=/var/lib/minikube/certs/apiserver.crt
    - --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
    - ......
......
 podIP: 10.122.101.148
   podIPs:
   - ip: 10.122.101.148
   qosClass: Burstable
```

将admin.conf中的证书与/var/lib/minikube/certs下的证书进行互相验签，均可通过。真相由此浮出水面。

## 前行

知道了问题的症结，需要向最初的目的前进了。先忽视掉上面揪出来的野minikube，尝试连接新搭建的minikube，另一个后续会通过一个额外的context加入到本地多集群管理中。

由于8443端口已经被先前的端口占用了，那么我们就选择暴露一个额外的端口**38443**吧。

```shell
# 首先启动minikube（我在启动前删除了之前部署的minikube数据），指定apiserver的端口为38443，指定apiserver的ip为10.122.101.148
# apiserver 端口其实不必指定，后续会介绍须引入本地端口转发来将服务暴露出去
$ minikube start --apiserver-ips=10.122.101.148 --apiserver-port=38443

# 查看apiserver的配置
$ kubectl get pods/kube-apiserver-minikube -o yaml --namespace kube-system

spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.17.0.40
    - --allow-privileged=true
    - --secure-port=38443 
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/var/lib/minikube/certs/ca.crt
...
  podIP: 172.17.0.40
    podIPs:
    - ip: 172.17.0.40
```

What? 不是设置了apiserver的ip为宿主机的ip了吗？为什么podIP还是虚拟机的ip？文档里还特意的解释了flag **apiserver-ips**的作用，就是要 **make the apiserver available from outside the machine**。

```shell
$ minikube start --help
Starts a local Kubernetes cluster

Options:
      --addons=[]: Enable addons. see `minikube addons list` for a list of valid addon names.
      --apiserver-ips=[]: A set of apiserver IP Addresses which are used in the generated certificate for kubernetes.
This can be used if you want to make the apiserver available from outside the machine
      --apiserver-name='minikubeCA': The authoritative apiserver hostname for apiserver certificates and connectivity.
This can be used if you want to make the apiserver available from outside the machine
      --apiserver-names=[]: A set of apiserver names which are used in the generated certificate for kubernetes.  This
can be used if you want to make the apiserver available from outside the machine
      --apiserver-port=8443: The apiserver listening port
......
```

[apiserver-ips option is not working](https://github.com/kubernetes/minikube/issues/11320) 这个issue专门描述了此问题，明确指出此参数**仅会将对应的ip添加到证书SAN，不会修改apiserver的podIP**。当然，作者困惑是否存在这种使用方式的场景。虽然不是很明确这样使用会带来什么样的问题（maybe端口冲突？安全？可扩展性？），但是作为初入坑k8s的同学来讲，这种简单的访问方式可能会降低一些我们的学习门槛。

{{< image src="issue11320.jpg" height=500 width=1000 caption="apiserver-ips option is not working">}}

既然这样，怎样才能将apiserver可以被本地访问到呢？这个也简单，可以通过**ssh本地端口转发**进行。

```shell
# 将服务器10.122.101.148的38443端口转发到172.17.0.40的38443端口
$ ssh -L 10.122.101.148:38443:172.17.0.40:38443 -N -f 10.122.101.148
```

最后，只需要将我们的kubeconfig的server字段改成**10.122.101.148:38443**就可以开心的访问远程的minikube了。


{{< admonition tip "x509证书扩展">}}
**将对应ip添加到证书SAN**，这个解释我并不是很理解，直到后面做了个实验才有些感觉。在没指定**apiserver-ips**去curl https://10.122.101.148，会出现诸如**Hostname 10.122.101.148 doesn't match certificate's altnames: "Host: XXX. is not in the cert's altnames:XXX**的错误，而指定了**apiserver-ips=10.122.101.148**后，apiserver的服务端证书便可以通过验证。这是因为该参数将ip加入到了服务端证书的SAN扩展字段中，使得10.122.101.148成为证书可信的域。

```shell
$ openssl x509 -text -in .minikube/profiles/minikube/apiserver.crt -noout

X509v3 extensions:
    X509v3 Subject Alternative Name:
       DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:
kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:10.122.101.148, IP Address:172.17.0.40, IP Address:10.96.0.1, IP Address:127.0.0.
1, IP Address:10.0.0.1
```
{{</admonition>}}

## 插曲

在验证双向认证时，为了避免两端都要使用openssl验证，便想通过curl去验证。使用curl验证的结果让我再次怀疑人生...

```shell
# 执行双向验证，客户端验证服务端证书失败
$ curl https://10.122.101.148:38443  --cert ~/.minikube/profiles/minikube/client.crt --key ~/.minikube/profiles/minikube/client.key --cacert ~/.minikube/ca.crt

curl: (60) server certificate verification failed. CAfile: ~/.minikube/ca.crt CRLfile: none
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
 
 # 关闭客户端的服务端证书验证，正常返回
$ curl https://10.122.101.148:38443  --cert ~/.minikube/profiles/minikube/client.crt --key ~/.minikube/profiles/minikube/client.key -k
 
 {
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    ...
    ]
 }
  
# 确认客户端CA对服务端的证书验证结果，认证通过
openssl verify -CAfile  ~/.minikube/ca.crt ~/.minikube/profiles/minikube/apiserver.crt
~/.minikube/profiles/minikube/apiserver.crt: OK
```

上面的测试结果看上去互相矛盾，折腾了很久，大概查了下curl的文档，感觉自己做的并没有问题，大胆的怀疑一下是curl的版本问题，于是准备在一个curl版本更高的镜像里去执行这个curl命令。

```shell
# 在golang:1.16.2镜像中通过curl验证双向认证，成功
$ docker run --rm -v $HOME/.minikube:/minikube golang:1.16.2 curl https://10.122.101.148:38443 --cert /minikube/profiles/minikube/client.crt --key /minikube/profiles/minikube/client.key --cacert /minikube/ca.crt

 {
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    ...
    ]
 }
  
```

果然是旧版本curl的问题。由于目前为止在这上面浪费了比较久的时间，并且感觉越往后排查越偏离初始方向，因此便未继续往下追溯。二者版本列举如下，希望自己未来有时间、有兴趣去往底层排查一下。

```shell
# 服务器10.122.101.148上curl版本
$ curl --version

curl 7.47.0 (x86_64-pc-linux-gnu) libcurl/7.47.0 GnuTLS/3.4.10 zlib/1.2.8 libidn/1.32 librtmp/2.3
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp smb smbs smtp smtps telnet tftp
Features: AsynchDNS IDN IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz TLS-SRP UnixSockets

# go:1.16.2 image中curl版本
$ docker run --rm -v $HOME/.minikube:/minikube golang:1.16.2 curl --version

curl 7.64.0 (x86_64-pc-linux-gnu) libcurl/7.64.0 OpenSSL/1.1.1d zlib/1.2.11 libidn2/2.0.5 libpsl/0.20.2 (+libidn2/2.0.5) libssh2/1.8.0 nghttp2/1.36.0 librtmp/2.3
Release-Date: 2019-02-06
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp scp sftp smb smbs smtp smtps telnet tftp
Features: AsynchDNS IDN IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz TLS-SRP HTTP2 UnixSockets HTTPS-proxy PSL

```



## 悟道

- **你永远不知道你不知道的事**，只有学习才会让我们发现原本我们不知道的事，哪怕过程使我们谦(zi)卑。
- 最初只是想体验一下telepresence，仅仅是一个准备工作就牵出如此多的知识点，遇到问题是痛苦的，但寻找解决问题的方法是快乐的。
- 与安全相关的问题搞得多复杂都不为过，仅仅是k8s认证阶段中的证书认证这一步就如此的丰富，后面的鉴权、准入等过程会更加刺激。
- 工具的使用很皮毛，比如curl，openssl，不过比较复杂的工具往往在实际问题排查时仅需要最基本的使用方法。
- 仍然存在诸多基础知识盲区，比如 **ssh端口转发**， **数字签名，x509证书相关概念**等，后面会有专门的文章来记录这些基础。
- 好记性不如烂笔头，文章落笔于解决问题的第二天，但是能记住的甚微，写文章时基本又根据history重放了一遍才将脉络重新梳理清楚。



## 启明

1. [远程访问minikube](https://cloud-atlas.readthedocs.io/zh_CN/latest/kubernetes/startup_prepare/remote_minikube.html#minikube)

2. [一文带你彻底厘清 Kubernetes 中的证书工作机制](https://zhaohuabing.com/post/2020-05-19-k8s-certificate/)
3. [Installing Kubernetes with Minikube](https://v1-18.docs.kubernetes.io/docs/setup/learning-environment/minikube/)
4. [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
5. [apiserver-ips option is not working](https://github.com/kubernetes/minikube/issues/11320) 
