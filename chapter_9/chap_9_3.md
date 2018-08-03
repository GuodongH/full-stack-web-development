# 监控服务

微服务是一个分布式的架构模式，这样一个结构虽然带来更灵活的特性，但也会带来单一应用中不会出现的一些问题。当系统从单个节点拓展到多节点的时候，如果系统的某个点出现问题，这时的问题定位可能就会变成一个挑战，因为要定位这样的问题仅仅依赖传统的日志或者 debug 往往是不够的。

再有是当有新的业务加入或者业务更改之后，系统是否运行正常？系统的资源和性能是否正常？这样都可以通过监控手段进行衡量。出现这些问题后，监控就是一个常用的、有效的一个手段。

## Spring Boot Admin

SpringBoot Admin <http://codecentric.github.io/spring-boot-admin> 是一套用于监控基于 SpringBoot 的应用，它提供完整的的监控 UI 界面和多种监控指标，适合中小团队用于监控微服务的状态。由于其内置了很多常用的健康指标和统计，所以属于开箱即用的一个软件包，但如果你的团队需要更多定制化的功能，就需要定制化 SpringBoot Admin  ，它也提供了一些方法可以自定义视图。

SpringBoot Admin 集成到系统中主要有两中方式，一种方式是建立一个 Admin Server ，然后通过将每个需要被监控的微服务的配置文件指向 Admin Server 。由于 SpringBoot Admin 2.0 添加了对 Spring Cloud 的支持就，另一种方式是通过 Eureka Server，将 Admin Server 注册到 Eureka，Admin Server 就可以可以读取到服务中心所有注册的应用以便实现监控。

我们这里要做的是第二种方式，但稍稍有点区别，因为如果单独做一个 Admin Server 的话，感觉有点浪费资源，我们将 Admin Server 集成到刚刚建立的 Eureka Server 上。这样的组合也比较符合逻辑，系统所有的应用都会注册到 Eureka Server 上，我们在这个 Server 上监控也是顺理成章的。

### 构建 SpringBoot Admin Server

构建一个 Admin Server ，首先需要添加一个依赖 `de.codecentric:spring-boot-admin-starter-server`

```groovy
apply plugin: 'org.springframework.boot'
configurations {
    springLoaded
    // 如果使用 undertow 或 jetty 需要把默认包含的 tomcat 排除在外
    compile.exclude module: 'spring-boot-starter-tomcat'
}
dependencies {
    implementation("de.codecentric:spring-boot-admin-starter-server:${springbootAdminVersion}")
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    implementation("org.springframework.cloud:spring-cloud-starter-bus-amqp")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}

```

然后给应用添加注解 `@EnableAdminServer` 将服务配置成 Admin Server

```java
package dev.local.gtm.discovery;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * 服务发现服务器
 */
@SpringBootApplication
@EnableAdminServer
@EnableEurekaServer
@RefreshScope
@Configuration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

最后需要更改一下 `application.yml` ，第一个要更改的是要将 Admin Server 的 `context-path` 更改成一个除了 `/` 之外的路径，因为如果不更改 Eureka Server 和 Admin Server 就冲突了，都定义在 `/` 。下面的例子中我们把 Admin Server 的路径定义成 `/admin` ，也就是说 <http://localhost:8761/admin> 。

另一个要改动的配置是 `eureka.instance` 的续租间隔时间，健康查询路径以及 `eureka.client` 的拉取间隔时间。本来这些对于 Eureka Server 是不需要的，但是 Admin Server 需要拉取注册的信息以便可以监控各服务，所以这里我们需要改成下面的配置。

```yml
server:
  port: 8761
spring.boot.admin.context-path: /admin
eureka:
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
  client:
    registryFetchIntervalSeconds: 5
    serviceUrl:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS

---

spring:
  profiles: prod
eureka:
  instance:
    hostname: discovery
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
  client:
    registryFetchIntervalSeconds: 5
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```

这样做好一个 Admin Server 之后，其他我们什么都不用做了，直接重新启动 discovery 和 configserver 以及 api-service ，我们就可以在 <http://localhost:8761/admin> 看到下面的界面列出了 3 个服务的状态。

![Spring Boot Admin 的首页](/assets/2018-08-03-20-34-00.png)

我们点击 `API-SERVICE` 就可以查看该微服务的详细监控界面

![服务的详细监控界面](/assets/2018-08-03-20-36-33.png)

在这个详情界面我们除了可以看到上面的图片所示的健康信息之外，还可以下拉看到常见的 JVM 监控指标的统计，比如进程、线程、垃圾回收、堆内存、非堆内存等。

![常用的 JVM 监控指标](/assets/2018-08-03-20-37-51.png)

点击其他 tab 可以查看 Spring Boot Admin 提供的其他特性。总的来说 Spring Boot Admin 具有以下特性：

* 显示健康状态
* 显示详细的指标，比如
  * JVM & 内存测量
  * micrometer.io 的度量
  * 数据源测量
  * 缓存测量
* 显示构建版本信息
* 跟踪和下载日志
* 查看 `jvm` 的 `system-` 和 `environment-` 开头的属性
* 支持 Spring Cloud 的 `/env-` 和 `/refresh-` 开头的路径
* 简单的日志级别管理
* 和 JMX-beans 进行交互操作
* 查看线程 dump
* 查看 Http 跟踪
* 查看审计事件
* 查看 Http 路径
* 查看计划任务
* View and delete active sessions (using spring-session)
* View Flyway / Liquibase database migrations
* 下载内存堆 dump
* 状态更新通知 (通过 `e-mail`, `Slack`, `Hipchat` 等等)
* 状态变更的事件的记录

### 监控日志

默认情况下 Spring Boot Admin 是只能调节日志级别，但看不到日志本身，这个原因是 Spring Boot 如果没有设置 `logging.path` 或 `logging.file` 的时候，是不可能通过 `actuator` 取得日志信息的。如果我们希望在 Spring Boot Admin 看到日志的话，需要在对应的微服务的 `applicaiton.yml` 中指定 `logging.path` 或 `logging.file`

```yml
logging:
  level:
    org.apache.http: ERROR
    org.springframework:
      web: ERROR
      data: ERROR
      security: ERROR
      cache: ERROR
    org.springframework.data.mongodb.core.MongoTemplate: ERROR
    dev.local.gtm.api: ERROR
  file: /Users/wangpeng/workspace/logs/gtm-api.log
  file.max-history: 20
  pattern.file: "%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(%5p) %clr(${PID}){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n%wEx"

```

上面的配置文件中，我们指定了 `/Users/wangpeng/workspace/logs/gtm-api.log` 作为日志文件的路径。 `file.max-history: 20` 定义保存最近的多少天的日志文件。而 `pattern.file` 指定了日志的格式。这样我们就可以在 Spring Boot Admin 中直接看到日志了。

![查看日志](/assets/2018-08-03-22-33-07.png)
