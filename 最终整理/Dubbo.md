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

ServiceBean#afterSetProperties 中调用了 <code>com.alibaba.dubbo.config.ServiceConfig#export</code>方法，主要代码如下：

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

# 五. 集群容错

在客户端已经从注册中心拉取和订阅服务列表完毕的前提下，Dubbo 完成一次完整的 RPC 调用，流程如下：

1. **服务列表聚合**；
2. **路由**；
3. **负载均衡**；
4. 选择一台机器进行 RPC 调用；
5. 请求交给底层 I/O 线程池处理；
6. 读写、序列化、反序列化；
7. 方法调用；

将上面的步骤进行细化，在一次 RPC 调用过程中，Cluster 层的流程如下：

1. 根据不同的**容错机制**，生成 Invoker 对象，调用 AbstractClusterInvoker 的 Invoker 方法；
2. 获得可调用的服务列表；
3. 使用 Router 接口处理服务列表，根据**路由**规则过滤一部分服务；
4. **负载均衡**；
5. **RPC 调用**；

其中步骤 1, 2, 3 是模板方法，使用通用的校验、参数准备等准备工作。最终，不同的容错机制的子类实现不同的 <code>doInvoke</code> 方法，每个子类方法都有各自的路由、负载均衡实现策略。

本章节主要总结 RPC 在 Cluster 层的工作，涉及步骤 1, 2, 3, 4，其中容错机制见[5.1](##5.1 容错机制)，容错过程中获取 Invoker 列表需要用到 Directory，见[5.2](##5.2 Directory)；Directory 过程中需要用到路由，见[5.3](##5.3 路由)；负载均衡见[5.4](##5.4 负载均衡)。剩余步骤 5, 6, 7 是具体的 RPC 调用，见[第六章](#6. 远程调用)。

## 5.1 容错机制

容错过程是在各容错机制实现子类的 <code>doInvoke</code> 方法重写实现的。容错过程对上层用户是完全透明的，上层用户不用关心容错过程是怎么实现的，同时用户也可以通过不同的配置项来选择不同的容错机制。支持的容错机制如下：

> 注：  
> 大部分容错机制的核心步骤都是：
> 
> 1. 校验；
> 2. 获取配置参数；
> 3. 实现各自容错机制的调用；
> 
> 在上述步骤 3 容错机制的调用中，主要步骤都是：
> 
> 1. 校验；
> 2. 负载均衡；
> 3. RPC 调用；
> 
> 如果有不同，在各自条目中进行说明

1. **Failover**：重试失败，默认策略
	- 调用失败，尝试调用其他服务器；
	- 根据配置的重试次数，进行重试；如果有成功，则返回；全部重试失败之后，抛出异常；
2. **Failfast**：快速失败
	- RPC 调用失败后，将异常封装为 <code>RpcException</code>，抛出并返回，不做任何重试；
3. **Failsafe**：安全失败
	- 出现异常时忽略；
4. **Failback**：定时重试失败
	- 调用失败后，将该失败的 <code>invocation</code> 缓存到 <code>ConcurrentHashMap</code> 中，并返回空结果集；同时设置定时线程池，定时时间到了就将失败的任务投入线程池，重新请求；
	- 如果重新请求成功，则从缓存中移除，请求失败则判断失败次数；如果失败次数少于设定的阈值，则重新投入定时线程池；如果多于设定的阈值，打印错误并放弃该请求；
	- 定时重试失败的实现思路，可以用于 **Kafka 的重试队列**；
5. **Forking**：并行
	- 根据设定的并行数量，循环执行负载均衡，筛选出可调用的 Invoker 列表；
	- 循环使用线程池，同时调用多个相同的服务；多个服务中，只要其中一个返回，就立即返回结果；所有线程调用失败，则抛出异常；
		- 该部分的实现是通过阻塞队列 <code>BlockingQueue</code> 实现的；将多个调用任务投入线程池后，任务执行结果投入 <code>BlockingQueue</code>；
		- 如果任务执行结果是异常类型，投入 <code>BlockingQueue</code> 抛出异常；此时记录异常次数，只有到**记录异常次数等于服务数量**时，说明所有服务都抛出异常，此时再将异常信息投入 <code>BlockingQueue</code>
		- 调用任务投入线程池之后，就立即调用 <code>BlockingQueue # poll(int)</code> 方法拉取结果，拉取到第一个结果就返回。如果返回值正常，就是其中一个服务的返回结果；如果返回值为 <code>Exception</code> 类型，说明所有服务都出现异常；
6. **Broadcast**：广播
	- 广播调用所有可用服务，循环遍历所有 Invoker，每个 Invoker 分别做 RPC 调用；
	- 如果有任意一个节点报错，等待广播最后完成之后抛出；如果多个节点异常，最后一个节点抛出的异常会覆盖前面抛出的异常；
7. **Available**：可用
	- 最简单的方式，请求**不会做负载均衡**，遍历所有服务列表，找到第一个可用节点，直接请求并返回结果；
8. **Mock**：仿真
	- 调用失败时返回伪造的响应结果，或者直接强行返回伪造结果；
9. **Mergeable**：合并：将多个节点请求的结果合并；

## 5.2 Directory

容错过程中需要获取 Invoker 列表，用于后续的路由和负载均衡。这个过程需要用到 <code>Directory # list</code> 方法执行。Directory 接口有一个**抽象类 AbstractDirectory**，以及两个主要实现类：**动态列表 RegistryDirectory**，以及静态列表 StaticDirectory。主要总结的是动态列表 <code>RegistryDirectory</code>，以及封装了基础方法的抽象类 <code>AbstractDirectory</code>。  
<code>RegistryDirectory</code> 主要实现了两个功能：

1. 与注册中心的**订阅，动态更新**本地的 Invoker 列表；
2. 实现父类的 <code>doList</code> 方法；

### 5.2.1 订阅与动态更新

注册中心订阅的部分主要在 <code>ZookeeperRegistry # doSubscribe()</code> 方法中实现，见[第二章注册中心](#二. 注册中心)部分。  
在监听到注册中心对应 URL 变化后，触发 <code>RegistryDirectory</code> 对各种本地配置的**动态更新**。更新的配置包括：

1. **路由信息**：通过路由工厂 <code>RouterFactory</code> 将 URL 包装成路由规则（见[5.3](#5.3 路由)），更新本地路由信息；
	- 更新路由规则，是通过 override 协议实现的；
2. **服务提供者配置 Configurator**：管理员可以在 dubbo-admin 下动态修改生产者的参数，这些参数会保存在配置中心的 configurators 类目录下；
3. **Invoker 修改**：如果监听到的 Invoker 类型 URL 不为空，则将新的 URL 与本地旧 URL 合并，同时销毁旧 Invoker；

### 5.2.2 doList

<code>doList</code> 方法主要作用，就是调用路由方法。

## 5.3 路由

> 注：路由的整体思路与笔者设计的动态汇总统计业务不谋而合，通过表达式的方式实现数据的处理。

**路由**会根据用户配置的不同路由策略，对 Invoker 列表进行过滤。主要分为**条件路由**、**文本路由**、**脚本路由**。路由工厂 <code>RouterFactory</code> 是一个 SPI 接口，用户可以自行通过实现 <code>Router</code> 接口扩展 Router 类；在调用的时候，在 URL 的 <code>protocol</code> 参数中可以设置 **file / script / condition**，分别寻找对应的实现类。

### 5.3.1 条件路由 (ConditionRouter)

条件路由使用的是 <code>condition://协议</code>，URL 形式是：<code>"condition://0.0.0.0/com.foo.DemoService?category=routers&dynamic=false&rule=" + URL.encode("host = 10.20.153.10 => host = 10.20.153.11")</code>；每个参数都是有含义的：

|参数名|含义|
|:--|:--|
|condition://|路由类型为条件路由（可扩展）|
|0.0.0.0|对全部 IP 生效，填入具体 IP，则只对该 IP 生效|
|com.foo.DemoService|对指定服务生效，必填|
|category=routers|当前设置指该数据为动态配置类型，必填|
|dynamic=false|当前设置表示该数据为持久数据，必填|
|enable=true|覆盖规则生效，默认生效|
|force=false|路由结果为空时，是否强制执行，默认为 false，路由为空时将自动失效|
|rule=...|路由规则内容，必填|

条件路由最关键的部分在于 **rule** 的路由规则。以下面的路由规则为例：

```java
method = find* => host = 192.168.1.22
```

1. 该路由规则的意义：所有调用 <code>find</code> 开头的方法，都会被路由到 192.168.1.22 的服务节点上；
2. <code>=></code> 之前部分是**服务消费者匹配条件**；
	- 如果匹配条件为空，则表示应用于所有消费者；
3. <code>=></code> 之后部分是**服务提供者列表的过滤条件**；
	- 如果过滤条件为空，则表示禁止访问；
4. 表示规则的表达式支持 <code>$protocol</code> 等**占位符**方式，也支持 <code>=, !=</code> 等条件，也支持**通配符** <code>*</code>。

条件路由的具体实现类是 <code>ConditionRouter</code>，整体的思想是通过正则表达式，按照 <code>=></code>进行分割，然后对符号前后的内容进行正则表达式的匹配，匹配结果存入对象 <code>MatchPair</code> 中。对于上述的**占位符、通配符**等，<code>MatchPair</code> 会进行匹配解析。

> 注：条件路由的整体思路，类似于笔者设计的动态汇总统计业务。

### 5.3.2 文件路由 (FileRouter)

文件路由通常和脚本路由搭配使用。文件路由将规则写到文件中，文件中写的是自定义的脚本规则，脚本可以是 Javascript, Groovy 等，文件路由 <code>FileRouter</code> 找到对应文件，将文件中的脚本内容按照类型匹配脚本路由，执行解析。

### 5.3.3 脚本路由 (ScriptRouter)

**脚本路由**使用 JDK 自带的脚本解析器，对脚本解析并运行，默认使用 Javascript 解析器。在构造脚本路由时初始化脚本执行引擎，根据脚本不同的类型，通过 JDK 提供的 <code>ScriptEngineManager</code> 创建不同的脚本执行器。接收到脚本内容后，执行 route 方法。具体的过滤逻辑需要用户自行定义。

> 注：在笔者设计的动态汇总统计业务中，笔者使用了 Aviator 表达式引擎，它与脚本路由中的脚本执行器 <code>ScriptEngineManager</code> 类似。

## 5.4 负载均衡

很多容错策略在路由选择出所有可用 Invoker 列表中实行最后一步筛选，负载均衡。  
负载均衡的核心是 <code>LoadBalance</code> 接口及其子类具体实现的，但并不是直接使用 <code>LoadBalance</code> 方法。在容错策略中的负载均衡先使用了抽象父类 <code>AbstractClusterInvoker</code> 中定义的 <code>Invoker select</code> 方法，它在 <code>LoadBalance</code> 基础上又封装了一些特性：

1. **粘滞连接**：尽可能让客户端总是向同一提供者发起调用。
	- 类似的策略，也在 Kafka 再均衡策略 StickyAssignor 中用过；
2. **可用检测**；
3. **避免重复调用**；

<code>select</code> 方法也使用了模板模式，在 <code>select</code> 方法中处理通用逻辑，最后提供 <code>doSelect</code> 抽象方法供各子类具体实现。Dubbo 内置了四种负载均衡算法，此外由于 <code>LoadBalance</code> 接口带有 @SPI 注解，所以用户也可以自行扩展负载均衡算法。在调用方法时我们可以在 URL 中通过 <code>loadbalance=xxx</code> 动态指定 select 方法的负载均衡算法。

### 5.4.1 Random

根据权重，设置随机概率做负载均衡。

### 5.4.2 RoundRobin

见[《Nginx》篇 6.2.2](./Nginx.md)。

### 5.4.3 LeastActive

LeastActive 就是**最少活跃调用负载均衡**，Dubbo 在运行过程中会统计每一次 Invoker 的调用，每次从活跃数最少的 Invoker 中选一个节点。

### 5.4.4 一致性 Hash

一致性 Hash 的原理见[《数据结构与算法》篇第五章](./数据结构与算法.md)。

Dubbo 的一致性 Hash 负载均衡，将**接口名 + 方法名**作为 Key 值，类型为 <code>ConsistentHashSelector</code> 实例对象作为 Value 存入一个 ConcurrentHashMap 中。每次请求进入，解析请求获取到方法，将该方法转为 Key 值，找到对应的 <code>ConsistentHashSelector</code> 进行负载均衡。所以 <code>ConsistentHashSelector</code> 是 Dubbo 中一致性 Hash 实现的核心。  
<code>ConsistentHashSelector</code> 的环形散列是用 **TreeMap** 实现的，所有真实节点、虚拟节点都放在 TreeMap 中。将**节点的 IP + 递增数字**，然后作 MD5 计算，最后进行 Hash 计算，作为 TreeMap 的 Key 值。TreeMap 的 Value 值为对应的某个可以调用的节点。关键代码如下：

```java
    // 遍历所有节点
    for (Invoker<T> invoker : invokers) {
        // 得到每个节点的 IP
        String address = invoker.getUrl().getAddress();
        // replicaNumber 是生成的虚拟节点数量，默认 160 个
        for (int i = 0; i < replicaNumber / 4; i++) {
            // 对 IP + 递增数字作 MD5 计算，作为节点标识
            byte[] digest = md5(address + i);
            for (int h = 0; h < 4; h++) {
                // 对标识作 Hash 计算，作为 TreeMap 的 Key 值
                long m = hash(digest, h);
                // 当前 Invoker 为 Value
                virtualInvokers.put(m, invoker);
            }
        }
    }
```

每次请求进来后，进行上述的 Key 值运算，每次请求的参数都不同，但是由于 TreeMap 是有序的树形结构，所以可以调用 <code>TreeMap#ceilingEntry</code> 方法，找到最近一个大于或等于给定 Key 值的节点 Entry。这样的操作相当于**一致性 Hash 算法的顺时针向前查找**的效果。

# 六. 远程调用
第六章

## 6.1 Dubbo 协议

## 6.2 编解码器

## 6.3 业务处理：ChannelHandler


# 七. 稳定性

（宕机、突发高流量）  
Sentinal？网上博客

# 八. Dubbo 过滤器

# 九. 决策
