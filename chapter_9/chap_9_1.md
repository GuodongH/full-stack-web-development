# 微服务的体系架构

什么是微服务呢？个人理解是微服务允许通过一系列可相互协作的组件来构建一个大型系统，这些组件就是微服务。一个完整的应用可以垂直拆分成多个不同的服务，每个服务都能独立部署、独立维护、独立扩展，服务与服务间通过诸如 RESTful API 或者消息的发布/订阅机制互相调用。

在我们开始介绍微服务之前，回顾一下单一应用（ monolithic application ）是什么样子吧。一个完整的企业级应用通常被构建成三个主要部分：

1. 前端用户界面 -- 由运行在 PC 浏览器中的 HTML 页面、Javascript 和 CSS 组成
2. 数据库 -- 由许多的表构成一个通用的、相互关联的数据管理系统
3. 服务端应用 -- 服务端应用处理 HTTP 请求，执行业务逻辑，增删改查数据库中的数据，将结果渲染成适当的 HTML 视图发送给浏览器。

这样一个应用是一个一体化的、完整的、大而全的应用，任何对系统的改变都需要到重新构建和部署一个新版本的服务端应用程序。

这样的单一应用是一种构建系统很自然的方式。虽然开发人员可以把应用程序模块化的组织成类、函数、命名空间等，但所有处理请求的逻辑都运行在一个单独的进程中。从拓展性上来说，这种架构可以使用横向扩展，通过负载均衡将多个应用部署到多台服务器上，也可以采用数据库的横向或纵向切分，实现数据库的拓展。这样的开发模型曾经在相当长一段时间内非常成功，但是随着云服务的流行，在云中部署时，我们只是变更应用程序的一小部分，却需要进行整个重新构建和部署。随着功能的增加更新，时间一长，我们就很难再保持一个好的模块化结构，使得一个模块的变更很难不影响到其它模块。扩展就需要整个应用程序的扩展，而不能进行部分扩展。

这种问题的持续出现自然而然的催生了微服务架构的流行：把应用程序构建为一套服务。服务可以独立部署和扩展，每个服务都有各自的边界，不同的服务可以用不同的编程语言编写，使用不同的数据库，可以让不同的团队编写维护。

## 服务即组件

组件（ `component` ）这个概念在客户端和前端比较普遍一些，但其实整个软件行业一直希望由组件构建系统，就像在其他工业领域我们看到那样，从简单的家具到复杂精密的飞机，从由几十个组件构成到由上百万组件构成，我们一直希望一个复杂的软件系统可以通过 n 个设计精良的组件构成。

微服务架构中的一个重要理念是把服务当成组件，而不是作为库（ `library` ）。因为如果应用程序是由若干库构成的话，那么对任何一个库的改变都需要重新部署整个应用程序。但是如果我们把应用程序拆分成很多组件，那你只需要重新部署那个改变的组件。而且服务组件化之后会有更清晰的接口和更明确单一的职责。

但是需要指出的是，微服务框架不是只有优点而没有缺点的，远程调用会比进程内调用消耗更多资源，也就是说微服务架构比单一应用会消耗更多资源，在组织上也会更复杂。但是随着应用复杂度的提升，微服务架构的威力也会逐渐显现。

![典型的一个微服务架构](/assets/2018-08-01-11-32-05.png)

## 微服务架构下的组织机构变化

一个技术团队的传统组织架构是按照前端、客户端、服务端、数据库、 UI/UE 和测试来组织团队的构成。这样一个架构在过去几十年也确实体现出强大的威力。但在今天这个节奏越来越快的时代里，这样的组织架构却显得臃肿和效率低下。一个大型团队，按业务线来分割自己会导致很多的依赖。大型的应用中每个模块的业务可能都很复杂，在单一应用中，如果问题或需求跨越很多模块边界的话，需要协调多个大团队，这种协调和沟通的成本是高昂的，也就意味着短期内修复它们是很困难的。

而微服务架构下，组织机构会变成一个个麻雀虽小五脏俱全的小团队，由于微服务有自己的边界，遇到问题的解决基本在小团队内部即可快速对应。

![微服务下的组织机构](/assets/2018-08-01-11-46-21.png)

## 产品化服务

很多公司，尤其是国内的软件公司，大部分都是接到一个项目需求，从需求分析开始逐步的进行设计、开发、测试、交付这样的流程。而且交付完成之后，一般团队都会解散了，大家各自被分配到新的项目组中。

但微服务提倡以产品的角度来去做这些事情，而不是项目的角度。团队应该负责产品的整个生命周期，这要求开发者每天都关注他们的软件运行如何，增加更用户的联系，同时承担一些售后支持。

产品的理念，跟业务能力联系起来。不是着眼于完成一套功能的软件，这样一个持续产品生命周期的团队，是能够帮助软件及其用户提升业务能力的。

## 持续集成和持续发布

基础设施自动化技术在近年来得到了长足的发展：云计算减少了构建、发布、运维微服务的复杂性。

使用微服务架构的产品或者系统，一般都采用持续部署 （ Continuous Delivery 简称 CD ）和持续集成（ Continuous Integration 简称 CI ）。团队使用这种方式构建软件致使更广泛的依赖基础设施自动化技术。下图说明这种构建的流程：

![使用 Jenkins 进行持续部署的流程](/assets/2018-08-01-12-09-41.png)

## 监控和报警

微服务带来的不仅仅是优点，有一些天生的缺点会一并送给我们。这些不好的后果之一就是，使用服务作为组件的话，组件就可能出错，但整个系统不能因为某一个组件的出错就整体停摆。任务服务都可能发生故障，有可能是第三方服务的原因，也有可能是我们某个微服务的原因，那么应用需要尽可能的优化这种场景的响应。跟单一应用相比，这是一个明显缺点，因为它带来的额外的复杂度。这将让微服务团队时刻的想到服务故障的情况下用户的体验。

这就要求我们可以对微服务进行快速故障检测，更进一步如果可以做到自动恢复变更就非常重要了。微服务应用把实时的监控放在应用的各个阶段中，检测构架元素和业务相关的指标。监控系统可以提供一种早期故障告警系统，让开发团队跟进并调查。

所以在微服务构架中，服务的监控和报警是必须的，一般我们会监控和记录每个服务的配置，上/下线状态、各种运维和业务相关的指标等。

## Spring Cloud 项目依赖

```groovy
/*
 * 这个 build 文件是由 Gradle 的 `init` 任务生成的。
 *
 * 更多关于在 Gradle 中构建 Java 项目的信息可以查看 Gradle 用户文档中的
 * Java 项目快速启动章节
 * https://docs.gradle.org/4.0/userguide/tutorial_java_projects.html
 */
// 在这个区块中你可以声明你的 build 脚本需要的依赖和解析下载该依赖所使用的仓储位置
buildscript {
    ext {
        springBootVersion = '2.0.3.RELEASE'
        springCloudVersion = 'Finchley.RELEASE'
        propDepsVersion = '0.0.9.RELEASE'
        springFoxVersion = '2.9.0'
        redissonVersion = '3.6.5'
        javerVersion = '3.10.0'
        jwtVersion = '0.9.0'
        mongobeeVersion = '0.13'
        problemVersion = '0.23.0'
        javaTuplesVersion = '1.2'
        jacksonDataTypeJsr310Version = '2.9.5'
        jpushVersion = '3.3.6'
        awsVersion = '1.11.136'
        javassistVersion = '3.18.2-GA'
        springbootAdminVersion = '2.0.2'
        jPinyinVersion = '1.1.8'
    }
    ext['spring.data.elasticsearch.version'] = '3.0.9.RELEASE'
    repositories {
        maven { setUrl('http://maven.aliyun.com/nexus/content/groups/public/') }
        maven { setUrl('http://maven.aliyun.com/nexus/content/repositories/jcenter') }
        maven { setUrl('http://repo.spring.io/plugins-release') }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("io.spring.gradle:propdeps-plugin:${propDepsVersion}")
        classpath("org.springframework:springloaded:1.2.5.RELEASE")
    }
}

allprojects {
    group = 'dev.local.gtm'
    apply plugin: 'idea'
}

subprojects {
    version = "0.0.1"
    tasks.withType(Jar) {
        baseName = "$project.name"
    }
    apply plugin: 'java'
    apply from: '../gradle/docker.gradle'
    apply plugin: 'propdeps'
    apply plugin: 'propdeps-idea'
    apply plugin: 'io.spring.dependency-management'
    sourceCompatibility = 1.8
    targetCompatibility = 1.8
    repositories {
        maven { setUrl('http://maven.aliyun.com/nexus/content/groups/public/') }
        maven { setUrl('http://maven.aliyun.com/nexus/content/repositories/jcenter') }
        maven { setUrl('https://repo.spring.io/libs-milestone') }
        jcenter()
    }
    dependencies {
        runtime('org.springframework.boot:spring-boot-devtools')
        compileOnly("org.projectlombok:lombok")
        compileOnly("org.springframework.boot:spring-boot-configuration-processor")
        testImplementation("org.springframework.boot:spring-boot-starter-test")
        testImplementation("org.projectlombok:lombok")
    }
    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```
