---
title: "再学 Spring Cloud"
tags: ["SpringCloud"]
date: "2022-06-03T00:00:00+08:00"
toc: true
mermaid: true
displayTOCPlace: "left"
---

**技术选型**

|                |        Dubbo        |       SpringCloud        |    SpringCloudAlibaba    |
| :------------: | :-----------------: | :----------------------: | :----------------------: |
|    注册中心    |      ZooKeeper      |      Eureka、Consul      |      Nacos、Eureka       |
|  服务远程调用  |     Dubbo 协议      |     Feign(Http 协议)     |       Dubbo、Feign       |
|    配置中心    |         无          |    SpringCloudConfig     | SpringCloudConfig、Nacos |
|    服务网关    |         无          | SpringCloudGateway、Zuul | SpringCloudGateway、Zuul |
| 服务监控和保护 | dubbo-admin，功能弱 |          Hystix          |         Sentinel         |

**微服务拆分的要求**

* 服务和服务都必须有各自的数据库，相互独立
* 服务都对外暴露 Restful 的接口
* A 服务如果需要查询 B 服务信息，只能调用用户 B 服务的 Restful 接口，不能查询 B 的数据库

**添加依赖**[^1]

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Hoxton.SR10</spring-cloud.version>
    <mysql.version>5.1.47</mysql.version>
    <mybatis.version>2.1.1</mybatis.version>
</properties>

<dependencyManagement>
    <dependencies>
        <!-- springCloud -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- mysql驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>
        <!-- mybatis -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${mybatis.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Eureka 注册中心

最广为人知的注册中心就是 Eureka，其基本原理如下：

![image-20220605195955492](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/202206052323872.png "注册中心基本原理")

### 搭建注册中心

引入 server 依赖，并在启动类上添加 `@EnableEurekaServer` 注解

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

配置文件

```yaml
server:
  port: 10086
spring:
  application:
    name: eureka-server
eureka:
  client:
    service-url: 
      defaultZone: http://127.0.0.1:10086/eureka
```

其中 `default-zone` 是因为前面配置类开启了注册中心所需要配置的 Eureka 的**地址信息**，因为 Eureka 本身也是一个微服务，这里也要将自己注册进来，当后面 Eureka **集群**时，这里就可以填写多个，使用 “,” 隔开。

### 服务注册

添加 client 依赖，并在启动类上添加 `@EnableEurekaServer` 注解

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

配置文件

```yaml
spring:
  application:
    name：orderservice
eureka:
  client:
    service-url: 
      defaultZone: http:127.0.0.1:10086/eureka
```

### 服务拉取

配置 RestTemplate，并添加 `@LoadBalanced` 开启负载均衡机制

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```



![image-20220605204842308](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/202206052323239.png "负载均衡流程")

**源码跟踪**

为什么我们只输入了 service 名称就可以访问了呢？为什么不需要获取 ip 和端口，这显然有人帮我们根据 service 名称，获取到了服务实例的ip和端口。它就是`LoadBalancerInterceptor`，这个类会在对 RestTemplate 的请求进行拦截，然后从 Eureka 根据服务 id 获取服务列表，随后利用负载均衡算法得到真实的服务地址信息，替换服务 id。

我们进行源码跟踪：

![image-20220605224628891](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220605224628891.png)

这里的 `intercept()` 方法，拦截了用户的 HttpRequest 请求，然后做了几件事：

- request.getURI()：获取请求 uri，即 `http://user-service/user/8`
- originalUri.getHost()：获取 uri 路径的主机名，其实就是服务 id `user-service`
- this.loadBalancer.execute()：处理服务 id，和用户请求

可以看到获取服务时，通过一个 `getServer()` 方法来做负载均衡:

![image-20220605231553345](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220605231553345.png)

- getLoadBalancer(serviceId)：根据服务id获取 `ILoadBalancer`，而 ILoadBalancer 会拿着服务 id 去 eureka 中获取服务列表。
- getServer(loadBalancer)：利用内置的负载均衡算法，从服务列表中选择一个。在图中**可以看到获取了 8000 端口的服务**

继续跟踪源码 `chooseServer()` 方法，发现这么一段代码：

![image-20220605232006382](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220605232006382.png)

![image-20220605232158342](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220605232158342.png)

![image-20220605232235140](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/image-20220605232235140.png)

> 负载均衡默认使用了轮训算法，当然我们也可以自定义。



[^1]: SpringBoot 和 SpringCloud 的版本必须对应，具体参考： https://spring.io/projects/spring-cloud 



