---
layout: post
title: 初探cilium---容器网络的福音
categories: Blog
keywords: cilium, Network
---

## 背景       
Clilium不是一个独立的项目，要说cilium就不得不提到k8s，k8s是一个优秀的容器编排开源项目，它提供了简洁的容器部署，规划，更新，维护的机制，但对于容器间的网络通信则显得较为无力，k8s本身并没有给出一个具体的网络通信逻辑，而是指定了一套CNI（countainer network interface）标准交由社区开发，我们来看下容器间通信有哪些情况：
- 1.需要通信的两个container位于同一个pod内：这种情况较为简单，k8s pod内部容器是共享网络空间的，所以容器直接可以使用localhost访问其他主机。
- 2.需要通信的两个container不同pod内，pod位于同一个node下：这种情况也不复杂，这属于典型的docker内通信，一般采用docker的虚拟网桥模式。
- 3.需要通信的两个container不同pod内，但pod位于不同node下：这就要麻烦很多了，现在用的比较多的解决方案就是coreOS开发的flannel,flannel在网络层实现了一个简单的overlay网络来保证容器间的通信，其大致通信流程如下：
![flannel架构图.jpg|center|400x120](https://github.com/william-zk/william-zk.github.io/blob/main/images/self-drawn/flannel.png)

## 思考问题
- 1.假如我们现在k8s集群中有两个服务service1和service2，我们要求service1只负责接收service2的数据包，我们如何实现这个功能呢
- 2.k8s为每个pod都设定了副本，以保证pod的安全扩容与缩容，这也就意味着同时会有多个副本存在于集群中，那么如何进行相同pod间的负载均衡呢？
- 3.我们能不能实现更细粒度的过滤规则呢？比如限定service1只接受server2的http请求中的get请求。
- 4.如何定位容器间通信过程中出现的问题和故障呢？

## 解决以上问题的cilium 
上诉四个问题代表了通信网络中的典型问题：安全，负载均衡，故障定位以及数据监控，我们来分析下k8s中这些问题的处理现状以及看看cilium是怎么解决这些问题的。
### 问题一
在flannel网络下我们只能通过iptables设置过滤规则来实现，而iptables是通过ip+port的模式来过滤数据包，在一个k8s集群中，容器的部署和删除都是相当频繁的操作，这可能导致个别IP地址的使用快速变化。一个IP地址可能被一个容器使用几秒钟，然后在几秒钟之后会换到一个容器使用。这就造成了整个集群的网络性能瓶颈，因为集群中的所有节点都必须始终知道最新的IP到容器的映射。
而在cilium中，采用服务标识和API协议来标识身份，身份信息在同一个服务的多个副本之间是一致的，并且在一段时间内是一致的，这意味着无论服务的ip地址如何变化，一个服务的身份信息是不会变的。Cilium将负载和身份信息在每个包内都打包在一起（而不是依靠源IP地址），提供高可扩展安全性。这种设计使得身份可以被嵌入任何基于IP的协议，而且与未来的SPIFFEE或者Kubernetes的Container Identity Working Group兼容。
### 问题二
flannel仅仅只是保证了跨node间的容器通信，并没有对这个问题做任何优化。k8s在kube-proxy组件中通过iptables实现了简单的负载均衡，当service数量激增时，需要添加多条iptables规则。对于添加到Kubernetes的每个服务，要遍历的iptables规则列表呈指数增长。最近的KubeCon议题中详细研究kube-proxy的性能表现细节。研究结果显示，随着服务数量的增长，网络延迟和性能下降无法估量。还披露了iptables的另一个主要缺点，无法实现增量更新。每次添加新规则时，必须更新整个规则列表。结果是对2万个Kubernetes服务16万条的iptables规则，装配这些规则需要耗时5个小时。
由于BPF的灵活性，clilium通过hash算法实现了一个简化的BPF map,达到了O(1)的平均运行时间复杂度，这意味着无论服务数量如何，查找延迟是不变的。同样，从用户空间更新这些BPF map是非常高效的，这意味着即使有20,000多个服务，更新转发规则的时间也是微秒，而不是几小时。负载均衡器可以用两种方法配置实现：
- （1）Kubernetes服务实现：所有Kubernetes集群IP服务会自动在BPF中实现，为kube-proxy在集群间提供负载均衡提供了一种高可扩展性选择。
- （2） 基于API驱动：对更超前的使用场景，扩展式API可以用来直接配置负载均衡模块       
### 问题三
这个功能仅靠iptables和flannel是无法实现的，iptables仅仅只能指定一个ip地址或者端口将其数据全部丢弃，要实现这样的功能可能就需要一个k8s的node中存在一个类似nginx的代理服务，然后在代理服务中编写相应的规则，这个过程无疑是复杂的。       
而cilium在解决问题一的同时就几乎给出了问题三的回答，cilium基于服务和API标识的思想，为网络提供了更加细粒度的安全规则，你可以通过几行代码就实现一个高级的过滤规则，比如我们想实现允许label为 env:prod的终端通过GET方式请求/public，那么可以这么编写： 
![cilium使用.jpg|center|300x90](https://github.com/william-zk/william-zk.github.io/blob/main/images/self-drawn/cilium_use.png)              

总的来说，cilium不仅提供了基于服务和标签的过滤方式，还提供了基于API协议的过滤方式，目前cilium已经支持了REST/HTTP, gRPC and Kafka等使用较为广泛的通信协议的过滤支持。当然，在这两种过滤方式都不适用的情况下，cilium依然提供了基于IP/CIDR安全方式控制安全访问。但Cilium还是建议在配置安全策略时尽量采用抽象的方式，避免写入具体IP地址。       

### 问题四
对于这个问题，仅靠flannel+iptable的模式是无法做到的，因为网络层给出的消息仅有IP地址，而在发现故障时，容器的IP地址往往已经发生了变化，这就意味着我们可能几乎没有任何有效信息去定位故障。       
Cilium则针对此问题实现了一个强大的数据的监控与问题定位平台，它提供了以下功能：       
- （1）对元数据的事件监控：当一个包被丢弃时，clilium的监控模块不仅会报告包的源ip和目的IP，还会提供许多其他信息，比如container/pod的label和服务名。       
- （2）集群层面提供提供安全和转发事件的可视化，同时周期性监控集群连接状态，包括节点之间延迟，判断节点失效和底层网络问题       
- （3）基于BPF高性能监控：clilium设置了一个高性能BPF性能循环缓冲区(perf ring buffer)，可以追踪每秒百万级的应用事件，提供集成BPF可编程性的高效信道，允许数据可视化同时增加最小额外负载。       
- （4）允许通过Prometheus导出监控数据，可以与自定义的监控平台集成.   

### beyond problem
在解决了以上问题的同时，cilium还实现网络模型的简化，得益于将安全策略从地址层解耦，cilium可以实现更加清晰简洁的网络拓扑，在网络层cilium负责提供链接，而在顶层通过自定义策略实现对网络的分段，这样的网络模型十分有益于故障定位以及水平扩容，cilium可以将网络配置成以下两种模式：       
- （1）overlay/VXLAN模式：这种模式是最简单的集成模式，该模式允许在任何基于IP协议的数据包中携带workload的身份标识       
- （2）直接路由模式：该模式允许你讲路由功能集成到自己的网络组件当中去，如本机Linux路由层，IPVLAN或云路由器。

## 总结       
在cilium的发布视频上，提到最高频的词汇就是BPF，BPF的高效性已是总所周知的，cilium, 就是BPF在容器领域的锋芒初现，目前，clilium已经提供了强大而高效的网络策略，安全策略，以及3-7层的负载均衡策略，现在cilium社区已经有大量人关注并投入开发，相信它未来会在容器网络领域给我们更惊艳的表现。