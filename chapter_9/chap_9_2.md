# 配置服务和发现服务

## 配置中心是什么？

一个应用运行起来依赖的不仅仅是代码，还需要连接资源以及对于某些资源或业务进行参数化的配置。这些配置一般来说我们经常会采用外部设置的配置文件去调整，如切换不同的数据库，设置功能开关等。

随着系统微服务的不断增加，每个微服务都有自己的配置文件，这种各自管各自的配置管理模式，在开发时没什么问题，部署到生产环境之后管理就会很头疼，到了要大规模更新就更烦了。在这种情况下，如果我们可以统一配置中心就是一个比较好的解决方案，下图就是一个配置中心的解决方案：

![Spring Cloud Config](/assets/2018-08-01-15-56-58.png)

Spring Cloud Config 就是这样一个中心配置服务

* 提供服务端和客户端支持
* 集中式管理分布式环境下的应用配置
* 基于 Spring 环境，无缝 与Spring 应用集成
* 可用于任何语言开发的程序
* 默认实现基于 git 仓库，可以进行版本管理
* 可自定义实现

Spring Cloud Config 的服务端叫做 Spring Cloud Config Server ，提供以下功能

* 拉取配置时更新 `git` 仓库副本，或指定分支、标签等
* 支持数据结构丰富，支持 `yml` , `json` , `properties` 等
* 配合 Eureka 可实现服务发现，配合 Spring Cloud Bus 可实现配置推送更新
* 配置存储基于 `git` 仓库，可进行版本管理
* 简单可靠，有丰富的配套方案

### 配置中心服务器

在基于 Spring Cloud 的项目中使用建立一个 Spring Cloud Config Server 非常简单，我们首先建立一个子工程 `gtm-config` ，然后配置 `settings.gradle` ，在项目中添加子工程：

```groovy
include 'gtm-api'
include 'gtm-config'
rootProject.name = 'gtm-backend'
```

接下来需要给子工程配置依赖，新建一个 ``gtm-config/build.gradle`

```groovy
apply plugin: 'org.springframework.boot'
configurations {
    springLoaded
    // 如果使用 undertow 或 jetty 需要把默认包含的 tomcat 排除在外
    compile.exclude module: 'spring-boot-starter-tomcat'
}
dependencies {
    implementation("org.springframework.cloud:spring-cloud-config-server")
    implementation("org.springframework.cloud:spring-cloud-config-monitor")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}
```

对于 Spring Cloud Config Server 来说，`org.springframework.cloud:spring-cloud-config-server` 这个依赖是必须的。然后在 `gtm-config/src/main/java/dev/local/gtm/configserver` 中建立一个新的 `Application.java` 文件。

```java
package dev.local.gtm.configserver;

import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * GatewayApplication
 */
@SpringBootApplication
@EnableConfigServer
@Configuration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

上面的 `@EnableConfigServer` 这个注解就是让这个应用成为配置中心了。剩下的事情就是在 `application.yml` 中定义配置文件的存储位置，此处我们使用了 git 仓库，在调试代码时，也可以使用本地路径。但一般在生产环境中都会使用 git 仓库作为配置文件存储，因为这样可以保留版本的变更记录，在遇到问题时可以非常方便的回滚。

```yml
spring:
  application:
    name: configserver
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/twigcodes_group/smartoffice-config.git
          username: 请填写自己的用户名
          password: 请填写自己的密码
server:
  port: 8888
logging:
  level:
    org.springframework:
      cloud: DEBUG
---

spring:
  profiles: prod
```

如果我们此时将 `gtm-api/src/main/resources/application.yml` 放到上面配置的 git 仓库中，请注意文件名需要和 `spring.application.name` 一致，也就是说在 git 仓库中 `gtm-api` 这个项目的配置文件应该叫做 `api-service.yml` ，因为  `gtm-api/src/main/resources/application.yml` 中的 `spring.application.name` 是 `api-service`。此时，如果我们访问 <http://localhost:8888/api-service/dev> 可以看到在开发环境中的配置。

![配置服务器可以读取配置文件](/assets/2018-08-01-20-39-20.png)

### 配置中心客户端

所哟希望从配置中心服务器取得配置文件的应用都是配置中心的客户端，客户端配置起来就非常简单，只需在子项目的 `build.gradle` 中先添加依赖：

```groovy
implementation("org.springframework.cloud:spring-cloud-starter-config")
```

然后在 `src/main/resources` 目录中新建一个 `bootstrap.yml` 。此处需要注意的一个地方是，默认的配置中心服务的 `ServiceId` 是 `configserver` ，但这个可以在 `bootstrap.yml` 中指定 `spring.cloud.config.discovery.serviceId` 为 Config Server 的 `spring.application.name` 。

```yml
spring:
  application:
    name: api-service
  cloud:
    config:
      uri: http://localhost:8888
---
spring:
  profiles: prod
  cloud:
    config:
      uri: http://configserver:8888

```

此时如果启动客户端的话，我们会在 `console` 中看到客户端从服务器（ <http://localhost:8888> ）拉取了配置文件。

```txt

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.3.RELEASE)

2018-08-02 09:13:27.150  INFO 17027 --- [  restartedMain] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888
2018-08-02 09:13:27.197 DEBUG 17027 --- [  restartedMain] o.s.web.client.RestTemplate              : Created GET request for "http://localhost:8888/api-service/default"
2018-08-02 09:13:27.342 DEBUG 17027 --- [  restartedMain] o.s.web.client.RestTemplate              : Setting request Accept header to [application/xml, text/xml, application/json, application/x-jackson-smile, application/cbor, application/*+xml, application/*+json]
2018-08-02 09:13:28.548 DEBUG 17027 --- [  restartedMain] o.s.web.client.RestTemplate              : GET request for "http://localhost:8888/api-service/default" resulted in 200 (OK)
2018-08-02 09:13:28.570 DEBUG 17027 --- [  restartedMain] o.s.web.client.RestTemplate              : Reading [class org.springframework.cloud.config.environment.Environment] as "application/json;charset=UTF-8" using [org.springframework.http.converter.json.MappingJackson2HttpMessageConverter@582fbb48]
2018-08-02 09:13:28.638  INFO 17027 --- [  restartedMain] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=api-service, profiles=[default], label=null, version=ecd523ebee6f04ff11bc67c04874ae3ea006bed8, state=null
2018-08-02 09:13:28.639  INFO 17027 --- [  restartedMain] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource {name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://gitee.com/twigcodes_group/smartoffice-config.git/api-service.yml (document #0)'}]}
2018-08-02 09:13:28.772  INFO 17027 --- [  restartedMain] dev.local.gtm.api.Application            : The following profiles are active: dev
```

## 发现服务

由于采用了微服务架构，所以当微服务间互相调用时，我们都要知道被调用方的 IP 和服务端口，或者域名和服务端口。如果是采用域名的情况下，我们可以利用基于 DNS 的服务发现，但因为 DNS 有缓存、无法自治等因素，这种方案就有很多局限。传统的 DNS 方式，都是通过 nginx 或者其他代理软件来实现，物理机器的 IP 和端口都是固定的，那么 nginx 中配置的服务 IP 和端口也是固定的，服务列表的更新只能通过手动来做，但如果后端服务很多时，手动更新容易出错，效率也很低，这在后端服务发生故障时，不可用时间就可能会加长。

在微服务中，尤其是使用了 Docker 等虚拟化技术的微服务，其 IP 都是动态分配的，服务实例数也是动态变化的，那么就需要精细而准确的服务发现机制。当微服务应用启动后，告诉一个服务发现服务器自己的 IP 和端口，这里的服务发现功能可以利用 Netflix 出品的 Eureka Server 和 Eureka Client 来配合实现。这样微服务架构内的服务都可以彼此了解对方的 IP 和端口以及哪些服务是在线的、可用的，也方便去做负载均衡和调用。

Eureka 是 Netflix 的一个开源的服务发现框架，主要用于服务的注册发现。Eureka 由两个组件组成：Eureka Server 和 Eureka Client 。 Eureka Server是一个服务注册中心，为服务实例注册管理和查询可用实例提供了 REST API ，并可以用其定位、负载均衡、故障恢复后端服务的中间层服务。在服务启动后 Eureka Client 用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。 Eureka Client 向服务注册中心注册服务同时会拉去注册中心注册表副本；在服务停止的时候，Eureka Client 向服务注册中心注销服务；服务注册后，Eureka Client 会定时的发送心跳来刷新服务的最新状态。

客户端发现模式的优点是服务调用、负载均衡不需要和 Eureka Server 通信，直接使用本地注册表副本，因此 Eureka Server 不可用时是不会影响正常的服务调用，性能也不会因为网络延迟和服务端延迟受到影响。但其缺点也很明显，但某个服务不可用时，各个 Eureka Client 不能及时的知道，需要 1~3 个心跳周期才能感知，但是，由于基于 Netflix 的服务调用端都会使用 Hystrix 来容错和降级，当服务调用不可用时 Hystrix 也能及时感知到，通过熔断机制来降级服务调用，因此弥补了基于客户端服务发现的时效性的缺点。

Eureka Server 采用的是对等通信( `P2P` )，无中心化的架构，没有 `master/slave` 的区分，每一个 Server 都是对等的，既是 Server 又是 Client ，所以其集群方式可以自由发挥，可以各点互连，也可以链式连接。 Eureka Server 通过运行多个实例以及彼此之间互相注册来提高可用性，每个节点需要添加一个或多个有效的 `ServiceUrl` 指向另一个节点。

### 区域与可用区

区域（ `Region` ）: 类似 AWS 或阿里云这种公有云服务在全球不同的地方都有数据中心，比如北美、南美、欧洲和亚洲等。与此对应，根据地理位置我们把某个地区的基础设施服务集合称为一个区域。通过区域，一方面可以使得云服务在地理位置上更加靠近我们的用户，另一方面使得用户可以选择不同的区域存储他们的数据以满足法规遵循方面的要求。

可用区（ `Zone` ）: 每个区域一般由多个可用区（ `Available Zone` ）组成，而一个可用区一般是由多个数据中心组成。可用区设计主要是为了提升用户应用程序的高可用性。因为可用区与可用区之间在设计上是相互独立的，也就是说它们会有独立的供电、独立的网络等，这样假如一个可用区出现问题时也不会影响另外的可用区。在一个区域内，可用区与可用区之间是通过高速网络连接，从而保证有很低的延时。

![Eureka 区域与可用区](/assets/2018-08-02-16-42-14.png)

从上面的架构图可以看出，主要有三种角色：

* Eureka Server - 通过 Register, Get，Renew 等接口提供注册和发现
* Application Service（服务提供方）：把自身服务实例注册到 Eureka Server
* Application Client（服务调用方）：通过Eureka Server 获取服务实例，并调用 Application Service

他们主要进行的活动如下：

* 每个区域有一个 Eureka 集群，每个区域中都至少有一个 Eureka Server。
* Service 作为一个 Eureka Client，注册到 Eureka Server ，并且通过发送心跳的方式更新租约。如果 Eureka Client 到期没有更新租约，那么过一段时间后， Eureka Server 就会移除该 Service 实例。
* 当一个 Eureka Server 的数据改变以后，会把自己的数据同步到其他 Eureka Server 。
* Application Client 也作为一个 Eureka Client 从 Eureka Server 中获取 Service 实例信息，然后直接调用 Service 实例。
* Application Client 调用 Service 实例时，可以跨可用区调用。

### 建立一个 Eureka Server

我们首先建立一个子工程 `gtm-config` ，然后配置 `settings.gradle` ，在项目中添加子工程：

```groovy
include 'gtm-api'
include 'gtm-discovery'
include 'gtm-config'
rootProject.name = 'gtm-backend'
```

接下来需要给子工程配置依赖，新建一个 ``gtm-discovery/build.gradle` ，请注意我们这里同时将 Eureka Server 配置成 Config Server 的客户端，所以除了添加 `org.springframework.cloud:spring-cloud-starter-netflix-eureka-server` 这个依赖之外，还需要添加 `org.springframework.cloud:spring-cloud-starter-config` 。

```groovy
apply plugin: 'org.springframework.boot'
configurations {
    springLoaded
    // 如果使用 undertow 或 jetty 需要把默认包含的 tomcat 排除在外
    compile.exclude module: 'spring-boot-starter-tomcat'
}
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}
```

然后在 `gtm-discovery/src/main/java/dev/local/gtm/discovery` 中建立一个新的 `Application.java` 文件。注解 `@EnableEurekaServer` 即可配置该应用成为 Eureka Server 。

```java
package dev.local.gtm.discovery;

import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * 服务发现服务器
 */
@SpringBootApplication
@EnableEurekaServer
@RefreshScope
@Configuration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

最后我们需要添加两个配置文件 `applicaiton.yml` 和 `bootstrap.yml` 。其中 `bootstrap.yml` 中需要指定 Config Server 的 `uri` ：

```yml
spring:
  application:
    name: discovery
  cloud:
    config:
      uri: http://localhost:8888

---

spring:
  profiles: prod
  cloud:
    config:
      uri: http://configserver:8888

```

而 `application.yml` 中我们需要禁用 Eureka Client 的行为，因为我们是一个 Server ，此处还需要配置 `eureka.instance.hostname` ，我们在开发环境下使用 `localhost` 而在生产环境使用 `discovery` 作为服务发现服务器的主机名。

```yml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

---

spring:
  profiles: prod
eureka:
  instance:
    hostname: discovery
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```

### 配置 Eureka Client

配置 Eureka Client 的过程更为简单一些，我们拿 `gtm-config` 项目作为示例为其添加 Eureka Client 特性，这样 Config Server 可以注册到服务发现上，在 `build.gradle` 中加上 `org.springframework.cloud:spring-cloud-starter-netflix-eureka-client` 这个依赖

```groovy

apply plugin: 'org.springframework.boot'
configurations {
    springLoaded
    // 如果使用 undertow 或 jetty 需要把默认包含的 tomcat 排除在外
    compile.exclude module: 'spring-boot-starter-tomcat'
}
dependencies {
    implementation("org.springframework.cloud:spring-cloud-config-server")
    implementation("org.springframework.cloud:spring-cloud-config-monitor")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.cloud:spring-cloud-starter-stream-rabbit")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}
```

然后在 `Application.java` 中加上 `@EnableDiscoveryClient` 这个注解

```java
package dev.local.gtm.configserver;

import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
@Configuration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

在 `application.yml` 中添加 `eureka.client.serviceUrl.defaultZone` 指向刚才的 Eureka Server

```yml
spring:
  application:
    name: configserver
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/twigcodes_group/smartoffice-config.git
          username: 请输入你的用户名
          password: 请输入你的密码
server:
  port: 8888
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

---

spring:
  profiles: prod
eureka:
  instance:
    hostname: discovery
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8761/eureka/

```

这样配置好之后，可以启动 Config Server 和 Eureka Server ，可以开启两个 terminal ，分别敲入下面两条命令

```bash
./gradlew :gtm-config:bootRun
```

```bash
./gradlew :gtm-discovery:bootRun
```

然后访问 <http://localhost:8761/> 就可以看到 `CONFIGSERVER` 已经注册到 Eureka 服务上了。

![配置 Config Server 使用发现服务](/assets/2018-08-02-20-26-11.png)

这里值得指出的是， Config Server 和 Eureka Server 是互相依赖的， Eureka Server 的配置需要到 Config Server 中获取，而 Config Server 也想要注册到 Eureka Server 。但这个不会变成死循环，因为如果没有找到配置中心服务器，会使用本地配置文件，而如果找不到服务发现服务器，会在启动后不断以一定间隔去查找。
