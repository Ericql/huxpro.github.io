---
layout:       post
title:        "SpringCloud第二篇: 服务注册与发现Eureka"
subtitle:     "服务注册与发现Eureka"
date:         2018-10-08 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - SpringCloud
---
# SpringCloud第二篇: 服务注册与发现Eureka
## Eureka简介
&emsp;&emsp;在Spring Cloud Netflix技术栈中,Eureka作为服务注册中心对整个微服务架构起着最核心的整合作用;Eureka是Netflix开发的服务发现框架,本身是一个基于REST的服务,主要用于定位运行在AWS域中的中间层服务,以达到负载均衡和中间层服务故障转移的目的.SpringCloud将它集成在其子项目spring-cloud-netflix中,以实现SpringCloud的服务发现功能.
&emsp;&emsp;Eureka由两个组件组成:Eureka服务端和Eureka客户端.Eureka Server是服务注册功能的服务器,作为注册中心.Eureka Client是服务的客户端,用来于Server的交互,项目中的其他服务都是通过Eureka Client与Server保持心跳.
## Eureka配置
&emsp;&emsp;Eureka的配置主要是三类:分别是eureka的服务配置、eureka客户端配置和eureka实例配置.
eureka的服务器配置:
&emsp;&emsp;org.springframework.cloud.netflix.eureka.server.EurekaServerConfigBean实现com.netflix.eureka.EurekaServerConfig.所有属性前缀为eureka.server
eureka的客户端配置:
&emsp;&emsp;org.springframework.cloud.netflix.eureka.EurekaClientConfigBean实现com.netflix.discovery.EurekaClientConfig.所有属性前缀为eureka.client
eureka的实例配置:
&emsp;&emsp;org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean实现(间接)com.netflix.appinfo.EurekaInstanceConfig.所有属性前缀为eureka.instance
### Eureka实例和客户端
&emsp;&emsp;不同于实例,eureka客户端是连接服务端,对客户端本身的服务发现、允不允许服务端搜索等配置,而实例则相当于服务,一个Java项目,Eureka的配置共有100多项,举下常用的,不一一说明,可查看源码中对应的bean都有解释.
eureka.server.enable-self-preservation  // Server的自我保护

eureka.client.registerWithEureka=false // 是否向服务注册中心注册自己
eureka.client.fetchRegistry=false // 是否检索服务
这样当client服务不想注册到Server端,可以采取这样的配置即可

eureka.instance.instance-id // 实例注册到eureka服务端的唯一的实例ID,其组成为${spring.application.name}:${spring.application.instance_id:${random.value}}

&emsp;&emsp;一个Eureka实例通常由eureka.instance.instanceId进行区分,若不存在的情况下将使用eureka.instance.metadataMap.instanceId进行识别.实例间彼此使用eureka.instance.appName来进行查找,Spring Cloud默认使用spring.application.name,appName未定义情况下是UNKOWN.
&emsp;&emsp;当registerWithEureka是true,一个实例使用给定的URL向Eureka服务器注册,然后它每隔30s(可配置eureka.instance.leaseRenewallntervallnSecond)发送心跳.如果服务器没有收到心跳,则eureka.instance.leaseExpirationDurationInSeconds在从实例注册表中删除实例之前等待90秒(可配置),并禁止该实例的流量.发送心跳是一个异步任务,如果操作失败,它将按指数次数2倍,直到eureka.instance.leaseRenewalIntervalInSeconds * eureka.client.heartbeatExecutorExponentialBackOffBound达到最大延迟.对Eureka注册的重试次数没有限制
&emsp;&emsp;当eureka.client.fetchRegistry是true,客户端在启动时获取Eureka服务器注册表,并在本地缓存.从那时起它只是获取增量(这可以通过设置来关闭eureka.client.shouldDisableDelta=false,尽管这会是浪费带宽).注册表提取是每30秒调度一个异步任务(可配置eureka.client.registryFetchIntervalSeconds).如果操作失败,它将按指数级数2倍,直到eureka.client.registryFetchIntervalSeconds * eureka.client.cacheRefreshExecutorExponentialBackOffBound达到最大延迟.获取注册表信息的重试次数没有限制
## Eureka原理
### Eureka架构图解析
Eureka架构图
上图是eureka官方架构图,是基于集群配置的eureka,主要是三个模块:
- Eureka Server:提供服务注册和发现,不同节点上的Eureka通过Replicate进行数据同步
- Application Server:服务提供者,将自身服务注册到Eureka
- Application Client:服务消费者,从Eureka获取注册服务表,从而能够消费服务.

&emsp;&emsp;Make Remote Call简单理解为调用RestFul的接口,us-east-1c、us-east-1d、us-east-1e等都是zone,它们都属于us-east-1这个region.
### Eureka流程:
&emsp;&emsp;当服务启动后向Eureka Server注册,Eureka Server会将注册信息向其他Eureka Server进行同步.当服务消费者要调用服务提供者,则向Eureka Server获取服务提供者地址,然后将服务注册表缓存在本地,下次调用时,则直接从本地缓存中获取完成调用
&emsp;&emsp;在应用启动后,将会向Eureka Server发送心跳(默认周期为30秒).如果在Eureka Server在多个心跳周期内没有接收到某个节点的心跳,Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)
### Eureka自我保护机制
> eureka.server.enable-self-preservation=true

&emsp;&emsp;当Eureka Server在短时间丢失过多的客户端(发生故障)时,那么节点将进入自我保护模式,不再注销任何微服务.当故障恢复后,Eureka Server将退出微服务.
&emsp;&emsp;当然也可以手动配置关闭自我保护(eureka.server.enable-self-preservation为false),此时Eureka Server如下图:

## Eureka单个示例
### 新建Maven工程
&emsp;&emsp;新建一个Maven多模块工程,作为parent项目,用来给各个子模块继承.其pom为父pom,起到版本控制的作用.其pom文件如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.eric</groupId>
    <artifactId>ecloud</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>ecloud</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <modules>
        <module>eurekaserver</module>
        <module>eurekaclient</module>
    </modules>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
### 编写eureka server模块
- 新建一个maven模块作为服务注册中心,其pom文件如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.eric</groupId>
    <artifactId>eurekaserver</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>eurekaserver</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.eric</groupId>
        <artifactId>ecloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
- 在EurekaServerApplication启动类上添加注解@EnableEurekaServer
```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
- application.yml添加eureka的配置
```
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
spring:
  application:
    name: eureka-server
```
### 编写eureka client模块
- 与server创建相似,其pom.xml如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.eric</groupId>
    <artifactId>eurekaclient</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>eurekaclient</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.eric</groupId>
        <artifactId>ecloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
- 启动类添加@EnableEurekaClient来表明是一个EurekaClient
```
@SpringBootApplication
@EnableEurekaClient
public class EurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```
- application.yml添加配置
```
server:
  port: 8762
spring:
  application:
    name: eureka-client
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```
### 整体代码测试
&emsp;&emsp;启动server和client后,打开http://localhost:8761.即可看到EUREKA-CLIENT实例已经注册到server
## Eureka集群示例
&emsp;&emsp;同上的eureka server,将application.yml修改如下:
```
spring:
  application:
    name: eureka-ha
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2:8762/eureka/
server:
  port: 8761

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/
server:
  port: 8762
```
此时直接启动会报错,需要在C:/window/System32/driver/etc/hosts文件中添加映射,
> 127.0.0.1	peer1 peer2

这样就完成了eureka的高可用