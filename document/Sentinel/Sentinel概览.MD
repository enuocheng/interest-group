# **sentinel概览**
**注：后面全部上源码解析，透过源码断点跟踪**
## 1：限流
**resource：资源名，即限流规则的作用对象**

**count: 限流阈值**

**grade: 限流阈值类型（QPS 或并发线程数）**

**limitApp: 流控针对的调用来源，若为 default 则不区分调用来源**

**strategy: 调用关系限流策略**

**controlBehavior: 流量控制效果（直接拒绝、Warm Up、匀速排队）**

### **sentinel的两种限流模式**

#### 1：**并发线程数的统计**。
  当我们的并发线程数达到设定的阈值的时候就自动舍弃，我只吞我吞的下的流量。
#### **2：QPS统计**
  同样的原理，我一秒承受峰值也就9999的qps，那么最多也就接受这么多，多余的自动舍弃。
  
#### 3:碎言碎语
   如果我们只是为了限流，举个很简单的例子，搜索增量这里，可是说是一个底层服务，不依赖任何服务，没有什么进行熔断的，熔断一般是调用方进行熔断的，所以只需要限流，保证我的一个流量处于一个相对来讲较能承受的范围，
那么可以考虑一下RateLimiter，更加轻量级。
   就主要是做流量控制的，sentinel这里有很多算法也都是借鉴这里的。稍微讲一下，内部实现原理就是一个令牌桶算法，每秒会往桶中扔令牌，令牌完了就限制，拿到令牌那么恭喜你可以进行正常的业务处理。有一点需要注意就是他可以保存一秒的剩余令牌。也可以借下一秒的令牌。（这个程序员估计信用卡很多，提前消费的习惯贯彻于编码）
比如第一秒分配了10个令牌，用了9个，剩余一个，第二秒分配了10个令牌，那么本次可以用的就是1个（old）+10个（new），他会优先使用old令牌，比如说第二秒来了10个请求，那么剩余令牌数就是11-1（old）-9（new），第三秒依然是10个令牌，但是来了15个请求，那么依然可以处理完，10个新令牌+1个旧令牌+借下次的4个令牌。


#### 限流后的问题（即达到阈值后的一个操作）：
##### 1：直接拒绝。
就是不处理，直接丢掉。默认的限流策略，拒绝方式为flowException。


#### ^加了限流之后循环调dubbo就会死翘翘.....预热流量准备可以考虑循环掉dubbo 具体的一个请参考之前写的jit相关以及相关优化^


##### 2：warm Up

就是说我第一次100个请求过来，我丢了99个，那么下一秒可能只丢98个，直到达到峰值阈值)
内部原理可以参看我之前的文章，JVM预热-JIT（java字节码普通模式下还是需要转化为机器码，但对于热点代码
片段，他会将机器码存在于内存之中，这也是为什么我们想要达到真正的预热还是得拿流量去预热。多线程的编译
只是说提高一个预热速度而已）


就是说你的系统可能长期处于一个低迷状态，处理业务并不多，因此并没有达到一个最优的状态，所以当流量骤增的时候，会逐步让流量升高，给冷系统一个预热的时间。让其性能达到最优。

内部实现的算法类似于令牌桶，通过判断每秒桶的一个饱和度来判断系统当前的一个状况。

这个不建议轻易开启，只针对核心业务。毕竟内存还是有限，只针对核心业务。以下文字引自紫苑github-go实现jvm一文：

**“代码缓存”，它是用来存储已编译方法生成的本地代码。 代码缓存确实很少引起性能问题，但是一旦发生其影响可能是毁灭性的。 如果代码缓存被占满，JVM会打印出一条警告消息，并切换到interpreted-only 模式：JIT编译器被停用，字节码将不再会被编译成机器码。 因此，应用程序将继续运行，但运行速度会降低一个数量级，直到有人注意到这个问题。就像其他内存区域一样，我们可以自定义代码缓存的大小。 相关的参数是-XX:InitialCodeCacheSize 和-XX:ReservedCodeCacheSize，它们的参数和上面介绍的参数一样，都是字节值。 -XX:ReservedCodeCacheSize=100m -XX:+UseCompressedOops** 



##### 3：匀速排队


相当于漏桶算法，你来了一百个请求，我一秒处理1个，中间的排队（尚未达到桶的一个最大容量），后面的丢弃（超出桶的容量）。反正我一秒就只能处理一个。


基于调用关系的流量控制

1:根据调用方来进行限流
2：根据调用链路来进行限流：链路限流
3：具有关系的资源流量控制：关联流量控制

## 2：熔断降级
当我们的某一个服务出现问题的时候，尤其是底层服务，如果没有做处理，那么整个链路都处于一种长时间的等待的状态。我们肯定不希望这么做，理想的做法是什么，你挂了，我就不会再调用你了，等你OK了，我再调用。

那么sentinel是怎么做的呢。

失败统计，当我们单位时间内的失败次数达到多少的时候我们熔断。

那么什么时候复苏。恢复正常呢？

这里会有一个时间窗口这样的一个概念，当你达到熔断要求的时候，我会断开。有流量过来直接走熔断处理，当休眠一段时间之后，会处于一个半开的状态，试探性的去看你OK了没有（放部分流量过去），达到一定频次的话就会继续断开，处于休眠状态。

降级策略：
1:平均响应时间
当资源平均响应时间超过阈值之后，资源进入一个准降级状态，如果后续的五个请求RT（写死的）都超过这个阈值，那么在接下来的时间窗口，对这个接口的调用都会自动返回（抛出DegradException）
2:异常比例
每秒的总流量与异常量的一个比率，超过之后就会进入降级状态。
3:异常数
近一分钟的异常数目超过阈值之后进行熔断。



简单来讲，熔断的过程主要有三个     画图画图画图
### 1：系统正常进行
监控是否达到熔断指标
### 2：熔断 
   达到指定的失败频次，熔断开启，熔断时间段内不会再去访问服务提供者，而是直接抛出失败或者自定义结果。
### 3：熔断半开 
   达到时间窗口，放少许流量进来判断系统是否复苏，复苏则熔断关闭，否则就继续睡眠，等待下一个时间到来继续试探。
   ![熔断三状态](img/熔断.jpg)

## 3：塑形

   一般来讲。我们的流量是这样的，零散的，千军万马毫无秩序的蜂拥而来，系统作为备战方，一会儿晴天一会
    儿暴雨，这里有一点需要注意，就是说当我们的流量蜂拥而至的时候我们会进行一次调整，将零散的流量进行
    一次规整，使系统相对处于一个负载较均衡的一个状态。比如我来了1000个请求，但是我的处理能力只有每秒1
    00，那么我每秒只处理100个，其他的丢弃或者排队等待。具体的根据自定义的策略以及业务来进行判断。其实可以理解为漏桶算法，就是说我高峰期处理的流量是一定的，超过我每秒处理的一个量的时候我会把超出的部分堵塞排队，桶的容量满的时候就直接进行丢弃。这里没什么可讲的，后面直接上源码看。
    



