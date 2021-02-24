## skywalking简介

### skywalking是什么

SkyWalking is an Observability Analysis Platform and Application Performance Management system.

Provide distributed tracing, service mesh telemetry analysis, metric aggregation and visualization all-in-one solution.

GITEE:

SkyWalking 是一款开源的应用**性能监控**系统，包括指标监控，分布式追踪，分布式系统性能诊断



![Cloud Native Computing Foundation](http://skywalking.apache.org/images/skywalking-arch-3800x1600.jpg)

### skywalking包含组件

![架构图](http://static.iocoder.cn/2d559e500ea828b2922ea75768d576a7)



### 分布式调用链标准 - OpenTracing

![img](https://img-blog.csdnimg.cn/img_convert/b0c329e7f9f936ec63b63adc8358627a.png)

OpenTracing 通过提供平台无关，厂商无关的 API，使得开发人员能够方便地添加追踪系统的实现.

#### 数据模型

- **Trace**：一个完整请求链路
- **Span**：一次调用过程(需要有开始时间和结束时间)
- **SpanContext**：Trace 的全局上下文信息, 如里面有traceId

![img](https://img-blog.csdnimg.cn/img_convert/5a01a30504a06e75ca0bbdbeee47e943.png)



### javaagent

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321234650938.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNDk3MzYx,size_16,color_FFFFFF,t_70)

![image-20210114101700204](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114101700204.png)

· 必须 META-INF/MANIFEST.MF中指定Premain-Class 设定启agent启动类

打成jar包

![image-20210114101755586](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114101755586.png)

# 插件体系

## 样例：流控中

1. 在flowcontrol-sdk-pulgin下层新建一个module

![image-20210112215210026](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210112215210026.png)

2. 在java目录下面新建Instrumentation和Interceptor目录
3. 两个java都存在模板样例
4. resources目录下新建skywalking-plugin.def文件，插件名称与自己编写的java名称对应即可
5. done



指南：http://www.itmuch.com/books/skywalking/guides/Java-Plugin-Development-Guide.html



![img](https://img2018.cnblogs.com/blog/730202/201908/730202-20190815190840752-2032735944.png)

# 个人解析

skywalking大致分为两块

+ Agent
+ console

## Agent

agent里面总共有3个主要功能

1. 初始化配置
2. 初始化插件
3. 初始化服务

其他杂七杂八的属于字节码增强技术，目前先使用，不深究原理

### 资料链接

https://jimmy2angel.github.io/tags/SkyWalking/

### 初始化配置

读取config/agent.config文件，将所有配置读取进来，过程暂不赘述，后续有需求再添加

### 初始化插件

很多文档写这里的，后续需要再添加

### 初始化服务

插件是用来放置到各个app里面进行拦截的，而服务则包含如何将插件拦截的信息传输个collector，因此这里所有的服务有着对接collector以及实现机制的很多代码

skwaling中的服务模板是按照prepare, startup, onComplete来完成的。

主函数使用遍历将所有服务查询一遍，并且所有服务并行执行prepare, startup, onComplete, shutdown的逻辑。

其中，boot包含了前三者的逻辑，是一个服务开始的地方。而shutdown则是关闭的地方。

![image-20210129095647359](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129095647359.png)

那service什么时候关闭呢？

使用的是主函数中的addShutdownHook

![image-20210129095748382](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129095748382.png)

该函数在jvm中增加一个钩子函数，当jvm在关闭时，会执行系统中已经设置的所有通过方法添加的钩子，全部执行完钩子时才会关闭jvm。

多用于内存清理，对象销毁，线程池在进程关闭时的处理。

百度一下即可。

这里说明，一个servicemanager启动了所有的service，并且在jvm最后退出的时候才关闭所有的service

在servicemanager里面，通过load(allservices)加载了所有的服务，这些服务是怎么被找到并且加载进去的呢？这又是一个大工程

**通过serviceloader添加服务，serviceloader属于java基础知识，如果后续需要添加服务则详细了解**

D:\skywalking\skywalking\apm-sniffer\apm-agent-core\src\main\resources\META-INF\services

在此底下有所有的services的列表

![image-20210129111138299](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129111138299.png)

servicemanager里面包含什么样的服务呢？

盗图使用，服务有可能不全

![image-20210129101951400](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129101951400.png)

#### TraceSegmentServiceClient

负责将TraceSegment异步发送到Collector

位置：

skywalking\apm-sniffer\apm-agent-core\src\main\java\org\apache\skywalking\apm\agent\core\remote

##### prepare

![image-20210129102357546](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129102357546.png)

# 问题1

有一个问题，GRPCChannelManager也是需要加载的服务，这里的prepare需要它，怎么确保GRPC一定比Tracesegment要早呢？

直接通过findservice添加listener就可以了吗？



##### boot

#### contextmanager

```
{@link ContextManager} controls the whole context of {@link TraceSegment}. Any {@link TraceSegment} relates to
* single-thread, so this context use {@link ThreadLocal} to maintain the context, and make sure, since a {@link
* TraceSegment} starts, all ChildOf spans are in the same context. <p> What is 'ChildOf'?
```

由于tracesegment涉及到多线程的问题，所以这里采用ThreadLocal来持有context

![image-20210129144537298](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210129144537298.png)

持有两个context，一个是AbstractTracerContext，一个是RuntimeContext

这两个都是线程的局部变量，为了保持数据的一致性











- ContextManager 负责管理 TraceSegment 的上下文
- SamplingService 负责管理如何进行 TraceSegment 的采样
- GRPCChannelManager 负责管理 GRPCChannel 的创建及将状态发送至各个监听器
- JVMService 负责将 JVM 以及 GC 信息发送到 Collector
- AppAndServiceRegisterClient 作为一个客户端，负责进行服务注册、服务实例注册、心跳检测等调用
- ContextManagerExtendService 作为 ContextManager 的扩展帮助类



每个服务需要具体用到的时候再进行学习。

## OAP-server

![image-20210201111225487](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210201111225487.png)

启动的模块









## Trace

### TraceSegment



/d/code/APM/apm-sniffer/apm-agent-core/src/main/java/org/apache/skywalking/apm/agent/core/context



`org.skywalking.apm.agent.core.context.trace.TraceSegment`是一次分布式链路追踪的一段

- **一条** TraceSegment ，用于记录所在**线程**( Thread )的链路。
- **一次**分布式链路追踪，可以包含**多条** TraceSegment ，因为存在**跨进程**( 例如，RPC 、MQ 等等)，或者垮**线程**( 例如，并发执行、异步回调等等 )。

https://github.com/opentracing/specification/blob/master/specification.md#references-between-spans

refs（List<TraceSegmentRef> 类型）：它指向父 TraceSegment。在我们常见的 RPC 调用、HTTP 请求等跨进程调用中，一个 TraceSegment 最多只有一个父 TraceSegment，但是在一个 Consumer 批量消费 MQ 消息时，同一批内的消息可能来自不同的 Producer，这就会导致 Consumer 线程对应的 TraceSegment 有多个父 TraceSegment 了，当然，该 Consumer TraceSegment 也就属于多个 Trace 了。



relatedGlobalTraces（DistributedTraceIds 类型）：记录当前 TraceSegment 所属 Trace 的 Trace ID

spans（List<AbstractTracingSpan> 类型）：当前 TraceSegment 包含的所有 Span。

ignore（boolean 类型）：ignore 字段表示当前 TraceSegment 是否被忽略。主要是为了忽略一些问题 TraceSegment（主要是对只包含一个 Span 的 Trace 进行采样收集）。

isSizeLimited（boolean 类型）：这是一个容错设计，例如业务代码出现了死循环 Bug，可能会向相应的 TraceSegment 中不断追加 Span，为了防止对应用内存以及后端存储造成不必要的压力，每个 TraceSegment 中 Span 的个数是有上限的（默认值为 300），超过上限之后，就不再添加 Span了。



#### ID

![image-20210114111521033](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114111521033.png)

在跨线程、或者跨进程的情况下时，创建的 TraceSegment 对象，需要指向父 Segment 的 DistributedTraceId ，所以需要移除默认创建的。

![image.png](http://www.zyiz.net/upload/202003/19/202003191701558350.png)



#### span

span编号从0开始自增，在创建span的时候生成

apm-sniffer/apm-agent-core/src/main/java/org/apache/skywalking/apm/agent/core/context/trace

![img](http://static.iocoder.cn/images/SkyWalking/2020_10_01/03.png)



![image-20210127140625526](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127140625526.png)

AbstractTracingSpan的start和finish

start的时候记录当前时间，finish的时候将span自己添加到tracesegment中

![image-20210127140951668](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127140951668.png)

在abstraceTracingSpan下面的StackBasedTracingSpan中，复写finish方法：

![image-20210127141204551](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127141204551.png)

只有在栈深度为0的时候才允许结束，其余return false





#### EntrySpan

实现StackBasedTracingSpan，**入口**span，用于服务提供者（Tomcat）

![image-20210114113009794](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114113009794.png)

![image-20210127141624526](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127141624526.png)

currentMaxDepth在entrySpan中先设置为0，当start()后，栈深度等+1，正确时启动上层的start（）函数

![image-20210127142107772](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127142107772.png)

后续在插件中均使用createEntrySpan来创建一个EntrySpan，已经封装好了？

问题1：

stackDepth和currentMaxDepth之间是什么关系呢？

stackDepth根据当前栈的深度不断的更新，当stackDepth在0的时候才可以使用finish进行结束span

那么currentMaxDepth中存取得是什么信息呢？当前span的深度吗？



问题2：

peer是干什么的



```
EntrySpan 是 TraceSegment 的第一个 Span ，这也是为什么称为"入口" Span 的原因。

那么为什么 EntrySpan 继承 StackBasedTracingSpan ？

例如，我们常用的 SprintBoot 场景下，Agent 会在 SkyWalking 插件在 Tomcat 定义的方法切面，创建 EntrySpan 对象，也会在 SkyWalking 插件在 SpringMVC 定义的方法切面，创建 EntrySpan 对象。那岂不是出现两个 EntrySpan ，一个 TraceSegment 出现了两个入口 Span ？

答案是当然不会！Agent 只会在第一个方法切面，生成 EntrySpan 对象，第二个方法切面，栈深度 + 1。这也是上面我们看到的 #finish(TraceSegment) 方法，只在栈深度为零时，出栈成功。通过这样的方式，保持一个 TraceSegment 有且仅有一个 EntrySpan 对象。

当然，多个 TraceSegment 会有多个 EntrySpan 对象 ，例如【服务 A】远程调用【服务 B】。

另外，虽然 EntrySpan 在第一个服务提供者创建，EntrySpan 代表的是最后一个服务提供者，例如，上面的例子，EntrySpan 代表的是 Spring MVC 的方法切面。所以，startTime 和 endTime 以第一个为准，componentId 、componentName 、layer 、logs 、tags 、operationName 、operationId 等等以最后一个为准。并且，一般情况下，最后一个服务提供者的信息也会更加详细。
```





#### exitspan

用于服务消费者( Service Consumer ) ，例如 HttpClient 、MongoDBClient 。

当前请求离开当前服务，进入其他服务时创建的span



#### localspan

本地方法调用时可能创建的span类型



#### setComponent()

用于设置组件类型

在ComponentDefine中可以找到skywalking目前支持的组件类型

./apm-protocol/apm-network/src/main/java/org/apache/skywalking/apm/network/trace/component/ComponentsDefine.java

![image-20210114141248600](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114141248600.png)



#### log()

用于向当前span中添加log，一个span可以包含多条日志

用List<LogDataEntity>来记录log

如：

异常日志中，会记录日志时间戳以及k - v信息

key：stack

value：异常堆栈



#### start()

开启span，会设置当前span的开始时间和调用层级信息



#### isEntry()

判断是否是EntrySpan



#### isExit()

功能类似同上



#### ref()

设置关联的traceSegment



#### finish(TraceSegment owner)

关闭span，记录最后关闭时间。并将span记录到所属的tracesegment中的spans 里面



### TraceSegmentRef

父span信息

traceSegmentId： 父traceSegment的ID

spanid：父span的id，与traceSegmentID结合确认父span

type：CROSS_PROCESS/ CROSS_THREAD

peerID和peerHost：父应用的地址信息

parentEndpointName和 parentEndpointId：父应用的Endpoint信息



### TracingContext

TracingContext保存在ThreadLocal中，包含了链路中的所有数据，以及对数据的管控方法，主要是对Span的管控。同时提供对ContextCarrier数据的处理，包括：

- 将TraceSegment转换为ContextCarrier，即org.apache.skywalking.apm.agent.core.context.TracingContext#inject
- 从ContextCarrier数据中抽取TraceSegment数据，即org.apache.skywalking.apm.agent.core.context.TracingContext#extract

**一个** TraceSegment 对象，关联到**一个**线程，负责收集该线程的链路追踪数据，因此使用线程变量。

而**一个** AbstractTracerContext 会关联**一个** TraceSegment 对象，ContextManager 负责获取、创建、销毁 AbstractTracerContext 对象。

#### TracingContext



#### inject(ContextCarrier)

![image-20210114150745311](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114150745311.png)

TracingContext.java

在跨进程调用之前，调用方会通过inject方法将当前的Context上下文记录全部的信息都注入到ContextCarrier里面，Agent后续会将ContextCarrier序列化并输出

#### extract()

功能类似上文



#### ContextSnapshot capture()

跨线程调用的接收方会从收到的ContextSnapshot中读取trace信息并填充到context上下文中



#### createEntrySpan(). createLocalSpan(). createExitSpan()

略

#### activeSpan()

获取当前活跃的Span

#### stopSpan()

停止指定的Span

## ContextManager

ContextManager类是各种skywalking agent插件的枢纽。不同组件的skywalking插件，如mq，dubbo，tomcat，spring等skywalking agent插件均是通过调用ContextManager创建和管控TracingContext，TraceSegment，EntrySpan，ExitSpan，ContextCarrier等数据。可以说ContextManager管控着agent内的链路数据生命周期。

在 ContextManager 提供的 createEntrySpan()、createExitSpan() 以及 createLocalSpan()等方法中，都会调用其 getOrCreate() 这个 static 方法，在该方法中负责创建 TracingContext，如下所示：

之后会再调用TracingContext的相应方法，创建指定类型的 Span。

ContextManager 中的其他方法都是先调用 get() 这个静态方法拿到当前线程的 TracingContext，然后调用 TracingContext 的相应方法实现的



上下文载体ContextCarrier

为了实现分布式追踪，需要绑定跨进程的追踪，并且上下文应该在整个过程中随之传播。这就是contextcarrier的职责。

一下是有关如何A->B分布式调用中使用ContextCarrier的步骤

1.在客户端，创建一个新的空的ContextCarrier

2.通过ContextManager#createExitSpan创建一个ExitSpan或者使用ContextManager#inject来初始化ContextCarrier

<font color=red>这里为什么使用createExitSpan来举例呢？其他类型的span是否可以初始化一个ContextCarrier呢？</font>

3.将ContextCarrier所有信息放到请求头（如HTTP HEAD），附件（如Dubbo RPC框架），或者消息（如Kafka）中。

4.通过服务调用，将ContextCarrier传递到服务端。

<font color=red>什么样的服务调用可以将上下文传递呢？</font>

5.在服务端，在对应组件的头部，附件或消息中获得ContextCarrier所有内容。

6.通过ContextManger#createEntrySpan创建EntrySpan或者使用ContextManager#extract来绑定服务端和客户端



http://www.itmuch.com/books/skywalking/guides/Java-Plugin-Development-Guide.html

还是这个文档

模块的大概介绍在这里

http://www.zyiz.net/tech/detail-119884.html

# Skywalking Java Agent源码结构简介

在讲trace segment id生成逻辑前先简要介绍一下Skywalking Java Agent的源码结构。 Skywalking v5.0.0-GA中，Java Agent对Skywalking链路信息的抽象与封装是位于源码的`apm-sniffer/apm-agent-core`模块中。本文对该模块的链路数据处理逻辑做如下分析。

- 链路数据模型抽象与封装 作用是封装链路数据协议约定的数据抽象，同时总体兼容了Opentracing规范。重点类有如下几个
  - `org.apache.skywalking.apm.agent.core.context.trace.TraceSegment`
  - `org.apache.skywalking.apm.agent.core.context.trace.TraceSegmentRef`
  - `org.apache.skywalking.apm.agent.core.context.trace.EntrySpan`
  - `org.apache.skywalking.apm.agent.core.context.trace.ExitSpan`
  - `org.apache.skywalking.apm.agent.core.context.ids.DistributedTraceIds`
- 跨进程链路数据抽象与封装 重点类是`org.apache.skywalking.apm.agent.core.context.ContextCarrier`。即是对本文所讲解的协议规范的封装。
- 链路数据采集模型抽象与封装 重点类是`org.apache.skywalking.apm.agent.core.context.TracingContext`。TracingContext保存在ThreadLocal中，包含了链路中的所有数据，以及对数据的管控方法，主要是对Span的管控。同时提供对ContextCarrier数据的处理，包括：
  - 将TraceSegment转换为ContextCarrier，即org.apache.skywalking.apm.agent.core.context.TracingContext#inject
  - 从ContextCarrier数据中抽取TraceSegment数据，即org.apache.skywalking.apm.agent.core.context.TracingContext#extract
- 链路数据采集模块 重点是`org.apache.skywalking.apm.agent.core.context.ContextManager`类。ContextManager类是各种skywalking agent插件的枢纽。不同组件的skywalking插件，如mq，dubbo，tomcat，spring等skywalking agent插件均是通过调用ContextManager创建和管控TracingContext，TraceSegment，EntrySpan，ExitSpan，ContextCarrier等数据。可以说ContextManager管控着agent内的链路数据生命周期。
- 链路数据上传模块 重点是`org.apache.skywalking.apm.agent.core.remote`包中的`TraceSegmentServiceClient`类。 ContextManager采集节点内的链路数据片段（TraceSegment）后，通知TraceSegmentServiceClient将数据上报到Collector Server。其中涉及到 `TracingContextListener`与skywalking封装的`内存MQ`组件，后面会详细分析。

以上每个部分都涉及复杂的处理逻辑，后面会分篇一一做详细讲解。

```
这样的话就会导致某些调用在服务 A 上被采样了，在服务 B，C 上不被采样，也就没法分析调用链的性能，那么 SkyWalking 是如何解决的呢。
它是这样解决的：如果上游有携带 Context 过来（说明上游采样了），则下游**强制**采集数据。这样可以保证链路完整。
```



<font color=red>问题</font>

问题1：

一个完成的链路数据，存放在哪里呢？

collector和最后一次子调用所包含的ContextCarrier中？



问题2：

是否可以直接使用skywalking的架构，在ContextCarrier中修改源码，添加repeat-context类型，完成封装

解耦问题



问题3：



## 跨进程链路数据抽象与封装

contextcarrier

![image-20210127110437615](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127110437615.png)

### contextcarrier

![image-20210127110924127](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127110924127.png)

![image-20210127111015457](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127111015457.png)

其中的item定义处

![image-20210127112601298](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210127112601298.png)





### collector

![img](http://static.iocoder.cn/images/SkyWalking/2020_10_10/01.png)

### tomcat样例

skywalking-plugin.def

![image-20210114103457589](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114103457589.png)



#### TomcatInstrumentation

![image-20210114103843020](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114103843020.png)

standard implementation of the HOST interface. Each child container must be a Context implementation to process the requests directed to a particular web application



![image-20210114144305036](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114144305036.png)



## 部署

### 虚拟机下部署

安装zookeeper时出现的问题：

https://blog.csdn.net/qq_40133908/article/details/89603471

启动zookeeper

zkServer



kafka验证命令

在kafka/bin/windows目录下

kafka-console-producer.bat --broker-list localhost:9092 --topic test

kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test





容器部署

kafka，zookeeper

https://www.cnblogs.com/lxc123/p/13963316.html

skywalking

https://www.cnblogs.com/fengyun2050/archive/2004/01/13/12115097.html



## 验证

### mvn test

https://blog.csdn.net/weixin_43978412/article/details/99977481



# skywalking分析

## skywalking entrance

skywalking是一款性能监控软件，抛开架构来说，先找其主入口

![image-20210125152200336](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125152200336.png)

![image-20210125152252782](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125152252782.png)

#### 初始化config

#### 查找插件

#### agentbuilder注入插件

#### openReadEdge

这是啥

#### cache缓存机制

#### 实例启动

serviceManager

bootservice管理器。负责管理，初始化bootservice实例

一次性将所有的serviceload进来，一次性将所有service以prepare，startup，oncomplete，shutdown位置拉起

 SPI的机制，在rescoure底下也会有meta-inf

所有的服务直接或者间接的实现bootservice





#### 添加shutdownhook





# 流量录制与回放

## URI与URL

### 定义

https://www.cnblogs.com/throwable/p/9740425.html

### 为什么留量录制和回放要知道这个？

尚未弄明白。。。

## **架构设计**

### 流量录制框架

#### 一次整个的调用

框架的核心逻辑录制协议基于JVM-Sandbox的`BEFORE`、`RETRUN`、`THROW`事件机制进行录制流程控制，详见[DefaultEventListener](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/impl/api/DefaultEventListener.java)：

> 基于[TTL](https://github.com/alibaba/transmittable-thread-local)解决跨线程上下文传递问题，开启`RepeaterConfig.useTtl`之后支持多线程子调用录制
>
> 开放插件定义enhance埋点/自定义调用组装方式快速实现插件适配
>
> [Invocation](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/domain/Invocation.java)抽象[Identity](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/domain/Identity.java)统一定位由插件自己扩展实现
>
> 基于[Tracer](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/trace/Tracer.java)实现应用内链路追踪、采样；同时支持多种过滤方式，插件可自由扩展；

### 流量录制插件

插件接口

[InvokePlugin](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/spi/InvokePlugin.java)

基础结构按照这个来



## 数据流

获取数据

例如：springboot中获取数据

before method和after method里面，通过对args参数的获取得到数据

spring中使用：

blog.csdn.net/hustspy1990/article/details/79251552

样例

保存数据

1. 存在哪里
2. 什么格式



调取数据

## dubbo应用在jvm-sandbox-repeater中使用：

### test

## 核心原理

### 流量录制

对于Java调用，一次流量录制包括一次入口调用(`entranceInvocation`)（eg：HTTP/Dubbo/Java）和若干次子调用(`subInvocations`)。流量的录制过程就是把入口调用和子调用绑定成一次完整的记录，框架抽象了基础录制协议，调用的组装由调用插件([InvokePlugin](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/spi/InvokePlugin.java))来完成，需要考虑解决的核心问题：

- 快速开发和适配新插件
- 绑定入口调用和子调用（解决多线程上下文传递问题）
- `invocation`唯一定位，保障回放时精确匹配
- 自定义流量采样、过滤、发送、存储

框架的核心逻辑录制协议基于JVM-Sandbox的`BEFORE`、`RETRUN`、`THROW`事件机制进行录制流程控制，详见[DefaultEventListener](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/impl/api/DefaultEventListener.java)：

> 基于[TTL](https://github.com/alibaba/transmittable-thread-local)解决跨线程上下文传递问题，开启`RepeaterConfig.useTtl`之后支持多线程子调用录制
>
> 开放插件定义enhance埋点/自定义调用组装方式快速实现插件适配
>
> [Invocation](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/domain/Invocation.java)抽象[Identity](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/domain/Identity.java)统一定位由插件自己扩展实现
>
> 基于[Tracer](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/trace/Tracer.java)实现应用内链路追踪、采样；同时支持多种过滤方式，插件可自由扩展；

```
public void onEvent(Event event) throws Throwable {
    try {
        /*
         * event过滤；针对单个listener，只处理top的事件
         */
        /** -------- **/
        /*
         * 初始化Tracer开启上下文追踪[基于TTL，支持多线程上下文传递]
         */
        /** -------- **/
        /*
         * 执行基础过滤
         */
        /** -------- **/
        /*
         * 执行采样计算
         */
        /** -------- **/
        /*
         * processor filter
         */
        /** -------- **/
        /*
         * 分发事件处理
         */
    } catch (ProcessControlException pe) {
        /*
         * sandbox流程干预
         */
    } catch (Throwable throwable) {
    	 /*
    	  * 统计异常
    	  */
    } finally {
        /*
         * 清理上下文
         */
    }
}
```

### 流量回放

流量回放，获取录制流量的入口调用入参，再次发起调用。注意：**读接口或者幂等写接口可以直接回放，否则在生产环境请谨慎使用，可能会造成脏数据**；用户可自行选择mock回放或者非mock，回放过程要解决的核心问题：

- 多种入口(HTTP/Dubbo/Java)的回放发起
- 自定义回放流量数据来源、回放结果的上报
- 自定义mock/非mock回放、回放策略
- 开放回放流程关键节点hook

回放过程通过异步EventBus方式订阅回放请求；基于[FlowDispather](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/api/FlowDispatcher.java)进行回放流量分发，每个类型回放插件实现[Repeater](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/spi/Repeater.java)SPI完成回放请求发起；每次回放请求可决定本地回放是否mock，插件也自由实现mock逻辑，mock流程代码

> mock回放：回放流量子调用（eg:mybatis/dubbo)不发生真实调用，从录制子调用中根据 [MockStrategy](https://github.com/alibaba/jvm-sandbox-repeater/blob/master/repeater-plugin-api/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/spi/MockStrategy.java) 搜索匹配的子调用，利用JVM-Sandbox的流程干预能力，有匹配结果，进行`throwReturnImmediately`返回，没有匹配结果则抛出异常阻断流程，避免重复调用污染数据

```
public void doMock(BeforeEvent event, Boolean entrance, InvokeType type) throws ProcessControlException {
    /*
     * 获取回放上下文
     */
    RepeatContext context = RepeatCache.getRepeatContext(Tracer.getTraceId());
    /*
     * mock执行条件
     */
    if (!skipMock(event, entrance, context) && context != null && context.getMeta().isMock()) {
        try {
            /*
             * 构建mock请求
             */
            final MockRequest request = MockRequest.builder()
                    ...
                    .build();
            /*
             * 执行mock动作
             */
            final MockResponse mr = StrategyProvider.instance().provide(context.getMeta().getStrategyType()).execute(request);
            /*
             * 处理策略推荐结果
             */
            switch (mr.action) {
  					...
            }
        } catch (ProcessControlException pce) {
            throw pce;
        } catch (Throwable throwable) {
            ProcessControlException.throwThrowsImmediately(new RepeatException("unexpected code snippet here.", throwable));
        }
    }
}
```

![image-20210119093740688](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210119093740688.png)

这里面通过两个EnhanceModel的实例，实现了onResponse录制和invoke回放的实现？

下面问题：

1. enhancemodel是怎么实现的

## EnhanceModel



## RepeatContext

![image-20210120162608233](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120162608233.png)

其中主要数据

### RepeatMeta

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120162839477.png)

#### appName --string

#### traceId --string

与上层的traceid有什么不同？

#### mock --boolean

尚未明确

#### StrategyType

mock的策略，目前并未分析

#### repeatId

回放id？干嘛用的

#### matchPercentage --double

目前并不需要

#### datasource --string

http还是rpc？

#### timeout

超时时间

#### extension

map类型，尚未了解

### RecordModel

![image-20210121134545067](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210121134545067.png)

主要是下面的invocation吧

这里的invocation是指的是jvm-sandbox的invocation，并不是dubbo的invocation

#### invocation //实现序列化接口

![image-20210120164045771](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120164045771.png)

![image-20210120164154538](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120164154538.png)

这里的request和response参数只snapshot下来，并不进行传输保存，下面有其序列化的string，在mock的时候才会进行使用，从cache中寻找相似度高的序列化值，作为mock打桩的返回值



##### invokeType

![image-20210120164326765](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120164326765.png)

##### Identity

![image-20210120164700254](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120164700254.png)



#### ！！！request和response

##### dubbo泛化调用

https://www.jianshu.com/p/3a22a53c7068

./jvm-sandbox-repeater/repeater-plugins/dubbo-plugin/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/dubbo/DubboRepeater.java

![image-20210120210239084](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120210239084.png)

这两句，所有铺垫之后的重点，在这里用dubbo的泛化调用，直接完成了调用



![image-20210120210442935](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120210442935.png)

从框架角度来说，无所谓如何去执行录制和回放，先把接口搞定。如上图dubbo通过dubbo的插件其中的泛化调用来实现回放。我草，实在是太惊艳了。泛化调用还需要研究一下

### DefaultEventListener

除了http插件，其他插件都默认使用的defaultEventListener

![image-20210125095040940](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125095040940.png)

这说明里面的东西其实都是使用cache的啊

![image-20210125095711720](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125095711720.png)

./repeater-console/repeater-console-service/src/main/java/com/alibaba/repeater/console/service/convert/InvocationConverter.java

这里是repeater-console里面的东西，做的就是将invocation转化成invocationBO，try中就是看是否可以进行序列化，如果可以的话就可以转，不然的话就异常

![image-20210125104339940](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125104339940.png)

这里response可以理解，是一个object类型，但是request是object[]，里面也没有实现object[]的东西啊，实现方式难道没有缺失？

![image-20210125104638094](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125104638094.png)

### 其他

./repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/impl

研究一下这个底下的实现，很有用

Abstract*相关的

./repeater-plugin-core/src/main/java/com/alibaba/jvm/sandbox/repeater/plugin/core/impl/AbstractInvokePluginAdapter.java

![image-20210120211303672](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120211303672.png)

processor是用来处理入参和返回值的吗？

![image-20210120212042495](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120212042495.png)

invocationlistener和invocationprocessor有什么区别呢

![image-20210120212243922](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120212243922.png)

![image-20210121135058563](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210121135058563.png)

回调调用监听器InvocationListener的onInvocation方法，判断Invocation是否一个入口调用，如果是的话调用消息投递器的broadcastRecord将录制记录序列化后上传给repeater-console。如果不是的话当做子调用保存到录制缓存中。

![image-20210120212334364](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120212334364.png)

![image-20210120212540177](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210120212540177.png)

其中保存了里面的classloader，这个我们是否需要保存呢？

在这里触发的调用时间的方法参数用``` Object[] argumentArray```来保存，其实我们通过skywalking也是可以获得这个参数的，但是在dubbo应用中是否泛化调用只需要入参就可以呢？其他还需要保存什么东西，如下列个清单：

对不起。。上面已经显示出来了

![image-20210121091548884](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210121091548884.png)

只需要按照模板，写出来一个就好了。。

下一个问题：

dubbo插件里面的invoke和invocation有什么用呢？通过args里面是获得了他们。

问题：

invocation不传输参数，只做snapshot，那下面怎么使用该参数呢

![image-20210121104244430](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210121104244430.png)

这里将invocation保存在cache里面，没有进行存储？

intimeserializerequest这里面，其中包含object[]数组，但是transient了，所以其他参数都是可以序列化的，唯独此次调用的入参没办法进行序列化保存





# java transient

是什么：



### string traceId



#### Invocation

入口调用和子调用



#### invoke



# flow-copy

## 概述

该组件由下面几项功能组成：

前端ui：

应用列表+机器列表

复制回放任务管理

结果分析、标记



服务端：

复制回放任务下发

数据脱敏

结果比对

mockServer



配置中心：

zookeeper

servicecombKie



agent端：

应用采集

心跳上报

任务监听

流量复制

流量回放



后端存储：

elasticsearch



## API

## CORE

![image-20210125214228382](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125214228382.png)



```
基础功能：
trigger来触发，触发点获得invocation信息，来完成流量复制的功能。
再一个trigger，触发流量回放的功能；repeater实现基本类，由plugin来完成回放。

上述两个完成了功能上的结构，从这两个trigger来想下面发生的问题：
trigger-record：
获取context上下文信息，包括但不限于：
1.traceid
2.span信息？需要将调用联系起来，比如通过相同traceid不同span来判断调用的流程，**查看能否复用skywalking的东西得到此功能**
3.本次invocation信息，存在invocation的一个list，存放所有的子调用。这个也要**查看是否能复用skywalking信息**，最终目标<通过不同span可以在list中找到不同的invocation，通过不同的回放策略，判断触发的是流量的回放还是直接返回存放的response>[1]
4.app的名字，时间，采样率等相关信息<后续逐步的补充>

并且异步保存至数据库(是将repeat-context存入数据库)

[1]回放功能使用调用是真的调用接口，mock的话直接返回已经记录的返回值response
问题1：
一个调用在before method函数中存放的是入参，返回值如何获得？，获得后怎么存入相应的invocation
after method中存放返回值


问题2：
如何存放span和invocation，能最快的查找到，map？

问题3：
mock的策略有哪些
1.不进行mock，所有的信息由子调用自行完成
2.子调用均进行mock，只有入口调用使用真实调用----意义在哪里
3.选择性mock，对数据库及写的接口类使用mock，查询类接口子调用自行完成----mock点在哪里

问题4：
如何在数据库中查找invocation<<<<查找repeat-context>>>>
回放的时候需要将他们提取出来，提取的特征是什么：
1.某段时间触发后的所有repeat-context信息？

trigger-repeat:
前端ui触发流量回放信息，提供包括但不限于：
    应用信息，ip信息，回放时间，mock策略，是否录制测试环境来提供不同点报告，采样率等
获取上述前端信息，在指定时间内进行流量的回放，提供的功能：
1.获得所有的repeat-context
2.按照时间序列回放，各插件各自领取并执行回放功能
3.判断是否需要录制当前环境所有的调用请求，保存至数据库

问题1：
repeat-conext包含着一次完整的调用，其中有可能包含不同的app，不同的invocation，回放器基于repeat-context分发不同invocation给不同插件？如何确保有序？
不需要确保有序，我们只发入口调用即可，后续的调用通过mock的策略来看到底mock还是真实调用
**记录子调用的主要原因1.为了mock的时候提供返回值。2.若需要对比生产环境和测试环境不同点时，需要提取出来对比**

问题2：
大规模回放，需要线程池的支持？








终极问题：
流量回放是用来干什么的？
回放流量只是为了复现问题，还是需要展示出复现的情况和生产情况的不同点
只复现问题只需要回放流量即可，要展示的话还需要**录制**测试环境的调用链，再将调用链与生产环境的做对比，最终发送给前端

采样率
如果线上出现了大规模的调用，通过采样率来限制次数，在plugin里面实现，在流量回放的时候需要按比例回放
如何限制：
控制method里面的次数，但是每次调用其实其中的数据是不同的，因此有可能导致无法复制出生产环境有问题的数据
一个tradeoff，接受这个问题？

上下文携带问题
依存在skywalking的contextCarrier中，新添加我们结构体repeat-context，并且在各插件中完善结构体内容

x-paas和skywalking
差距多少，我们可以改他们代码吗？不能解耦的地方怎么处理




```



### skywalking中traceid的实现



bridge

目前看起来不需要，我们所有的都保存到后端存储中，或改名为connection之类的？

cache

暂时不加cache，后续补充

eventbus

回放序列化属于cpu密集型，需要进程池来完成

impl

核心关键库

获得plugin封装的invocation，封装成为recordcontext



model

serialize

spring

trace

util

wrapper



## plugin-api

![image-20210125214708119](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210125214708119.png)

存所有的数据结构（domian）

异常（exception）

api，spi

### recordcontext：

#### traceid---唯一性

string类型

通过traceid来进行数据库查询？

#### record-meta

##### appName

string类型，是哪个应用传递来的参数

##### traceid

是否需要继承到下面来呢？

##### mock

boolean

是否启用mock，启用后子调用都不真正实现

##### strategyType

mock的策略

##### repeatid

回放id

##### matchpercentage

匹配度

##### datasource

回放数据源

##### timeout

##### extension

不知道是干嘛的



#### record-model

##### timestamp

##### traceid

##### appName

##### environment

##### host

##### invocation

##### subinvocation



##### invocation



## es存储字段

公共字段为

#### index

有下面几种index类型

#### type

种类

#### id

把自己本身标识

### GlobalTrace

#### global_trace_id

全局链路追踪编号的数组

#### segment_Id

traceSegment链路编号

#### time_bucket

时间戳

### InstPerformance

#### application_id

应用编号

应用名太长，用编号辨识

#### instance_id

应用实例编号

#### calls

调用总次数

#### cost_total

消耗总时长

#### time_bucket

时间戳

### SegmentCost

SegmentCost：TraceSegment = 1：1

#### segment_id

TraceSegment编号

#### application_id

应用编号

#### start_time

开始时间

#### end_time

结束时间

#### service_name

操作名

#### cost

消耗时长

#### time_bucket

时间戳

### nodeComponent

#### nodeComponent

component_id类型

#### segmentId

链路编号

#### timeBucket

时间戳

#### peer_id

看不懂

### NodeMapping

#### address_id

node地址id

#### time_bucket

时间戳

#### application_id



应用编号

### NodeReference

#### front_application_id

#### behind_application_id







## Plugin

before method中：

1. 封装invocation
2. 触发点



触发点 trigger，使得repeater对封装的invocation再次封装数据结构，并且保存到后端存储中。

实现repeater里面的executeRepeat方法，达成回放



获取config信息，判断需要截获的方法，这里获取的是本地的config，在远程的config由core来实现从配置中心拉取的计划

## CONSOLE



## HESSIAN



## 问题列表

### 整体框架结构使用skywalking还是sandbox-repeater

#### 我们能否保持解耦状态

skywalking：

流控的代码其实将skywalking进行了删减，如果需要改的话xpaas里面还需要对两个工程都进行修改

不使用skywalking好像也不行，因为需要得到traceid和span的信息，这些东西skywalking已经做好了，可以直接拿来用？

我们独立出来，还是在span里面保存invocation呢？

独立的话会好一点，我们只从中提取出我们需要的信息，比如traceid和span类型，剩下的都在我们自己的module里面维护

----因此需要依赖skywalking的信息

先搞懂skywalkoing是怎么使用的，我们使用的规格到底应该是什么



思路：

1. 确认x-paas中skywalking版本，或在x-paas中进行演进
2. 侵入修改skywalking源码的话最快捷，找到trace，读取tracesegement里面的span的参数即可。

若需要自行实现的话只能自己保存结构体信息，通过自己console进行整合分析，最终存入其中。

3. 存储估计还要参照skywalking类似的存储形式





当前有几块内容：

1. 整体架构（数据结构如何构建，需要提供什么样的功能）

   repeat-context所有内容

   contextmanager等功能实现

2. agent探针

   span的整体流程

   如何将我们自己的内容也放入context上下文进行传输

   如何整合传送给console端

3. console端进行整合数据信息

4. 与后端存储的连接

5. mock整体策略





基线：

x-paas版本

skywalking本身版本，其中是否有功能影响后续



dev-repeat分支构建

逐步往上合代码





# 资料

testerhome.com/topics/21165





1. 
2. zookeeper和agent的监听问题，是否需要lisener
3. http，dubbo，mysql，kafka，redis支持
4. kafka发送的粒度
5. 生产环境异步同步数据到kafka
6. 扩容flowrecord的时候，es的qps压力怎么搞
7. 一个任务id对应所有原始流量1:1亿很大，是否可以多建几级表
8. context上下文携带问题
9. 时间差异导致的差异如何去解决会好一点
10. 