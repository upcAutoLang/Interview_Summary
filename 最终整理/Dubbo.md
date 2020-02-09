需要理解的内容：负载均衡、熔断、注册中心、稳定性（宕机、突发高流量）、决策、架构关系、架构分层；

# 一. Dubbo 架构

> 参考地址：[《dubbo系列三、架构介绍及各模块关系》](https://www.cnblogs.com/wangzhuxing/p/9725096.html)

Dubbo 是阿里服务化治理方案的核心框架，是一种分布式 RPC 框架，它提供了**注册中心机制**，解耦了消费方与服务方动态发现的问题。

## 1.1 Dubbo 架构

Dubbo 的架构主要分为四部分：**服务提供方、服务消费者、注册中心、统计中心**，这四部分也被称为**部署逻辑拓扑节点**。Dubbo 运作模式如下：

1. **注册**：**服务提供方 (Provider)** 启动时，把自己的元数据向**注册中心 (Registry)** 上面注册；
2. **订阅**：**服务消费者 (Consumer)**启动时，从**注册中心 (Registry)** 订阅服务提供方的元数据；
3. **通知**：注册中心的数据发生变更，变更的数据会推送给订阅的服务消费者；
4. **调用**：获取服务提供方的元数据后，消费者可以发起 RPC 调用；
5. **监控**：RPC 调用发生前后，会向**监控中心 (Monitor)** 上报统计信息；

![Dubbo 架构](./pic/Dubbo架构.jpg)

Dubbo 的架构如上图所示，上图图例说明：

- 图中小方块 **Protocol, Cluster, Proxy, Service, Container, Registry, Monitor** 代表**层**或**模块**，<font color=blue>**蓝色**</font>的表示与业务有交互，<font color=green>**绿色**</font>的表示只对 Dubbo 内部交互；
- 图中**背景方块 Consumer, Provider, Registry, Monitor** 代表 Dubbo 架构的四个部分；
- 图中**<font color=blue>蓝色虚线</font>**为初始化时调用，<font color=red>**红色虚线**</font>为运行时异步调用，<font color=red>**红色实线**</font>为运行时同步调用；
- 注：图中只包含 RPC 的层，不包含 Remoting 的层，Remoting 整体都隐含在 Protocol 中；关于分层见下一节的图示。

## 1.2 Dubbo 分层

![Dubbo 分层](./pic/Dubbo分层.png)

上图为 **Dubbo 的分层结构**。横向将其分为两部分，左边淡蓝色背景是**消费者**使用的接口，右边淡绿色背景是**提供者**使用的接口，位于两者中轴线上的是消费者与提供者共同使用的接口。  
纵向可以将 Dubbo 分为十层。如果按照**总体功能**，可以想左侧的分类一样，将 Dubbo 分为三层：

- **业务层 (Business)**：Business
- **RPC 层**：Config, Proxy, Registry, Cluster, Monitor, Protocol
- **远程调用层 (Remote)**：Exchange, Transport, Serialize

如果按照**调用方**的角度：

- **API 层**：**Service, Config**，供用户调用；
- **SPI 层**：除 API 层其他所有层，这些层全部可扩展的主要提供给扩展者使用，这也是 Dubbo 扩展能力强的原因。  

各层及各层的核心组件如下：

|层次名|作用|
|:--|:--|
|业务层 (Service)|业务代码的接口与实现，即开发者实现的业务代码|
|配置层 (Config)|对外配置接口，以用来**暴露服务配置的 ServiceConfig** 与**引用的服务配置 ReferenceConfig** 为中心|
|服务代理层 (Proxy)|无论是服务提供者还是消费者，Dubbo 框架都会生成一个代理类，整个过程对上面是透明的，调用远程接口就像调用本地方法一样。核心为 **ServiceProxy**|
|注册层 (Registry)|负责 Dubbo 框架的**服务注册与发现**。集群中有服务上下线时，注册中心通知订阅该服务的订阅方相应的动作|
|集群容错层 (Cluster)|又称路由层，负责远程调用失败的**容错**策略，以及选择调用节点的**负载均衡**策略|
|监控层 (Monitor)|监控 RPC 调用次数、调用时间|
|远程调用层 (Protocol)|封装 **RPC 调用的具体过程**。Dubbo 以 **Invoker 为核心模型**，框架中其他所有模型向它靠拢，可以是一个本地、远程、或者集群的实现|
|信息交换层 (Exchange)|封装请求响应模式|
|网络传输层 (Transport)|把网络传输抽象成统一的接口，以 Mina, Netty 为统一接口，以 **Message** 为中心|
|序列化层 (Serialize)|负责管理整个框架网络传输时的序列化、反序列化工作|

## 1.3 Dubbo 架构关系

首先在 Dubbo 架构图中的服务提供者 (Provider) 与服务消费者 (Consumer) 都是抽象的、相对性的概念，类似于我们平常说的服务器、客户端结构。这样的概念主要是为了让读者更加直观的了解哪些类属于客户端与服务端。  
在 RPC 层中，**Protocol 是核心层**。只要有 Protocol + Invoker + Exporter，就可以完成非透明的 RPC 远程调用。

在**远程调用层 Remote** 中，基本是 **Dubbo 协议的实现**。Remote 层内部再划分为传输层 Transport 与信息交换层 Exchange，传输层只负责消息的单向传输，是对不同协议 Mina, Netty, Grizzly 的抽象，同时也可以扩展接口；而Exchange 层在 Transport 层之上封装了**请求-响应**的模型。

> 该部分需要进一步深入理解。

# 二. 注册中心
第三章

首先需要说明的一点是，服务的暴露与注册是两个不同的概念。在Dubbo中，微服务之间的交互默认是通过Netty进行的，而服务之间的通信是基于TCP以全双工的方式进行的。那么也就是说，每个服务都会存在一个ip和port。所谓的服务暴露就是指根据配置将当前服务使用Netty绑定一个本地的端口号(对于消费者而言，则是尝试连接目标服务的ip和端口)。至于注册，由于微服务架构中对于新添加的服务，需要一定的机制来通知消费者，有新的服务可用，或者对于某些下线的服务，也需要通知消费者，将这个已经下线的服务给移除。Dubbo中服务的注册与发现默认是委托给zookeeper来进行的。

暴露：书 5.2.2 章，文章[《Dubbo服务暴露与注册》](https://zhuanlan.zhihu.com/p/87075790)


> 参考地址：[《阿里dubbo服务注册原理解析》](https://www.cnblogs.com/linlinismine/p/7814521.html)  
>
> 详细调用过程：[《dubbo服务注册与发现、服务调用过程》](https://www.jianshu.com/p/1ff25f65587c)

ServiceBean#afterSetProperties 中调用了 <code>**com.alibaba.dubbo.config.ServiceConfig#export**</code>方法，主要代码如下：

```java
//如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露本地服务)
if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
    if (logger.isInfoEnabled()) {
        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
    }
    if (registryURLs != null && registryURLs.size() > 0
            && url.getParameter("register", true)) {
        for (URL registryURL : registryURLs) {
            url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
            URL monitorUrl = loadMonitor(registryURL);
            if (monitorUrl != null) {
                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
            }
            if (logger.isInfoEnabled()) {
                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
            }
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
            Exporter<?> exporter = protocol.export(invoker);
            exporters.add(exporter);
        }
    } else {
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        Exporter<?> exporter = protocol.export(invoker);
        exporters.add(exporter);
    }
}
```

整个服务的暴露过程如下图所示：

![img](https://images2017.cnblogs.com/blog/905730/201711/905730-20171110175055856-2054003383.png)

然后再补上一消费者调用提供者的图：

![img](https://images2017.cnblogs.com/blog/905730/201711/905730-20171110175212669-2102453811.png)

下面给出具体某一种协议的实现，假设配的协议是dubbo，注册中心用的是zookeepr。那么代码的调用过程大致是这样：

1. 调用 JavassistProxyFactory 生成一个Invoker；

2. 用com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#export 方法，进行一个服务的暴露；

3. DubboProtocol#export 方法最终会调用 com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory#createRegistry；进行一个服务注册；

这时 zookeeper 相应的目录下面就会有对应的内容，这时服务注册就算完成了。

> 注：本业务组实现的框架中，服务注册与发现方式见[《项目经历—注册中心》篇](./项目经历/注册中心.md)

# 三. Dubbo 扩展点

第四章、第八章

# 四. 启动与服务暴露

第五章

暴露：书 5.2.2 章，文章[《Dubbo服务暴露与注册》](https://zhuanlan.zhihu.com/p/87075790)

# 五. 远程调用

第六章

# 六. 负载均衡
第七章

# 七. 容错
第七章

# 八. 稳定性

（宕机、突发高流量）  
第七章，网上博客

# 九. Dubbo 过滤器

# 十. 决策
