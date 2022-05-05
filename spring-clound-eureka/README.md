# 注册中心Eureka

[http://www.ityouknow.com/springcloud/2017/05/10/springcloud-eureka.html](http://www.ityouknow.com/springcloud/2017/05/10/springcloud-eureka.html)

## 背景介绍

#### 服务中心

服务中心又称注册中心，管理各种服务功能包括服务的注册、发现、熔断、负载、降级等，比如dubbo admin后台的各种功能

#### eureka

1. Eureka由两个组件组成：Eureka服务器和Eureka客户端。
2. Eureka服务器用作服务注册服务器
3. Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持

**说明：**

1. Spring Cloud 封装了 Netflix 公司开发的 **Eureka** 模块来实现服务注册和发现。Eureka 采用了 C-S 的设计架构。
2. **Eureka Server** 作为服务注册功能的服务器，它是服务注册中心
3. 系统中的其他微服务，使用 **Eureka 的客户端**连接到 Eureka Server，并维持心跳连接

**角色：**

1. eureka server
    - 提供服务注册和发现
2. service provider
    - 服务提供方
    - 将自身服务注册到Eureka，从而使服务消费方能够找到
3. service Consumer
    - 服务消费方
    - 从Eureka获取注册服务列表，从而能够消费服务

service provider和serivce Consumer都是客户端

## 案例实践

#### [Eureka Server](./spring-cloud-eureka)

1. pom添加依赖

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2021.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2. 添加启动代码中添加@EnableEurekaServer注解

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}

}
```

3. 配置文件

```
spring:
  application:
    name: spring-cloud-eureka

server:
  port: 8000

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

- eureka.client.register-with-eureka ：表示是否将自己注册到Eureka Server，默认为true。
- eureka.client.fetch-registry ：表示是否从Eureka Server获取注册信息，默认为true。
- eureka.client.serviceUrl.defaultZone ：设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。

启动工程后，访问：http://localhost:8000/，可以看到下面的页面，其中还没有发现任何服务

## 集群

#### 双节点注册中心

1. 创建application-peer1.properties，作为peer1服务中心的配置，并将serviceUrl指向peer2

```
server:
  port: 8000

eureka:
  instance:
    hostname: peer1
  client:
    service-url:
      - http://peer2:8001/eureka/
```

2. 创建application-peer2.properties，作为peer2服务中心的配置，并将serviceUrl指向peer1

```
server:
  port: 8001

eureka:
  instance:
    hostname: peer2
  client:
    service-url:
      - http://peer1:8000/eureka/
```

3. 打包启动

```
#打包
mvn clean package
# 分别以peer1和peeer2 配置信息启动eureka
java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer1
java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer2
```

#### eureka集群使用

在生产中我们可能需要三台或者大于三台的注册中心来保证服务的稳定性，配置的原理其实都一样，将注册中心分别指向其它的注册中心。这里只介绍三台集群的配置情况，其实和双节点的注册中心类似，每台注册中心分别又指向其它两个节点即可，使用application.yml来配置。

application.yml配置：

```
spring:
  application:
    name: spring-cloud-eureka
  profiles: peer1
server:
  port: 8000
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2:8001/eureka/,http://peer3:8002/eureka/
---
spring:
  application:
    name: spring-cloud-eureka
  profiles: peer2
server:
  port: 8001
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8000/eureka/,http://peer3:8002/eureka/
---
spring:
  application:
    name: spring-cloud-eureka
  profiles: peer3
server:
  port: 8002
eureka:
  instance:
    hostname: peer3
  client:
    serviceUrl:
      defaultZone: http://peer1:8000/eureka/,http://peer2:8001/eureka/
```

分别以peer1、peer2、peer3的配置参数启动eureka注册中心。

```
java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer1
java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer2
java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer3
```

依次启动完成后，浏览器输入：http://localhost:8000/ 
