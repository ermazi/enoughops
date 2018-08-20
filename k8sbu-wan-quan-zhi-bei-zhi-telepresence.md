# k8s不完全指北之本地开发工具telepresence

> 大开发："嘿，小运维~，帮忙把kubernetes的服务IP以及POD的IP暴露出来，我本地需要访问。"
>
> 小运维："不，请把服务放到k8s中调试。"
>
> 大开发（白眼）："不就是指个路由的事吗？"
>
> 小运维："这..."
>
> （小运维由于没法解决k8s集群内部服务本地联调的需求，k8s的推广停滞不前；开发原本上k8s的激情，不复存在；年终奖也就木有了，木有了~）

话说回来，我们从技术层面想想，这个需求是否合理。

原则上，大部分k8s内部服务都是不对外暴露的，k8s内部服务使用service的clusterip访问，但是本地开发的时候，我们会用到很多别人的东西（http或者rpc），希望直接调用，而不是本地起一整套。。。另外现在微服务盛行，服务注册和服务发现也成为刚需，我本地开发调试，自然会注册，并且从注册中心发现对应的服务所对应的ip，而且是`POD IP`，综上，所以开发的这个需求是刚需。√

然而矛盾面是，cluster-ip以及pod-ip都是集群内部的ip，pod-ip是局限于k8s集群的内部子网，是实实在在存在的，但cluster-ip是虚拟ip，仅仅在k8s内部可用。你一个laptop开发环境，怎么能够访问到service或者pod对应的ip？？？难受~

但是天无绝人之路，目前有以下几种方式：

- 任性型，硬件支持，pod ip直接和物理办公网络打通，但是也没法解决service ip的问题
- openvpn型，直接kubernetes内部安装openvpn，所有laptop安装openvpn client，接受推送的路由以及dns，可以解决service ip访问问题，以及pod ip访问问题，而且互通哟，比较完美。具体的连接见：[kube-openvpn](https://github.com/pieterlange/kube-openvpn)
- 工具型，这里主要是指`telepresence`，支持各种乱七八糟地模式，符合需求，基本上不需要手动做啥，[官网](https://www.telepresence.io/)



## + 安装

### ++ mac os x

```bash
brew cask install osxfuse
brew install socat datawire/blackbird/telepresence
```



### ++ ubuntu

```bash
curl -s https://packagecloud.io/install/repositories/datawireio/telepresence/script.deb.sh | sudo bash
sudo apt install --no-install-recommends telepresence
```



其它系统安装，见[install](https://www.telepresence.io/reference/install)，excuse me?Windows？Sorry，not fully support！瞧[这里](https://www.telepresence.io/reference/windows)



## + 简单实用

额，文档比较复杂，首先你得有个kubeconfig文件，跟你们的小运维要，不给？陪吃陪喝陪睡，软磨硬泡呗，好吧，可能小运维也不知道怎么生成。

kubeconfig对应的权限，看[这里](https://www.telepresence.io/reference/connecting)：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: telepresence-clusterrole
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["list", "create", "delete"]
- apiGroups: ["", "extensions"]
  resources: ["deployments"]
  verbs: ["list", "create", "get", "update", "delete"]
- apiGroups: ["", "extensions"]
  resources: ["deployments/scale"]
  verbs: ["get", "update"]
- apiGroups: ["", "extensions"]
  resources: ["replicasets"]
  verbs: ["list", "get", "update", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

然后就正常用吧，看懂[这个](https://www.telepresence.io/discussion/how-it-works)，基本就知道能做啥，不能做啥了。

首先你得建个通道吧，随便来个：

```bash
telepresence --run /bin/bash
```

进入这个交互式界面：

```bash
➜  ~ telepresence --run /bin/bash
T: Invoking sudo. Please enter your sudo password.
Password:
T: Starting proxy with method 'vpn-tcp', which has the following limitations: All processes are affected, only one
T: telepresence can run per machine, and you can't use other VPNs. You may need to add cloud hosts and headless
T: services with --also-proxy. For a full list of method limitations see
T: https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See https://telepresence.io/howto/volumes.html for details.

T: No traffic is being forwarded from the remote Deployment to your local machine. You can use the --expose option to
T:  specify which ports you want to forward.

@kubernetes-welab|bash-3.2$
```

官方的~~--run-shell~~，不好意思，找不到对应的bash，报错。

运行了上述命令，k8s内部也会启动一个`Deployment`资源， 当本地进程销毁的时候，远端的`Deployment`也会被销毁。

按照howitworks的文档，只有被代理的进程拥有k8s的所有环境变量，而其他进程并没有对应的环境变量。

当然，它给的提示也仔细瞅瞅，部分命令使用是有限制的，你没法使用其他vpn，并且部分dns无法解析，(⊙o⊙)…额，还好吧。。。

## + 还能做啥

- 本地的服务暴露给k8s集群，通过service访问
- 访问k8s集群内部的服务



## + 随便试试

不信，日志截屏来见。

### ++ 直接访问k8s内部服务

```bash
➜  ~ curl urlechoserver
{"url":"/","method":"GET","host":"urlechoserver","header":{"Accept":["*/*"],"User-Agent":["curl/7.54.0"]}}
```



### ++ 暴露本地服务给k8s

启动：

```bash
➜  code telepresence --expose 8080 \
             --new-deployment enoughops \
             --run python3 -m http.server 8080
T: Starting proxy with method 'vpn-tcp', which has the following limitations: All processes are affected, only one
T: telepresence can run per machine, and you can't use other VPNs. You may need to add cloud hosts and headless
T: services with --also-proxy. For a full list of method limitations see
T: https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See https://telepresence.io/howto/volumes.html for details.
T: Forwarding remote port 8080 to local port 8080.

Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
127.0.0.1 - - [20/Aug/2018 21:44:19] "GET / HTTP/1.1" 200 -
```

访问：

```bash
➜  ~ curl http://enoughops:8080
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="telepresence.log">telepresence.log</a></li>
<li><a href="weops/">weops/</a></li>
<li><a href="wepipe/">wepipe/</a></li>
</ul>
<hr>
</body>
</html>
```



### ++ 将k8s中的服务环境暴露给本地，包括环境变量

```bash
➜  ~ telepresence --swap-deployment urlechoserver --env-json urlechoserver_env.json --run /bin/bash
```

将对应的环境变量保存到`urlechoserver_env.json`中，可用于ide设置环境变量调试。



## + 说在最后

其实最依赖的也就是那条通道，随意回归最简，直接使用`telepresence --run /bin/bash`创建一个隧道，然后就可以直接访问大部分资源了，当然，如果你需要环境变量，那么再考虑其他手段，比如通过`--swap-deployment`获取环境变量。

运维从简，够用就行。