关于尚硅谷SpringCloud2020的学习项目

## 项目简介:

该项目是跟着尚硅谷周阳的Springcloud2020做出来的，里面涉及传统的 eureka、hystrix、ribbon，更是讲解了最新的 alibaba的 Nacos和sentinel、Seata，相当的给力。

首先在这里感谢阳哥，让我加深了对SpringCLoud的理解,写到吐的案例是真的让我不慌微服务的代码了。

该项目中有我按照视频的内容总结的思维导图，基本和阳哥的那个差不多，同样是mmap格式的。

与视频不同的是，我在思维导图中加入了一些我配置组件遇到的部分问题的解决方案，如果有和我一样，可以参考思维导图。

另外是推荐小伙伴们还是得多看官网的文档，视频只是引路人，很多东西是需要通过自己总结说出来才算正在掌握了的。



## 官网文档传送门：

SpringCloud: https://spring.io/projects/spring-cloud/

这个网址是各springcloud组件的配置介绍，自己搭建组件环境可以考虑看这个。

Seata： https://seata.io/zh-cn/docs/overview/what-is-seata.html

分布式事务解决的框架，文档介绍很详细，推荐。

Nacos： https://nacos.io/zh-cn/docs/what-is-nacos.html

那个替代Eureka和Config的男人。

Sentinel：[https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D](https://github.com/alibaba/Sentinel/wiki/介绍)

在Hystrix基础上增加了流控规则和持久化,alibaba体系的一员。


## eureka
    简介：服务注册中心
    原理：服务提供者注册在eureka上之后回吧自己的信息放在eureka上，消费者就从eureka上获取消息，并放在自己的jvm中
          调用服务提供者(底层还是用httpclient)。每三十秒再从eureka上获取提供者的信息
    知识点 ：
        1. 2.x代以后，依赖文件分成了server和client server的用于创建eureka服务注册中心
        2. 在server上 主启动类，注解的是 @EnableEurekaServer 在client 上注解的是@EnableEurekaClient
        3. server 上的yml文件上不用配置在eureka注册为true  
        4. eureka的集群配置是eurekaServer相互注册
        5. 负载均衡,使用restTemplate @loadBalance.  eureka-client包中有ribbon
        6. discoveryClient的使用
        7.自我保护机制 provider断线了，一段时间内provider的信息不会马上被eureka清除，还是继续使用 cap 中的ap
        
## zookeeper 服务注册中心
      1.注意依赖的zookeeper的版本和安装的zookeeper是一致的
      2.临时结点（创建结点的宕机了，就消失了）、永久结点  
        服务注册结点是 临时节点
## consul 注册中心(go 写的)
       eureka 满足的是ap(cap),consul/zookeeper 满足的是cp 
## ribbon  负载均衡  
      1.核心组件 iRule 
          roundrobinrule、randomrule、retryrule、bestAviable、
      2.创建自己的IRule 不能放在@component-scan包下
      3.在主启动类上配置@ribbonClient、负载平衡规则
      
## openfeign 服务调用,接口调用 底层有ribbon
      1.没有使用restemlate 直接调用接口就行
            主启动类加上@EnableFeignClients  接口上添加@FeignClient(value = "CLOUD-PAYMENT-SERVICE") 
            自带负载均衡ribbon
      2.超时读取设置
            如果provider的服务用时超过1s，那么就会报错 因此必须设置超时时间 
            ribbon.ReadTimeout=1000 //处理请求的超时时间，默认为1秒
            ribbon.ConnectTimeout=1000 //连接建立的超时时长，默认1秒
      3.feign日志以什么级别监控那个接口
## hystrix
      1.服务降级
         简介：服务断线，给个备选方案
         1. 主程序上标注
         2.provider 上@EnableCircuitBreaker
           在降级的方法上使用
           ```@HystrixCommand(fallbackMethod = "timeoutHandler",commandProperties = {   
                       @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "3000")
            })``` //java
           HystrixProperty的name 可以在 HystrixCommandProperties 中找到
          consumer上可以在controller上标注  @DefaultProperties(defaultFallback = "global_hystrix_fallback") 
            在具体的方法上使用 @HystrixCommand( commandProperties ={@HystrixProperty
            (name = "execution.isolation.thread.timeoutInMilliseconds",value="1500")}) 配置
              调用其他微服务的时候 可以在接口配置服务降级  
          避免代码爆炸
            使用feign 降级  配置service接口的实现类并制定该实现类处理降级业务
            @FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT", fallback = PaymentFallbackService.class)
      2.服务熔断
            简介：类似保险丝，达到一些限制条件时，直接跳闸不允许请求
              开启熔断器  @HystrixProperty(name = "circuitBreaker.enabled", value = "true")
                触发熔断器条件之后就禁止访问了，还继续访问的就返回fallback
              默认是20 线程数，错误率是50% 1s
      3.服务限流alibaba sertinal
              秒杀高并发时配置
              服务提供者服务超时
              服务提供者服务出错
              自己服务出错或超时
           hutool工具包
       4.hystrix dashboard
## 服务网关  zuul gateway
       1.gateway  底部集成了netty包 非阻塞式
           调用架构：前端 -> nginx -> 网关-> 微服务   
       2.三大核心：路由、断言、过滤
           路由：路由在yml文件上配置(断言，路径相匹配的进行路由)
           断言： 对请求的信息的匹配
           过滤: 
              自定义过滤器，实现GlobalFilter,Ordered
           路由转发，执行过滤链
## 服务配置 config
       简介： 集中管理微服务的配置、动态化的配置不同环境的配置。就是把配置文件放在github 然后读取github上的配置信息  
              创建config-center(3344): 通过访问config-center 获取在github上的文件
                1.主启动类 添加 @EnableConfigServer
                2.yml 文件添加 github 相关配置
              创建config-client(3355) 
                1.bootstrap.yml 文件上添加 config 配置。因为是client要读取github上的文件，因此需要用bootstrap.yml
               config 的动态刷新的问题
              修改 github上的配置文件后 3344 可以及时获取修改值，但3355 不能通过3344获取修改值
               问题：github上配置文件修改之后，center(3344)可以收到,client(3355)收不到。因此需要使用@refreshScope
                    然后运维人员发个post http://localhost:3355/actuator/refresh
                    每次都要使用去发送post ?  使用bus
## 消息bus 
       简介：分布式自动刷新配置功能  支持rabbitMq和kafka
         推送给各个服务消息 使用rabbitmq 
## stream 消息驱动
          屏蔽消息中间件的具体实现，只关注消息的模型 目前只支持 rabbitmq\kafka
          binder 屏蔽消息中间件的具体实现  source、sink 表示来源
          消息生成者，
              1.配置 消息组件类型(rabbitmq\kafkfa) ,配置output
              2.bindings: #服务的整合处理
                  output: #这个名字是一个通道的名称
                       destination: studyExchange #表示要使用的Exchange名称定义
                       content-type: application/json #设置消息类型，本次为json，本文要设置为“text/plain”
                       binder: defaultRabbit #设置要绑定的消息服务的具体设置
              3.消息发送类
                    1.配置@EnableBinding(Source.class) 
                    ```    @Resource
                     2.使用      private MessageChannel output; // 消息发送管道
                           @Override
                               public String send() {
                                   String serial = UUID.randomUUID().toString();
                                   output.send(MessageBuilder.withPayload(serial).build());
                                   System.out.println("*****serial: "  +serial);
                                   return null;
                               }
                     ```
          生成消费者
            1. 配置 消息组件类型(rabbitmq\kafkfa) ,配置output
            2. bindings: #服务的整合处理
                output: #这个名字是一个通道的名称
                     destination: studyExchange #表示要使用的Exchange名称定义
                     content-type: application/json #设置消息类型，本次为json，本文要设置为“text/plain”
                     binder: defaultRabbit #设置要绑定的消息服务的具体设置
            3.接受消息
                ```   @StreamListener(Sink.INPUT)
                       public void input(Message<String> message) {
                           System.out.println("消费者1号， -----> 接受到的消息： " + message.getPayload()
                           + "\t port: " + serverPort);
                       }
                ```
          重复消费问题的出现？
              8801 消息生成者   8802、8803(集群)消息消费者  
              把8802、8803设置成相同的组
          持久化
            我们发现有指定分组的服务8803，消息可以持久化，即使服务中途断开后重启仍然可以获得，
            而未指定分组的服务就会丢失断开期间发送到MQ的消息
## sluth 求链路跟踪整合了zipkin
              请求链路跟踪   


## alibaba
   ### nacos->eureka+config+bus 服务注册、配置中心、总线
     直接下载 start.up
     1.服务注册
     2.配置中心 DataId、namespace、group
     3.nacos集群  持久化     
   ### sentinel ->hystrix 
     1.流控规则
        关联
        直接
        链路
        结果 ：快速失败、warm up 、排队等待
     2.降级规则
        RT、异常比例、异常数 时间窗口之后 熔断器关闭 恢复正常
     3.自定义兜底逻辑、和代码耦合了的处理
        @sentinelResource
     sentinal 服务停止、宕机过后，规则会消失。消失消息的持久化问题？
       配置持久化到nacos
   ### seata 分布式事物
      简介：各个未付无之间只能保证自己本地的事务，而不能保证全局事务的一直性
      tc 事务协调者  tm 事务管理者  rm 资源管理者
      1.tm 向tc发出请求，生成一个全局事务id,并把这个事务id 分发给调用链上的微服务。
      2.rm 想tc注册进事务。 
      3.tm 向tc提出提交或回滚事务
      4.tc向其中的 xid 下的事务提交或回滚
   ### 唯一主键的生成
       唯一id生成。
           id要求： 唯一，趋势递增，时间戳
           性能要求：高可用、低延迟，高qps
        方式：   
           1.uuid:可以保证唯一性，但是对数据库不友好。因为作为主键是要成为索引的，uuid是无序的因此对于数据库的性能有所消耗，每次都要重新编排索引。因此不是很友好。降低插入的性能
           2.可以使用redis 但是要使用步长
           3.雪花算法  64bit   1bit:表示符号位 41bit表示时间戳、10bit表示工作机器id datasourceId、workid   12bit：同毫秒内产生不同的id