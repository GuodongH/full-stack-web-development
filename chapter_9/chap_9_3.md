# 监控服务和路由服务

微服务是一个分布式的架构模式，这样一个结构虽然带来更灵活的特性，但也会带来单一应用中不会出现的一些问题。当系统从单个节点拓展到多节点的时候，如果系统的某个点出现问题，这时的问题定位可能就会变成一个挑战，因为要定位这样的问题仅仅依赖传统的日志或者 debug 往往是不够的。

再有是当有新的业务加入或者业务更改之后，系统是否运行正常？系统的资源和性能是否正常？这样都可以通过监控手段进行衡量。出现这些问题后，监控就是一个常用的、有效的一个手段。

## Spring Boot Admin

Spring Boot Admin <http://codecentric.github.io/spring-boot-admin> 是一套用于监控基于 Spring Boot 的应用，它提供完整的的监控 UI 界面和多种监控指标，适合中小团队用于监控微服务的状态。由于其内置了很多常用的健康指标和统计，所以属于开箱即用的一个软件包，但如果你的团队需要更多定制化的功能，就需要定制化 SpringBoot Admin  ，它也提供了一些方法可以自定义视图。

Spring Boot Admin 集成到系统中主要有两中方式，一种方式是建立一个 Admin Server ，然后通过将每个需要被监控的微服务的配置文件指向 Admin Server 。由于 SpringBoot Admin 2.0 添加了对 Spring Cloud 的支持就，另一种方式是通过 Eureka Server，将 Admin Server 注册到 Eureka，Admin Server 就可以可以读取到服务中心所有注册的应用以便实现监控。

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

## 路由服务

路由服务是由于在系统中存在多个微服务的情况下，每个微服务都会有一系列的公开的 API 接口，这种分布式的状态当然对于每个微服务来说只专注于自身的业务逻辑会更清晰，但对于消费服务接口的客户端，比如前端或 App 客户端等就比较痛苦了。难道我们需要了解所有的微服务地址和其 API 才能调用吗？

如果我们把系统的各个微服务提供的 API 集成在一起，根据某种规则进行转发，这样消费端就可以只和这样一个集成的服务对话，而不需考虑服务架构的内部构建和部署方式了。

例如, `/` 可以映射到首页的前端应用，`/api/users` 映射到用户的微服务， `/api/shop` 映射到在线商铺的微服务。

要达成这样的效果，根据系统的规模，有几种可选方案：

* `nginx` 作为服务路由 -- 对于中小型规模的系统，我推荐这种方式，因为其配置相对简单、快速，而且效率上也没有太多损耗。
* 利用专有的路由服务 -- 比如 Netflix 团队开源的 `zuul` ，这种方式更专业，对于较复杂的路由规则有良好的支持，也可以方便的和 Spring Cloud 集成。

### Nginx 作为路由服务

使用 Nginx 作为路由是非常方便的，我们只需要定义系统中需要整合的微服务为上游 Server （ `upstream app_server` ），然后在 `server` 那段中设计不同的 `location` 规则，然后将对应请求路由到对应的上游 Server ，或者如果是静态资源，一般就 Nginx 自己处理就好。

下面就是一个典型的 Nginx 的配置文件

```nginx
upstream app_server {
  server api_server:8080;
}

server {
  listen 443;
  listen [::]:443;
  ssl on;
  ssl_certificate /etc/nginx/certs/server.crt;
  ssl_certificate_key /etc/nginx/certs/server.key;
  charset utf-8;

  location ~ ^/(api|management)/ {
    proxy_pass http://app_server;
    proxy_redirect     off;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Host $server_name;
    proxy_connect_timeout   10;
    proxy_send_timeout      15;
    proxy_read_timeout      20;
  }

  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html =404;
  }
}
```

在 Nginx 配置中的上游 Server 如果是使用 IP 的话就不太灵活，所以我们采用了主机名的方式。但主机名如果不使用 Docker 就需要自己配置，所以我们采用容器的方式避免直接配置主机名，将需要在 Nginx 中配置的服务使用同一个 network 。

```yml
version: '3.2'
services:
  elasticsearch:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/elasticsearch-wo-xpack:5.5.0
    volumes:
      - esdata:/usr/share/elasticsearch/data
      - ./docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - docker-app
  redis:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/redis:4-alpine
    command: [ "redis-server", "--protected-mode", "no" ]
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - docker-app
  mongo:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/mongo:3.6.4
    ports:
      - "27017:27017"
    volumes:
      - api_db:/data/db
    networks:
      - docker-app
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672" # JMS 端口
      - "15672:15672" # 管理端口 default user:pass = guest:guest
    networks:
      - docker-app
  configserver:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/gtm-config
    ports:
      - 8888:8888
    depends_on:
      - rabbitmq
    networks:
      - docker-app
  discovery:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/gtm-discovery
    ports:
      - 8761:8761
    depends_on:
      - rabbitmq
      - configserver
    networks:
      - docker-app
  api-server:
    image: registry.cn-beijing.aliyuncs.com/twigcodes/gtm-api-service
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    ports:
      - "8080:8080"
      - "5005:5005"
    links:
      - mongo
      - redis
      - elasticsearch
    networks:
      - docker-app
  nginx:
    build:
      context: .
      dockerfile: ./frontend/docker/nginx/Dockerfile
      args:
        - env=production
    container_name: nginx
    ports:
      - 80:80
      - 443:443
volumes:
  api_db: {}
  redis-data: {}
  esdata: {}
networks:
  docker-app:
    driver: bridge

```

当然这样做其实在扩展性方面还是有问题的，如果我们的容器不在同一宿主机，就又得采用 IP 方式，但是这样就缺乏了动态配置的优点。接下来就看一下使用路由服务的方式

### Zuul 路由服务

`Zuul` 是 Netflix 开发的一个基于 `JVM` 的路由组件，而且可以进行服务端负载均衡。

Zuul 提供了以下特性：

* 鉴权
* 压力测试
* 动态路由
* 服务迁移
* 安全
* 静态响应处理
* 流量管控

添加一个基于 Zuul 的路由服务需要新建一个 gradle 子项目 `gtm-gateway` 。接下来要为其添加一个依赖 `org.springframework.cloud:spring-cloud-starter-netflix-zuul`

```groovy
apply plugin: 'org.springframework.boot'
configurations {
    springLoaded
    // 如果使用 undertow 或 jetty 需要把默认包含的 tomcat 排除在外
    compile.exclude module: 'spring-boot-starter-tomcat'
}
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    implementation("org.springframework.cloud:spring-cloud-starter-bus-amqp")
    implementation("org.springframework.cloud:spring-cloud-starter-stream-rabbit")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-zuul")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}

```

然后在 `gtm-gateway/src/main/java/dev/local/gtm/gateway/Application.java` 添加 `@EnableZuulProxy` 这个注解。

```java
package dev.local.gtm.gateway;

import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * 路由网关服务器
 */
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
@RefreshScope
@Configuration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

然后我们就可以添加 Zuul 相关的配置了， Zuul 自带强大的反向代理功能。反向代理简单的理解就是让一个 HTTP 请求被转发到对应的服务。

下面的例子中，`zuul.ignored-services` 这个属性是决定哪些服务 ID 是不走反向代理的，但是如果一个服务在忽略服务的表达式中匹配了，但是在 `routes` 中列出了，那么这个忽略就是无效的了。也就是说下面的例子中虽然我们忽略的是所有服务，因为使用了通配符 `*` ，但由于在 `routes` 中列出了 `api` 这个定义，所以 `/api` 开头的请求（我们在 `zuul.routes.api.path` 中定义了 `/api/**` 这个路径）都会被转发到一个 ID 为 `api-service` 的服务。

Zuul 代理使用 `Ribbon` 通过 Eureka 定位该服务。所有的请求会在一个 `Hystrix` 命令中执行，所以如果发生了异常，这些失败信息就会在 `Hystrix` 中体现出来，此时我们的反向代理就不会再去联系目标服务了。

```yml
zuul:
  ignoredServices: '*'
  routes:
    api:
      path: /api/**
      serviceId: api-service
      stripPrefix: true
```

`stripPrefix` 这个属性是决定是否去掉前缀，也就是 `/api` ，我们设置的就是去掉 `api` 。举个例子，如果我们对路由网关发送这个请求 <http://localhost:8090/api/actuator/info> ，那么转发到 `api-service` 时，请求就变成了 <http://localhost:8080/actuator/info> ，去掉了 `/api` 这个路径前缀。

完整的 `gtm-gateway` 的 `application.yml` 现在看起来就是下面的样子。

```yml
server:
  port: 8090
eureka:
  client:
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
zuul:
  ignoredServices: '*'
  routes:
    api:
      path: /api/**
      serviceId: api-service
      stripPrefix: true
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
