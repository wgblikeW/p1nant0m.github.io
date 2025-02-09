# 在 K8s 集群上部署 Cilium 与 Hubble


{{< admonition type=tip title="关于笔者" open=false >}}

目前，笔者还是一名在校大学生，就读于信息安全专业，对云原生安全、分布式系统、观测领域有较大的兴趣。因而小站上的很多文章都是萌新学习踩坑的经验分享，更多的是“玩”的心态向优秀、有趣的项目，学习其设计思路、编码风格、文档编写、开源社区/团队协作方面的知识，并辅以自己小小的见解。作为一名还未有充分经验的用户，文章中难免有所纰漏，还望能够与大家交流学习，共勉。

{{< /admonition >}}

### 在 K8s 集群上部署 Cilium 与 Hubble

近段时间一直在思考与研究云原生中应用的网络、性能与行为监控/控制平台，Cilium 作为 CNCF 中的后起之秀，利用eBPF 提供对于内核行为观测的强大能力，实现了 K8s 集群工作负载的网络、可观测性以及安全，它的实现方式以及架构思路给了我极大的启发。下图给出了 Cilium 的基本架构，

{{< image src="https://cilium.io/static/diagram-9836e6891afc6fcbf30b85b31ca2b37e.svg" caption="Cilium 基本架构" src_s="https://cilium.io/static/diagram-9836e6891afc6fcbf30b85b31ca2b37e.svg" src_l="https://cilium.io/static/diagram-9836e6891afc6fcbf30b85b31ca2b37e.svg" >}}



#### 使用 Kind 部署 K8s 集群

我们有三种方式去创建 K8s 集群，在这里我们仅作为一个 demo 来去对 Cilium 进行考察，因此我们选择使用最简单的方式去部署 K8s 集群，它为我们免去了许多对集群繁琐的配置，皆在帮助开发者快速的搭建一个集群，并通过这个简易集群来测试自己的云原生应用。想要详细了解 [Kind](https://kind.sigs.k8s.io/) 的可以点击此处，我们这里仅使用其进行快速集群创建。

在 Cilium 的官方文档中给出了使用 Kind 来创建集群所使用的命令，我们会跟随着文档来一步步的创建使用 Cilium 作为 CNI 组件的 K8s 集群。

> 鉴于国内特殊的网络环境情况，在创建集群或在 apply yaml 的时候会出现很多因网络问题导致的错误，这里也会给出笔者自己摸索出来的解决方案，这可能不是最好的解决方案但是它可行。😊

我们直接复制 Cilium 文档中给出的命令，使用 Kind 创建 K8s 集群，

```bash
curl -LO https://raw.githubusercontent.com/cilium/cilium/1.12.0/Documentation/gettingstarted/kind-config.yaml
kind create cluster --config=kind-config.yaml
```

稍微看一下用户创建集群的 yaml 文件配置，

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
```

你可以根据需要和机器性能增添或删减工作节点的数量，笔者在这里便使用官方给出的定义。在我们执行第二条命令后 kind 会使用 docker 拉取对应的镜像，但 kind 中并不会显示拉取的进度，你可以自行使用 `docker pull` 拉取对应版本的 kind 镜像并确认你的网络环境是否能够访问对应的仓库。

值得一提的是，在我们创建的 k8s 集群中，默认的 CNI 插件已经被 **disable**，我们在下一步中安装的 Cilium 会提供相应的 CNI 实现。

若一切的顺利执行，你的 K8s 集群便会被顺利创建，你可以使用 `kubectl cluster-info` 查看你的集群是否已成功部署，

{{< image src="https://image.p1nant0m.xyz/202209281510675.png" caption="kubectl cluster-info 命令执行情况" src_s="" src_l="https://image.p1nant0m.xyz/202209281510675.png" >}}

使用 `docker ps` 命令查看在本地机器上已经启动了的工作节点以及控制平面，

{{< image src="https://image.p1nant0m.xyz/202209281510146.png" caption="docker ps 命令查看运行的“节点”" src_s="https://image.p1nant0m.xyz/202209281510146.png" src_l="https://image.p1nant0m.xyz/202209281510146.png" >}}

**容器中代理服务的配置**

值得一提的是，受限于国内网络环境情况，许多镜像在拉取时会出现问题，这将导致 Kubernetes 部署一些 Pod 的时候会出现经常失败的情况。因此，我在 Systemd 启动 Docker 守护进程的相关配置中增加了相关的代理配置内容，具体的可以参考 [HTTP/HTTPS proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy) ，这样便可以在 Docker 容器内注入相关的环境变量，实现代理。

还有一种方式是通过 `kind load` 命令向相关容器中加载指定的镜像，但是这种方式在一些特殊的场景下比较麻烦，有兴趣的朋友可以试试。

{{< admonition type=warning title="代理的问题" open=false >}}

当你选择代理后，会出现非常多奇奇怪怪的问题，这在很多情况下是因为代理网络与集群网络的原因。

一个比较典型的例子是访问某些 Pod 容器中进程暴露的健康状态检测探针 REST API 接口。因为代理的原因，你很有可能访问不到相关资源的路径，从而导致超时，进而 k8s 会根据相关的策略去重启相关的服务，如此反复。

{{< /admonition >}}

#### 安装 Cilium CLI 终端并在集群中部署 Cilium

根据官方文档，我们下一步需要安装 Cilium 的 CLI 二进制程序，该程序用于提供自动在 k8s 集群中部署安装 Cilium 提供的 CNI 实现以及能够检查 Cilium 的安装状态，启停相关特性等功能。出于安全原因的考虑，你可以参考[官方文档](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)来完成此步骤的安装。

在 Cilium 部署相关组件时，执行相关命令的进程会阻塞等待 k8s 集群完成组件的部署，我们可以在另一个命令行终端中执行 `kubectl get pods --all-namespace -w` 来实时查看相关进度。

{{< admonition type=tip title="郁闷" open=false >}}

k8s 在相关节点拉取 YAML 文件中指定的镜像时并没有能够显示拉取进度的地方，有时候都不知道网络是不是正常的，只能等到它失败或是成功，心里没底，有点慌。

{{< /admonition >}}

一切都正常执行完毕后，你可以使用 `cilium status` 命令来查看集群中 Cilium 的状态，

{{< image src="https://image.p1nant0m.xyz/202209281537611.png" caption="cilium status 命令执行" src_s="https://image.p1nant0m.xyz/202209281537611.png" src_l="https://image.p1nant0m.xyz/202209281537611.png" >}}



#### 启用 Hubble UI

Hubble 是 Cilium 项目的可观测层，它可以用来获取集群范围内在网络以及安全层面的可观测性，从而为运维人员提供进行排障工作的信息依据。

执行 `cilium hubble enable --ui` 启用 **hubble UI** 特性，Cilium 将会通过 Patch cilium-confg、重启 Cilium Pods 、部署 Hubble UI 相关组件的方式来启用相关服务，最终部署执行完成后执行 `cilium hubble ui` 命令将相关服务映射到我们本地的端口上，访问相关页面，可以看到 hubble-ui 与服务健康检测探针之间通信的数据流。

{{< image src="https://image.p1nant0m.xyz/202209281620041.png" caption="Hubble UI 前端页面" src_s="https://image.p1nant0m.xyz/202209281620041.png" src_l="https://image.p1nant0m.xyz/202209281620041.png" >}}


#### 部署一些简单的 Workload （Star Wars Demo）并在 Hubble UI 中观察网络流

Cilium 官方文档中提供了一些示例 Demo 展示 Cilium 基于身份和 HTTP 的策略执行，其中一个 Demo 便是 Star Wars ⭐。根据 [Docs](https://docs.cilium.io/en/stable/gettingstarted/http/#deploy-the-demo-application) 里的示例，我们可以快速部署这样的 Demo Workload。官方给出的 Demo 应用拓扑结构图如下所示，

{{< image src="https://docs.cilium.io/en/stable/_images/cilium_http_gsg.png" caption="应用拓扑结构图" src_s="https://docs.cilium.io/en/stable/_images/cilium_http_gsg.png" src_l="https://docs.cilium.io/en/stable/_images/cilium_http_gsg.png" >}}

我们可以通过调用这几个服务的 REST API 接口来产生相应的网络流量，比如在 xwing Pod 中调用 `kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing` 向 `deathstar` 服务的 `/v1/request-landing` 接口发出降落请求，该服务会将该请求转发给相应的后端 Pod 处理。

同样，我们也可以在 tierfighter 中执行相似的命令，`kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing`，其效果与上一个例子相似。我们能够在 hubble-ui 的 default 命名空间下看到相关 Pod 的网络流量，如下所示，

{{< image src="https://image.p1nant0m.xyz/202209281648078.png" caption="default 命名空间下 Pods 间的网络流量" src_s="https://image.p1nant0m.xyz/202209281648078.png" src_l="https://image.p1nant0m.xyz/202209281648078.png" >}}

不过，这里有一个奇怪的地方，盟军的 xwing 战机竟然能够落到帝国的 deathstar 上！这里我们便引出本文后续的内容，使用 Cilium 对网络流量进行 Policy Enforement。

#### 使用 Cilium 对 Layer 3, 4, 7 流量进行控制

所有的 Pod 在 Cilium 中都表示为一个 Endpoint，我们在 Cilium Pod 上执行 `cilium` 工具去将集群中 Endpoints 罗列出来，
```bash
$ kubectl -n kube-system exec cilium-gvmxn -- cilium endpoint list
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                                                         IPv6   IPv4           STATUS
           ENFORCEMENT        ENFORCEMENT

858        Disabled           Disabled          1          k8s:node-role.kubernetes.io/control-plane                                                                 ready
                                                           k8s:node.kubernetes.io/exclude-from-external-load-balancers

                                                           reserved:host

889        Disabled           Disabled          26464      k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system                 10.244.0.60    ready
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=coredns

                                                           k8s:io.kubernetes.pod.namespace=kube-system

                                                           k8s:k8s-app=kube-dns

982        Disabled           Disabled          28376      k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system                 10.244.0.36    ready
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=hubble-relay

                                                           k8s:io.kubernetes.pod.namespace=kube-system

                                                           k8s:k8s-app=hubble-relay

1282       Disabled           Disabled          5963       k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system                 10.244.0.151   ready
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=hubble-ui

                                                           k8s:io.kubernetes.pod.namespace=kube-system

                                                           k8s:k8s-app=hubble-ui

1610       Disabled           Disabled          26464      k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system                 10.244.0.19    ready
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=coredns

                                                           k8s:io.kubernetes.pod.namespace=kube-system

                                                           k8s:k8s-app=kube-dns

1758       Disabled           Disabled          9328       k8s:app.kubernetes.io/name=xwing
                              10.244.0.95    ready
                                                           k8s:class=xwing

                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=default

                                                           k8s:io.kubernetes.pod.namespace=default

                                                           k8s:org=alliance

2090       Disabled           Disabled          21851      k8s:app.kubernetes.io/name=deathstar
                              10.244.0.72    ready
                                                           k8s:class=deathstar

                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=default

                                                           k8s:io.kubernetes.pod.namespace=default

                                                           k8s:org=empire

2319       Disabled           Disabled          4          reserved:health
                              10.244.0.209   ready
2349       Disabled           Disabled          21851      k8s:app.kubernetes.io/name=deathstar
                              10.244.0.169   ready
                                                           k8s:class=deathstar

                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=default

                                                           k8s:io.kubernetes.pod.namespace=default

                                                           k8s:org=empire

2579       Disabled           Disabled          13871      k8s:app.kubernetes.io/name=tiefighter
                              10.244.0.240   ready
                                                           k8s:class=tiefighter

                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default
                                                           k8s:io.cilium.k8s.policy.cluster=kind-kind

                                                           k8s:io.cilium.k8s.policy.serviceaccount=default

                                                           k8s:io.kubernetes.pod.namespace=default
```

可以看到上述 Pod 中并没有对 Ingress 或 Egress 向流量进行 Policy Enforcement，我们下一步就是要去创建对应的 Policy，实现我们对三层、四层以及七层流量的控制。

##### 应用 L3/L4 层网络安全策略

使用 Cilium 与 Kubernetes 部署 L4 网络策略，我们可以使用在创建 Pod 时为它们指定的标签来构建我们的安全策略，这样我们就不需要知道它们到底具体 Pod 运行在集群的哪一个网络位置在何时运行了。这提供了一层抽象，让用户只需要关心具体的业务逻辑，而不需要对底层的细节进行过多的干预或操心，Kubernetes 会为我们准备好一切！

例如，在上述我们用到的星战例子中，xwing Pod 中我们添加了标签 `class=xwing org=alliance`，而在 tierfighter Pod 中我们的标签则是 `class=tiefighter org=empire`，我们可以在编写安全策略时直接使用这些标签，Kubernetes 和 Cilium 会帮助我们找到对应的后端。

下图展示的是我们想要达成的目标，

{{< image src="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png" caption="xwing 战机不能降落在死星上！" src_s="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png" src_l="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png" >}}

为了达成我们的目标，我们可以通过编写 **CiliumNetworkPolicy** 实现，
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP

```
上述规则是一个白名单规则，其仅允许标签为 `org:empire` 的 Pod 通过 TCP 协议访问具有标签 `org:empire class:deathstar` Endpoint 的 80 端口，其它不符合规则的入向流量都会被丢弃。这样我们便实现了 L3/L4 的 Policy Enforcement.

在 Hubble UI 中，我们可以看到由 Pod xwing 发出的流量被 Pod deathstar 丢弃，

{{< image src="https://image.p1nant0m.xyz/202209281930575.png" caption="Policy Enforement 丢弃相关数据包" src_s="https://image.p1nant0m.xyz/202209281930575.png" src_l="https://image.p1nant0m.xyz/202209281930575.png" >}}

我们可以在此执行上面提到的查看 Cilium Endpoint 的命令，这时候我们发现 deathstar Pod 中 Ingress Policy Enforcement 被激活。我们也可以通过执行 `kubectl describe cnp` 来查看相关的 CiliumNetworkPolicy 规则。
```bash
kubectl describe cnp
Name:         rule1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2
Kind:         CiliumNetworkPolicy
Metadata:
  Creation Timestamp:  2022-09-28T11:27:31Z
  Generation:          1
  Managed Fields:
    API Version:  cilium.io/v2
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:description:
        f:endpointSelector:
          .:
          f:matchLabels:
            .:
            f:class:
            f:org:
        f:ingress:
    Manager:         kubectl-create
    Operation:       Update
    Time:            2022-09-28T11:27:31Z
  Resource Version:  21081
  UID:               5c405d24-b059-4992-8250-f9963a42858b
Spec:
  Description:  L3-L4 policy to restrict deathstar access to empire ships only
  Endpoint Selector:
    Match Labels:
      Class:  deathstar
      Org:    empire
  Ingress:
    From Endpoints:
      Match Labels:
        Org:  empire
    To Ports:
      Ports:
        Port:      80
        Protocol:  TCP
Events:            <none>
```

##### 应用 L7 层网络安全策略

有些时候我们需要更加细粒度的规则去进行过滤，L3/L4 层中的语义相对来说更加偏向底层且与应用本身无关，而 L7 层的报文信息具有更加丰富的语义并且与应用相关，对 L7 层进行安全访问控制能够取得更好的防护效果、同时也能够针对不同的应用、不同的使用场景来进行个性化需求定制，给用户更加丰富多样的选择。

回到我们在 Star War 中的例子，假设现在 deathstar 中有一个 api `/v1/exhaust-port`，它能够让死星爆炸💣！对其进行访问需要进行严格的访问控制，并不是任何帝国飞船都能够发出请求并成功响应的！如果不对其进行任何限制，可能会有下面的悲剧发生（盟军可要高兴坏了！），
```bash
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```

通过 **CiliumNetworkPolicy**， 我们最终需要达成下图所示的目标，

{{< image src="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_l7_gsg.png" caption="安全策略的目标" src_s="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_l7_gsg.png" src_l="https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_l7_gsg.png" >}}

我们可以通过下述策略进行 L7 层的访问控制，
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

上述策略对带有 `org:empire` 标签的 Pod 能对 `org:empire class:deathstar` 的 Pod 进行的操作做出了限制，符合目标的 Pod 只能够通过 HTTP POST 方法访问目的 Pod （Deathstar）的 `/v1/request-landing` 接口。

我们再次访问 Service 的 ``/v1/request-landing`` 接口，可以看到我们这次的请求被拒绝了，
```bash
$ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

这便是简单使用 Cilium 进行 L3/L4、L7 层访问控制的示例，以及我们能够从 Hubble UI 中获得相关的数据信息。

### 总结

Cilium 使用 eBPF 技术实现 CNI 网络插件，并且利用 eBPF 在安全方面的能力，实现用户自定义策略的 Policy Enforcement，这让人们看到 eBPF 在网络与安全方面的巨大潜能。不过，Cilium 项目本身还有许多可以改进的地方。

比如我们在上述的 Hubble UI 中看到的数据维度还是相对来说较为单一的，只能看见数据包的端到端的流向而无法看到其中的载荷信息，也不能通过前端运维面板或 dashboard 的形式实时下发策略。另外，在安全策略编写方面的可扩展性和语义仍较为简单，编写具有丰富安全上下文信息的安全策略受限（可以看看 OPA，或者cel-go？）。

笔者认为在云原生环境下，架构的复杂性以及通信的流量的纵横交错本身就会给系统的观测和维护带来挑战，但于此同时也会带来巨大的上下文环境语义信息，业界对于可观测性也有比较好的解决方案（Tracing、Logging、Metric），安全作为系统的重要一环，本身就可以受益于可观测性提升带来的可供分析的信息量增加。强语义、智能的 Policy Enforcement 结合 HIDS、IAST、RASP 等前沿安全技术的应用，能够解决大多数云原生应用场景下的运行安全问题，能够给运维工程师、安全工程师提供充分、精确的数据去分析相关的安全事件，从而能够尽早进行研判、封堵，形成安全事件闭环。

平台层应用除了提供数据的采集、聚合/聚类、关联、可视化、持久化之外，还需要提供主动的手段，对相关行为进行反制。譬如对实时的入侵者进行热动态蜜罐迁移、身份实体封禁，化被动安全为主动安全。

{{< admonition type=note title="猜想" open=false >}}

eBPF 与网络相关的程序中有 XDP 和 TC 类别，TC 程序能够访问内核的路由表，从而决定数据包发出的方向，使用 eBPF 技术可以实现 LoadBalancer，对 L3/L4 层流量进行负载均衡，这是在分布式系统中架构中的使用。

在安全方案，这种动态低成本的网络“改道”能力似乎很适合用作蜜罐，云原生环境下可以快速部署一个容器环境，并将被标记的相关流量牵引于此，由于是蜜罐，我们可以在里面加载尽可能多的探针，尽可能精确的捕获攻击者的行为图谱，从而发现业务代码的漏洞，并对其进行及时的修复和处理。

{{< /admonition >}}
