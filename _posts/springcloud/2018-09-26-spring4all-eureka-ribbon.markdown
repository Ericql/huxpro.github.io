---
layout:       post
title:        "Eureka和Ribbon实现服务发现和客户端侧负载均衡"
subtitle:     "Eureka和Ribbon实现服务发现和客户端侧负载均衡"
date:         2018-09-23 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - SpringCloud
---
# Service Discovery and Client-Side Load Balancing With Eureka and Ribbon

原文链接：[https://dzone.com/articles/service-discovery-and-load-balancing-using-eureka](https://dzone.com/articles/service-discovery-and-load-balancing-using-eureka "Eureka实现服务发现和Ribbon的负载均衡")

作者：[Kaushal Gupta](https://dzone.com/articles/service-discovery-and-load-balancing-using-eureka)

译者：Eric

_想学习如何使用服务发现和客户端负载均衡来创建一个应用程序？查看此帖子以了解有关使用Eureka和Ribbon的更多信息_

![eureka](/img/in-post/SpringCloud/spring4all/eureka.png)

今天我们将构建一个具有负载均衡和服务发现的程序。传统意义上，一个负载均衡器是一个运行的独立应用程序，带有所有服务节点，同样。我们将构建一个应用程序，客户端不需要记住服务器的IP，而且我们可以动态添加和删除节点。

让我们从app1和app2开始，它们是两个符合REST风格接口的应用程序或服务器，例如上图的/process。我们将对其进行服务发现和负载均衡。app1和app2开始后，我们将其命名为app-producer并将它们注册到Eureka服务器上。

app-consumer这个应用程序是一个消费端的应用程序，它通过调用app-producer实现具体功能，作为eureka的消费端程序，它也要将自己注册到Eureka服务注册表中。

## Step-By-Step Code

当客户端发出一个请求(localhost:7070/process)到app-consumer应用程序，app-consumer会去请求Eureka服务器显示可用的实例。然后，Eureka会告诉app-consumer有两个可用的app-producer注册实例，因为客户端侧负载均衡需要添加到应用程序。

### Eureka Pom.xml

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>zuul</groupId>
  <artifactId>zuul</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>
    <name>SpringBootHelloWorld</name>
<description>Demo project for Spring Boot</description>
<parent>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>1.4.1.RELEASE</version>
<relativePath /> <!-- lookup parent from repository -->
</parent>
<properties>
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
<java.version>1.8</java.version>
</properties>
<dependencies>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
  <dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>3.10.3</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-client</artifactId>
    <version>3.10.3</version>
</dependency>
</dependencies>
<dependencyManagement>
<dependencies>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-dependencies</artifactId>
<version>Camden.SR6</version>
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

### Eureka app

```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
@SpringBootApplication
@EnableEurekaServer
public class Eureka {
  public static void main( String[] args ) {
		SpringApplication.run(Eureka.class, args);
    }   
}
```

### App1 pom.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<groupId>com.app1</groupId>
<artifactId>app1</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>
<name>app1</name>
<description>Demo project for Spring Boot</description>
<parent>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>1.4.1.RELEASE</version>
<relativePath/> <!-- lookup parent from repository -->
</parent>
<properties>
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
<java.version>1.8</java.version>
</properties>
<dependencies>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-test</artifactId>
<scope>test</scope>
</dependency>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
</dependencies>
<dependencyManagement>
<dependencies>
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-dependencies</artifactId>
<version>Camden.SR6</version>
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

### App1 application

```
package com.app1.app1;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
@SpringBootApplication
@EnableDiscoveryClient
public class App1Application {
public static void main(String[] args) {
SpringApplication.run(App1Application.class, args);
}
}
```

### App1 REST endpoint

```
package com.app1.app1;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
@Controller
public class AppController {
    @RequestMapping(value="/process", method = RequestMethod.GET)
    public @ResponseBody String  showLoginPage(ModelMap model){
        return "Processing aap1";
    }
}
```

### App1 properties

```
server.port=8091
eureka.client.serviceUrl.defaultZone=http://localhost:8080/eureka
spring.application.name=app-producer
eureka.instance.instanceId=${spring.application.name}:${random.int}
```

对于本教程，我不再赘述app2的创建，只需创建一个与app1相同的应用程序。唯一需要变化的是它的REST端点，它将返回"Processing aap2".

现在我们将开始app-consumer的编写:

### application code

```
package com.app2.app2;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
@SpringBootApplication
public class AppConsumer {
  public static void main(String[] args) {
  	SpringApplication.run(AppConsumer.class, args);
  }

  @Bean
  public  Client  client() {
  	return  new Client();
  }
}
```

### Rest endpoint with client-side balancing

```
package com.app2.app2;
import java.io.IOException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.client.RestClientException;
@Controller
public class AppController {
  @Autowired
  private Client client;

	@RequestMapping(value="/process", method = RequestMethod.GET)
    public @ResponseBody String  showLoginPage(ModelMap model) throws RestClientException, IOException{
        return client.getEmployee("/process");
    }
}
```

```
package com.app2.app2;
import java.io.IOException;
import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;
public class Client {
  @Autowired
  private LoadBalancerClient loadBalancer;
  
  public String getEmployee(String url) throws RestClientException, IOException {
      ServiceInstance serviceInstance=loadBalancer.choose("app-producer");
      System.out.println(serviceInstance.getUri());
      String baseUrl=serviceInstance.getUri().toString()+url;
      RestTemplate restTemplate = new RestTemplate();
      ResponseEntity<String> response = null;
      try {
          response = restTemplate.exchange(baseUrl, HttpMethod.GET, getHeaders(), String.class);
      } catch (Exception ex) {
          System.out.println(ex);
      }
      return response.getBody();
  }
  
  private static HttpEntity<?> getHeaders() throws IOException {
      HttpHeaders headers = new HttpHeaders();
      headers.set("Accept", MediaType.TEXT_PLAIN_VALUE);
      return new HttpEntity<>(headers);
  }
}
```

### application.properties

```
server.port=7070
eureka.client.serviceUrl.defaultZone=http://localhost:8080/eureka
spring.application.name=app-consumer
eureka.instance.instanceId=${spring.application.name}:${random.int}
```

最后，你需要启动所有的SpringBoot应用程序，然后发起请求(localhost:7070/process)

循环方式的负载策略输出如下:

```
Processing aap1

Processing aap2

Processing aap1
```