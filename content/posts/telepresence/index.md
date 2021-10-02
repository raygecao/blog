---
weight: 20
title: "Telepresence——本地调试k8s服务利器"
subtitle: ""
date: 2021-10-02T10:16:55+08:00
lastmod: 2021-10-02T10:16:55+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["k8s"]
categories: [探索与实战]


toc:
  auto: false
lightgallery: true
license: ""
---

Telepresence用于在本地轻松开发和调试服务，同时将服务代理到远程 Kubernetes 集群。 使用 telepresence 可以为本地服务使用自定义工具（如调试器和 IDE）， 并提供对 Configmap、Secret 和远程集群上运行的服务的完全访问。

<!--more-->

## 背景

应用容器化带来了高效率、强扩展性等优势的的同时也带来一些复杂性。尤其对于开发人员，需要将修改部署到容器中（**构建容器->发布到registry->部署到集群中**）。如果使用k8s这种容器平台，还需要维护容器编排及配置从而延长开发迭代周期。总之，在容器化应用中，研发要对软件的整个生命周期负责。

为了提升开发效率，需要引入将远程k8s集群与本地开发桥接起来的方法，以此减少反馈时间，提升debug效率。加速反馈的最佳实现方式是**单个本地服务，其他依赖服务都是远程**的，telepresence便是基于此思想的一个加速反馈， 便于调试及协作的开发者工具。

Telepresence被设计为让k8s**开发者的笔记本如同加入到了k8s集群中**一样，能够将服务在本地运行，并且被proxy到远端集群中。

## 架构

{{< image src="architecture.svg" caption="Telepresence架构图" width=800 height=200 >}}

- **Telepresence CLI**：用于编排出所有其他的组件，如telepresence daemon，traffic manager，授权Ambassador Cloud等。既起到bootstrap作用，又起到发送控制命令作用。

  ```shell
  # 首次执行telepresence list cli
  $ telepresence list
  Launching Telepresence Daemon v2.3.2 (api v3)
  Connecting to traffic manager...
  Connected to context minikube (https://10.122.101.148:38443)
  frontend      : ready to intercept (traffic-agent not yet installed)
  redis-follower: ready to intercept (traffic-agent not yet installed)
  redis-leader  : ready to intercept (traffic-agent not yet installed)
  ```

  上例可以看到，在初次执行telepresence cli时，其先启动了telepresence daemon，紧接着安装并连接traffic manager，最后才返回对应cmd的结果。

- **Telepresence Daemon**：是开发者本地的代理点。

- **Traffic Manager**：集群流量入口点，也是流量控制的中枢代理。同时它会与Ambassador Cloud交互以支持**Preview URL**的特性。

- **Traffic Agent**：用于流量代理的sidecar，主要根据拦截规则判断到达的请求，要么将请求直接交付给pod对应的端口，要么路由到traffic manager以转发到本地服务中。

{{< admonition note "拦截方式" >}}

目前主要的拦截方式有两种：**全量拦截**和**preview URL**，前者traffic agent会全量地路由给traffic manager；后者会对请求进行判断，只有preview URL请求会路由到traffic manager，其他请求交付给pod内的服务。

{{< /admonition >}}

- **Ambassador Cloud**：产生临时域名，将preview URL从授权的用户路由到traffic manager。

## 使用

### 准备工作

[安装telepresence](https://www.telepresence.io/docs/latest/install/)

[k8s集群中安装guestbook demo](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)

```shell
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
frontend-7bcb4574cb-j2wwz        1/1     Running   0          39s
frontend-7bcb4574cb-rbxkj        1/1     Running   0          41s
frontend-7bcb4574cb-xxm56        1/1     Running   0          44s
redis-follower-dd4df4648-pvd2c   1/1     Running   5          50d
redis-follower-dd4df4648-wzf54   1/1     Running   5          50d
redis-leader-6d7765b8f6-mbzm8    1/1     Running   5          50d
```

```shell
$ kubectl get svc
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
frontend         ClusterIP   10.111.60.121   <none>        80/TCP     50d
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP    50d
redis-follower   ClusterIP   10.96.4.108     <none>        6379/TCP   50d
redis-leader     ClusterIP   10.101.93.34    <none>        6379/TCP   50d
```

由于k8s环境是通过minikube虚拟机搭建的，为了方便在宿主机访问集群内部服务，我们使用port-forward进行端口转发。

```shell
# 集群服务器148开端口转发，将148宿主机上12180端口的流量转发到集群内frontend service中
$ kubectl port-forward svc/frontend 12180:80 --address=0.0.0.0
```

验证一下frontend service提供的是guestbook服务。

```shell
$ curl 10.122.101.148:12180

<html ng-app="redis">
  <head>
    <title>Guestbook</title>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.12/angular.min.js"></script>
    <script src="controllers.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular-ui-bootstrap/0.13.0/ui-bootstrap-tpls.js"></script>
  </head>
  <body ng-controller="RedisCtrl">
    <div style="width: 50%; margin-left: 20px">
      <h2>Guestbook</h2>
    <form>
    <fieldset>
    <input ng-model="msg" placeholder="Messages" class="form-control" type="text" name="input"><br>
    <button type="button" class="btn btn-primary" ng-click="controller.onRedis()">Submit</button>
    </fieldset>
    </form>
    <div>
      <div ng-repeat="msg in messages track by $index">
        {{msg}}
      </div>
    </div>
    </div>
  </body>
</html>
```



### 连接到集群

```shell
$ telepresence connect
Launching Telepresence Daemon v2.3.2 (api v3)
Connecting to traffic manager...
Connected to context minikube (https://10.122.101.148:38443)
```

需要确保`kubeconfig`的配置，示例中的context连接到的apiserver为`10.122.101.148:38443`

### 服务全量拦截

查看可拦截的服务列表：

```shell
$ telepresence list
frontend      : ready to intercept (traffic-agent not yet installed)
redis-follower: ready to intercept (traffic-agent not yet installed)
redis-leader  : ready to intercept (traffic-agent not yet installed)
```

结果中可以看出有三个service可以拦截，并且都没有安装traffic agent。不过不用担心，我们在进行服务拦截时会自动向pod内注入traffic agent。

拦截frontend留言板服务，拦截前我们先本地18888端口起一个nginx服务。

```shell
$ docker run --rm --name nginx-test -p 18888:80 -d nginx:1.21.1
9d514b006d2c386b7d3676bb40d1dafcbf9211ba9737e213e0dac180496c8d3a
```

验证一下nginx可以正常访问。

```shell
$ curl localhost:18888
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

下面我们来看看如何使用本地的nginx service来拦截集群中的frontend service。其实不过在本地执行一句简单的命令：

```shell
# telepresence intercept <service-name> --port <local-port>[:<remote-port>] --env-file <path-to-env-file>
$ telepresence intercept frontend --port 18888:80 --env-file frontend-svc.env
Using Deployment frontend
intercepted
    Intercept name         : frontend
    State                  : ACTIVE
    Workload kind          : Deployment
    Destination            : 127.0.0.1:18888
    Service Port Identifier: 80
    Volume Mount Error     : sshfs is not installed on your local machine
    Intercepting           : all TCP connections
```

这样，我们便将对`frontend:80`的访问就会被转发到`localhost:18888`，即访问到的是nginx服务。验证一下：

```shell
curl 10.122.101.148:12180 | head -5
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

使用telepresence cli可以查看到服务的拦截状态：

```shell
$ telepresence list
frontend      : intercepted
    Intercept name         : frontend
    State                  : ACTIVE
    Workload kind          : Deployment
    Destination            : 127.0.0.1:18888
    Service Port Identifier: 80
    Intercepting           : all TCP connections
redis-follower: ready to intercept (traffic-agent not yet installed)
redis-leader  : ready to intercept (traffic-agent not yet installed)
```

此外，frontend service的endpoints也从80端口（frontend container）切换到了9900端口（traffic agent）。这意味着pod内访问9900端口（traffic agent）会经由traffic manager转发到本地nginx服务中。

```shell
$ kubectl describe svc frontend
Name:              frontend
Namespace:         default
Labels:            app=guestbook
                   tier=frontend
Annotations:       telepresence.getambassador.io/actions: {"version":"2.3.2","make_port_symbolic":{"PortName":"","TargetPort":80,"SymbolicName":"tx-80"}}
Selector:          app=guestbook,tier=frontend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.111.60.121
IPs:               10.111.60.121
Port:              <unset>  80/TCP
TargetPort:        tx-80/TCP
Endpoints:         172.18.0.2:9900,172.18.0.7:9900,172.18.0.8:9900  # 端口从80切换到了9900
Session Affinity:  None
Events:            <none>
```

{{< admonition info >}}

上面提及到在执行`telepresence intercept`时会自动注入traffic agent这个sidecar，可以观察一下frontend svc对应的endpoint pod的信息。

```shell
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
frontend-698684655-bfd2l         2/2     Running   0          7m50s
frontend-698684655-ddh4c         2/2     Running   0          7m54s
frontend-698684655-zvnmg         2/2     Running   0          7m47s
redis-follower-dd4df4648-pvd2c   1/1     Running   5          50d
redis-follower-dd4df4648-wzf54   1/1     Running   5          50d
redis-leader-6d7765b8f6-mbzm8    1/1     Running   5          50d

$ kubectl describe pod frontend-698684655-bfd2l
Name:         frontend-698684655-bfd2l
Namespace:    default
......
Containers:
  php-redis:
    Image:          gcr.io/google_samples/gb-frontend:v5
    Port:           80/TCP
    ....
  traffic-agent:
    Image:         docker.io/datawire/tel2:2.3.2
    Port:          9900/TCP
    ...
......
```

{{< /admonition >}}

### 拦截Preview URL

一般在开发环境中，会有多个开发人员协同工作。上述的拦截方案的最大问题是**traffic agent无脑将请求全部转发给traffic manger，从而将流量全部转发到本地**。这势必会造成对其他开发人员的影响，因此我们需要能够在不影响其他人的情况下进行拦截调试。实现上利用了context 传递来实现可控的拦截，借助**Ambassador Cloud的preview URL**给我们提供了可控拦截的能力。

Preview URL需要与ingress配合使用，所以第一步我们先创建一个ingress：

```shell
$ cat ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - host: frontend.raygecao.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
                  
$ kubectl apply -f ingress.yaml

$ kubectl describe ing demo-ingress
Name:             demo-ingress
Namespace:        default
Address:          172.17.0.40
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                   Path  Backends
  ----                   ----  --------
  frontend.raygecao.com
                         /   frontend:80 (172.18.0.2:9900,172.18.0.7:9900,172.18.0.8:9900)
Annotations:             nginx.ingress.kubernetes.io/rewrite-target: /$1
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  UPDATE  3m54s (x2 over 20d)  nginx-ingress-controller  Ingress default/demo-ingress
```

然后我们清理掉之前的拦截。

```shell
$ telepresence leave frontend
$ telepresence list
frontend      : ready to intercept (traffic-agent already installed)
redis-follower: ready to intercept (traffic-agent not yet installed)
redis-leader  : ready to intercept (traffic-agent not yet installed)
```

注册并登陆Ambassador Cloud，然后重新进行拦截：

```shell
$ telepresence intercept frontend --port 18888:80 --env-file frontend-svc.env

To create a preview URL, telepresence needs to know how cluster
ingress works for this service.  Please Confirm the ingress to use.

1/4: What's your ingress' layer 3 (IP) address?
     You may use an IP address or a DNS name (this is usually a
     "service.namespace" DNS name).

       [default: 10.122.101.148]:

2/4: What's your ingress' layer 4 address (TCP port number)?

       [default: 9999]:

3/4: Does that TCP port on your ingress use TLS (as opposed to cleartext)?

       [default: n]:

4/4: If required by your ingress, specify a different layer 5 hostname
     (TLS-SNI, HTTP "Host" header) to access this service.

       [default: frontend.raygecao.com]: 
       
Using Deployment frontend
intercepted
    Intercept name         : frontend
    State                  : ACTIVE
    Workload kind          : Deployment
    Destination            : 127.0.0.1:18888
    Service Port Identifier: 80
    Volume Mount Error     : sshfs is not installed on your local machine
    Intercepting           : HTTP requests that match all headers:
      'x-telepresence-intercept-id: 2da7518d-ce3f-4732-9c02-f144de3443a8:frontend'
    Preview URL            : https://musing-kirch-7616.preview.edgestack.me
    Layer 5 Hostname       : frontend.raygecao.com
```

{{< admonition note >}}

由于之前拦截时配置过一次，因此这一次一路默认就好，第一次配置需要结合自身的ingress来配置。

为了使hostname可以访问到集群内的服务，需要手动添加一条DNS记录将hostname resolve到ingress节点上（即在hosts文件中添加一条`172.17.0.40 frontend.raygecao.com`）。或者利用ingress路由请求的原理，直接将hostname指定到`Host`header进行路由。

{{< /admonition >}}

可以看到，Ambassador Cloud为我们生成了一个preview URL `https://musing-kirch-7616.preview.edgestack.me`，并且添加了转发规则：所有包含`'x-telepresence-intercept-id: 2da7518d-ce3f-4732-9c02-f144de3443a8:frontend'`header并且host是**frontend.raygecao.com**的request会被本地nginx服务拦截。

{{< image src="ambassador.png" caption="Ambassador Cloud进行服务拦截管理" width=800 height=200 >}}

从效果上看，添加了这种特殊的header，就会将流量导到preview URL上，这正是ambassador cloud给予我们的便利。我们验证一下具体的拦截情况。

```shell
# 正常的请求不会被本地的nginx服务拦截
$ curl 10.122.101.148:9999 -H 'Host: frontend.raygecao.com' | head -5

<html ng-app="redis">
  <head>
    <title>Guestbook</title>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.12/angular.min.js"></script>

# 添加header会被转发到preview URL中
$ curl 10.122.101.148:9999 -H 'Host: frontend.raygecao.com' -H 'x-telepresence-intercept-id: 2da7518d-ce3f-4732-9c02-f144de3443a8:frontend' | head -5

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

## 参考文献
- [Telepresence官方文档](https://www.telepresence.io/)

