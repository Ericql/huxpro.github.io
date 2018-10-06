---
layout:       post
title:        "SpringCloud第一篇: 总述SpringCloud"
subtitle:     "SpringCloud概述"
date:         2018-10-03 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - SpringCloud
---
# SpringCloud第一篇: 总述SpringCloud
## 微服务概述
&emsp;&emsp;微服务架构风格这种开发方法,是以开发一组小型服务的方式来开发一个独立的应用系统的.其中每个小型服务都运行在自己的进程中,并经常采用HTTP资源API这样轻量的机制来相互通信.这些服务围绕业务功能进行构建,并能通过全自动部署机制来进行独立部署.这些微服务可以使用不同的语言来编写,并且可以使用不同的数据存储技术.对这些微服务我们仅做最低限度的集中管理.
## SpringCloud概述
&emsp;&emsp;SpringCloud是一个快速搭建分布式系统的通用模式的工具组件集.利用SpringBoot简化了分布式系统基础组件的开发,每个组件都是对应一个起步依赖,组件之间可以整合到一起发挥作用.  
&emsp;&emsp;SpringCloud融合了各种已经成熟的框架,通过SpringBoot风格进行封装出一套简单易懂、易操作的分布式系统开发的工具包.其主要组件有:Netflix公司下的Eureka、Ribbon、Feign、Hystrix、Zuul等组件,Spring Cloud Config、Spring Cloud Bus、Spring Cloud Cluster、Spring Cloud Consul、Spring Cloud Security、Spring Cloud Sleuth等组件,至于全部组件可查看官网.
## 特性
SpringCloud专注于提供良好的开箱即用的典型用例和可扩展性机制.
- 分布式/版本化配置
- 服务注册和发现
- 路由
- service-to-service调用
- 负载均衡
- 断路器
- 分布式消息传递

## 主要成员
### Netflix Eureka
&emsp;&emsp;在Spring Cloud Netflix技术栈中,Eureka作为服务注册中心对整个微服务架构起着最核心的整合作用;Eureka是Netflix开发的服务发现框架,本身是一个基于REST的服务,主要用于定位运行在AWS域中的中间层服务,以达到负载均衡和中间层服务故障转移的目的.SpringCloud将它集成在其子项目spring-cloud-netflix中,以实现SpringCloud的服务发现功能.
### Netflix Ribbon
&emsp;&emsp;Ribbon是Netflix发布的云中间层服务开源项目,其主要功能是提供客户端侧负载均衡算法.Ribbon客户端组件提供一系列完善的配置项如连接超时,重试等.简单的说,Ribbon是一个客户端负载均衡器,我们可以在配置文件中列出Load Balancer后面所有的机器,Ribbon会自动的帮助你基于某种规则(如简单轮询,随机连接等)去连接这些机器,我们也很容易使用Ribbon实现自定义的负载均衡算法
### Netflix Feign
&emsp;&emsp;Feign是一种声明式、模板化的HTTP客户端,这使得在Web服务客户端的写入更加方便,它具有可插入注释支持.Feign还支持可插拔编码器和解码器,Spring Cloud增加了对Spring MVC注释的支持,并使用Spring Web中默认使用的HttpMessageConverters.Spring Cloud集成Ribbon和Eureka以在使用Feign时提供复制均衡的http客户端.
### Netflix Hystrix
&emsp;&emsp;Netflix 创建了一个名为Hystrix的库,它实现了断路器模式.形成了容错管理工具.旨在通过熔断机制服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力.  
&emsp;&emsp;在微服务架构中,通常有多层服务调用.如图:
![HystrixGraph](/img/in-post/SpringCloud/cloudimage/HystrixGraph.png)  
较低级别的服务在的服务故障可能导致用户级联故障.当对特定服务的调用大于断路器设定值(Hystrix中的默认值为5秒内20次故障)、失效百分比大于断路器设定值(默认>50%)和滚动窗口定义值(默认为10s)时电路打开,不进行调用.在出现错误和开路的情况下,开发人员可以提供回退,如下图:  
![HystrixGraph](/img/in-post/SpringCloud/cloudimage/HystrixFallback.png)  
回退可以是另一个Hystrix受保护的调用、静态数据或正常的空值.回退是可以级联链接的,因此第一个回退调用其他业务后又回退静态数据
### Netflix Zuul
&emsp;&emsp;Zuul是Netflix开发的一款路由器和过滤器.路由在微服务体系结构的一个组成部分.例如:/可以映射到Web应用程序,/api/users映射到用户服务,将/api/shop映射到商店服务.Zuul是Netflix的基于JVM的路由器和服务端负载均衡器.  
&emsp;&emsp;Netflix使用Zuul进行以下操作:认证、洞察、压力测试、金丝雀测试、动态路由、服务迁移、负载脱离、安全、静态响应处理和主动/主动流量管理  
&emsp;&emsp;Zuul的规则引擎允许规则和过滤器使用任何JVM语言编写,内置支持Java和Groovy
> 注:配置属性zuul.max.host.connections已经被另外两个属性所替代, zuul.host.maxTotalConnections默认值为200, zuul.host.maxPerRouteConnections默认值为20

### Spring Cloud Config
&emsp;&emsp;Spring Cloud Config为分布式系统中的外部化配置提供服务器和客户端支持.它包括Config Server和Config Client俩部分.由于Config Server和Config Client都实现了对Spring Environment和PropertySource抽象映射,因此,Spring Cloud Config非常适合Spring应用程序,也可与任何其他语言编写程序配合使用  
&emsp;&emsp;Config Server是一个可横向扩展、集中式的配置服务器,它用于集中管理应用程序各个环境下的配置,默认使用Git存储配置内容(也可使用Subversion、本地文件系统或Vault存储配置)  
&emsp;&emsp;Config Client是Config Server客户端,用于操作存储在Config Server中的配置属性  
### Spring Cloud Bus
&emsp;&emsp;Spring Cloud Bus将分布式系统的节点与轻量级消息代理链接.总线就像一个分布式执行器,用于扩展的Spring Boot应用程序,但也可以用作应用程序之间的通信通道.目前唯一的实现是使用AMQP代理作为传输  
&emsp;&emsp;Spring Cloud中的事件、消息总线,用于在集群(例如:配置变化事件)中传播状态变化,主要用于实现微服务之间的通信.可与Spring Cloud Config联合实现热部署  
&emsp;&emsp;当我们需要更新所有微服务的配置,我们希望在不重启的情况下去更新配置,就可以依赖Config和Bus,让所有服务都去订阅事件,当事件发生改变后就去通知微服务去更新它们内存中的配置信息.
### Spring Cloud Stream
&emsp;&emsp;Spring Cloud Stream是构建消息驱动的微服务应用程序的框架.Spring Cloud Stream基于Spring Boot建立独立的生产级Spring应用程序,并使用Spring Integration提供与消息代理的连接.它提供了来自几家供应商的中间件的意见配置,介绍了持久发布订阅语义,消费者组和分区的概念.目前仅支持RabbitMQ、Kafaka
### Spring Cloud Sleuth
&emsp;&emsp;一款日志收集工具包,封装了Dapper和log-based追踪以及Zipkin和HTrace操作,为Spring Cloud提供了分布式跟踪解决方案.
我们可以使用Sleuth完成以下问题:
- 解析串联调用链,快速定位问题
- 理清微服务之间的依赖关系
- 进行各服务接口的性能分析
- 跟踪业务流的处理顺序
### Spring Cloud Security
&emsp;&emsp;Spring Cloud Security提供了一组用于安全应用程序和服务的最小化工具包,可以从外部(或集中)进行大量配置的声明式模型有助于大型协作的远程组件系统,通常具有中央身份管理服务.在像CloudFoundry这样的服务平台上也很容易使用.  
&emsp;&emsp;它是基于Spring Boot和Spring安全性OAuth2,我们可以快速实现常见模式的系统,如单点登录、令牌中继和令牌交换.
### Spring Cloud Task
&emsp;&emsp;Spring Cloud Task主要解决短时微服务的任务管理、任务调度的工作,为在JVM上运行短时应用的需求提供基本的技术支持  
&emsp;&emsp;Spring Cloud Task需要数据库来存储task运行的结果,如果在无数据库的环境下,重启应用后所有环境变量都会消失.现主要支持的数据库:DB2、H2、HSQLDB、MySQL、Oracle、Postgres、SqlServer等
