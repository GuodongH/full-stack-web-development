# Spring Boot 的自动化测试

一个大型项目如果没有自动化测试，那交付质量可就岌岌可危了。因为一个大型项目，会有较多的开发团队参与分工，代码的复杂加上协同人员的增多，导致产生的 bug 往往需要多个项目组配合才能发现根本原因，这一方面增加了项目成本，另一方面严重影响生产效率。因为现代软件的迭代速度很快，如果完全使用人工测试，会严重拖慢项目进度。自动化测试在大工程中会确保每次提交的代码保证一定程度的质量。

但自动化测试也不是万能的，并不能替代人工测试，比如有些需要叫复杂操作才能呈现的 bug ，比如由于软件架构的限制，无法或者难以实现测试自动化的地方。而且自动化测试也并不适合一个项目在原型开发的阶段大规模的推进，因为此时的代码修改会比较频繁。此外我个人也并不赞成追求测试的覆盖度，因为有一些类和方法完全没有测试的必要。比如单纯的数据对象，只有 `getter` 和 `setter` 这样的方法，我们对它们进行自动化测试的意义就仅仅在于测试覆盖度提升的百分点了，这种工作既耗时又无聊，不做也罢。但对于对外部的接口，以及内部重要的业务逻辑来说，这个自动化测试就很有必要。因为我们希望不会因为某个改动导致公开给其他团队的接口不好用了，或者影响到内部以前好用的业务逻辑，这个时候自动化测试就是一个提升生产效率的利器了。

## 如何开始构建一个测试

按照标准的 Java 项目结构， `src/main` 用于存储源代码，而 `src/test` 用于测试的代码。我们在 `src/test` 中使用项目主工程同样的包结构。只不过对于测试来说，我们一般对于类的命名后面都会有一个 `Test` 以标识这是一个测试类。

在开始写我们的测试类之前，首先确定你在 `build.gradle` 中是否已经引入测试所需要的依赖，我们对测试的大部分依赖通过 `org.springframework.boot:spring-boot-starter-test` 即可提供。

```groovy
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

Spring 会自动导入

* JUnit: Java 单元测试框架，已经基本是业界标配了。
* Spring Test & Spring Boot Test: 提供 Spring Boot 集成测试的支持类库和工具类库
* AssertJ: 断言类库
* Hamcrest: 用于条件约束和匹配的类库
* Mockito: 用于模拟对象的框架
* JSONassert: JSON 的断言类库
* JsonPath: JSON 路径表达语言类库

在 Spring 中进行测试是非常简单的，这是因为 Spring 提供良好的依赖注入特性，而这使得我们的代码更容易进行单元测试。在进行单元测试得时候，经常使用的一个技巧是使用 Mock Object 而不是真正的依赖项。除去单元测试外，我们很多时候还需要进行更真实的测试 -- 集成测试，这在 Spring 中意味着我们需要使用 Spring ApplicationContext 。在集成测试中，我们经常碰到的挑战是怎样才能在不需要部署应用程序或连接到其他服务的情况下执行测试。

Spring Boot 提供 `@SpringBootTest` 注解，当需要 Spring Boot 特性时，使用它就可以了，这个注解配合 `@RunWith(SpringRunner.class)` 注解即可标识一个测试类。

默认情况下，`@SpringBootTest` 不会启动服务器，但我们可以使用 `webEnvironment` 属性来进一步指定测试的运行方式：

* MOCK：这是默认值，提供模拟的 Web 环境，不会启动服务器。它可以与 `@AutoConfigureMockMvc` 或`@AutoConfigureWebTestClient` 配合使用进行基于模拟的 Web 环境的测试。
* RANDOM_PORT：随机端口，加载 WebServerApplicationContext 并提供真实的 Web 环境。嵌入式服务器的端口是随机分配的。
* DEFINED_PORT：指定端口，和 `RANDOM_PORT` 类似，只不过端口是通过 `application.yml` 指定的。
* NONE：不提供任何 Web 环境。

### Web 层单元测试

下面我们就来看看如何写一个测试，这个测试中，我们会将使用 `@WebMvcTest` 这个注解，和 `@SpringBootTest` 不同，这个 `@WebMvcTest` 注解是单独为 Web 层测试准备的。 `@WebMvcTest` 只扫描 `@Controller` ， `@ControllerAdvice` ， `@JsonComponent` ， `Converter` ， `GenericConverter` ， `Filter` ， `WebMvcConfigurer` 和 `HandlerMethodArgumentResolver` 。 其他的比如 `@Component` 之类就不会被扫描到了。如果需要使用其他类型，可以使用 `@Import` 导入。

```java
package dev.local.gtm.api.rest;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.local.gtm.api.config.AppProperties;
import dev.local.gtm.api.domain.Captcha;
import dev.local.gtm.api.domain.JWTToken;
import dev.local.gtm.api.service.AuthService;
import dev.local.gtm.api.web.exception.ExceptionTranslator;
import dev.local.gtm.api.web.rest.AuthResource;
import dev.local.gtm.api.web.rest.vm.UserVM;
import lombok.val;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.view.json.MappingJackson2JsonView;

import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.doNothing;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@WebMvcTest(controllers = {AuthResource.class}, secure = false)
public class AuthResourceTest {

    @MockBean
    private AppProperties appProperties;

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private AuthService authService;

    @Autowired
    private HttpMessageConverter[] httpMessageConverters;

    @Autowired
    private ExceptionTranslator exceptionTranslator;

    @Before
    public void setup() {
        val authResource = new AuthResource(authService, appProperties);
        mockMvc = MockMvcBuilders.standaloneSetup(authResource)
            .setMessageConverters(httpMessageConverters)
            .setControllerAdvice(exceptionTranslator)
            .setViewResolvers((ViewResolver) (viewName, locale) -> new MappingJackson2JsonView()).build();
    }

    @Test
    public void testCaptchaRequestSuccessfully() throws Exception {
        val captcha = new Captcha();
        captcha.setUrl("http://someplace/somepic.jpg");
        captcha.setToken("someToken");
        given(this.authService.requestCaptcha()).willReturn(captcha);
        mockMvc.perform(get("/api/auth/captcha"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.captcha_url").isNotEmpty())
            .andExpect(jsonPath("$.captcha_token").isNotEmpty());
    }

    @Test
    public void testCaptchaVerificationSuccessfully() throws Exception {
        val code = "testCode";
        val token = "testToken";
        val verification = new AuthResource.CaptchaVerification();
        verification.setCode(code);
        verification.setToken(token);
        given(this.authService.verifyCaptcha(code, token)).willReturn("testValidateToken");
        mockMvc.perform(post("/api/auth/captcha")
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .content(objectMapper.writeValueAsString(verification)))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.validate_token").isNotEmpty());
    }

    @Test
    public void testRegisterSuccess() throws Exception {

        val validateToken = "testValidateToken";
        val user = UserVM.builder()
            .login("test1")
            .mobile("13000000000")
            .email("test1@local.dev")
            .name("test 1")
            .password("12345")
            .validateToken(validateToken)
            .build();
        val security = new AppProperties.Security();
        doNothing()
            .when(this.authService)
            .verifyCaptchaToken(validateToken);
        doNothing()
            .when(this.authService)
            .registerUser(user.toUserDTO(), "12345");
        given(this.authService.login(user.getLogin(), user.getPassword()))
            .willReturn(new JWTToken("idToken", "refreshToken"));
        given(this.appProperties.getSecurity())
            .willReturn(security);

        mockMvc.perform(post("/api/auth/register")
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .content(objectMapper.writeValueAsString(user)))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id_token").isNotEmpty());
    }

}

```

上面的例子中由于我们只想测试 Web 层，所以我们对于 Web 层所依赖的 `AuthService` 和 `AppProperties` 使用了 `@MockBean` 注解。

运行测试时，有时需要在上下文中模拟某些组件或服务。例如，刚刚的例子中我们对于只想单独测试 Web 层，对于服务层可能并不关心，再比如在应用依赖的某些开发期间不可用的某些远程服务，或者模拟在真实环境中可能难以触发的故障时等等，我们就会发现需要模拟某些对象的行为。

Spring Boot 包含一个 `@MockBean` 注解，这个注解将会为修饰的类型创建一个基于 `Mockito` <https://site.mockito.org/> 的模拟对象。对于每个 `@Test` 方法，该模拟对象都会自动重置。

那么我们该如何使用这些模拟对象呢？我们通过一个简单的例子来说明，上面代码中的 `testCaptchaRequestSuccessfully` 测试方法中，我们是要测试 `/api/auth/captcha` 这个 Rest API 。那么先来看看要测试的目标方法是怎么定义的？

```java
@GetMapping("/auth/captcha")
public Captcha requestCaptcha() {
    log.debug("REST 请求 -- 请求发送图形验证码 Captcha");
    return authService.requestCaptcha();
}
```

这个方法简单到不行，就是直接调用了服务层的对应方法而已，从实践角度来说，这种就是典型的无需测试的那种方法，但这里为了更好的说明一些测试技巧，就把它当作例子了。

书归正传，这个 `requestCaptcha` 方法中有一个依赖，就是 `AuthService` ，也就是说我们如果访问 `/api/auth/captcha` ，那么就会进入 `requestCaptcha` 方法，但是这个方法中又调用了服务层的方法： `authService.requestCaptcha()` 。如果我们要单独测试 `requestCaptcha` ，就需要提供一个 `AuthService` 的模拟对象。这个模拟对象我们在测试类中已经使用 `@MockBean` 提供了，而且在 `setup()` 中通过 `val authResource = new AuthResource(authService, appProperties);` 将模拟对象使用构造注入了 `AuthResource` 中（请回过头再想想依赖注入的妙处）。也就是说，

在我们的测试中， `AuthResource` 使用的是这个模拟的 `AuthService` 而非真正的 `AuthService` 。 `mockMvc.perform(get("/api/auth/captcha"))` 说的是向 `/api/auth/captcha` 发送一个 `GET` 请求。然后使用 `andExpect` 来指定一个期望结果，比如我们的第一个期望结果就是 `status().isOk()` 返回码是 `Ok` ，一般来说就是 `200` 。而 `jsonPath` 是可以使用类似 `xml` 中 `xpath` 的形式来表达 `json` 中的节点路径，就是说在返回的 Http Response 中的 `json` 对象中的第一级节点中的 `captcha_url` 节点。我们期待这个节点不为空。

```java
@Test
public void testCaptchaRequestSuccessfully() throws Exception {
    val captcha = new Captcha();
    captcha.setUrl("http://someplace/somepic.jpg");
    captcha.setToken("someToken");
    given(this.authService.requestCaptcha()).willReturn(captcha);
    mockMvc.perform(get("/api/auth/captcha"))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$.captcha_url").isNotEmpty())
      .andExpect(jsonPath("$.captcha_token").isNotEmpty());
}
```

细心的同学此时会发现一个问题，如果方法有返回值的话，使用上面的 `given(...).willReturn(...)` 这种方法去模拟当然没有问题，但没有返回值的怎么办？这种情况下我们可以使用 `doNothing().when(someService).someVoidMethod()` 这种模拟方式：就是说在 `someService` 的 `someVoidMethod()` 被调用时什么都不做。下面的代码就很好的体现了这个范式：

```java
@Test
public void testRegisterSuccess() throws Exception {

    val validateToken = "testValidateToken";
    val user = UserVM.builder()
        .login("test1")
        .mobile("13000000000")
        .email("test1@local.dev")
        .name("test 1")
        .password("12345")
        .validateToken(validateToken)
        .build();
    val security = new AppProperties.Security();
    doNothing()
        .when(this.authService)
        .verifyCaptchaToken(validateToken);
    doNothing()
        .when(this.authService)
        .registerUser(user.toUserDTO(), "12345");
    given(this.authService.login(user.getLogin(), user.getPassword()))
        .willReturn(new JWTToken("idToken", "refreshToken"));
    given(this.appProperties.getSecurity())
        .willReturn(security);

    mockMvc.perform(post("/api/auth/register")
        .contentType(MediaType.APPLICATION_JSON_UTF8)
        .content(objectMapper.writeValueAsString(user)))
        .andDo(print())
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.id_token").isNotEmpty());
}
```

接下来的事情就比较有趣了，上面代码中，我们使用了一个类似自然语言的表达 `given(this.authService.requestCaptcha()).willReturn(captcha)` 。这个表达的意思就是如果遇到 `this.authService.requestCaptcha()` 方法调用时，请返回 `captcha` 这个对象。这样就摆脱了对于真实 `AuthService` 的依赖，如果我负责 Rest API ，而你负责服务层，在你真正的服务开发好之前，我已经可以测试了。

讲完了模拟对象，我们接下来看一下 `MockMVC` ， Spring 中提供的这个对象让我们可以非常快速方便的测试 `Controller` 而无需启动一个真实的 HTTP 服务器。

### 集成测试

通常情况下一个单元测试只关注自己这个逻辑单元，而集成测试则需要整个系统启动起来，其实不只是系统本身需要启动，可能还涉及到系统依赖的外围系统，比如数据库、缓存服务器等等。但如果我们每次测试都得把所有系统启动起来，这测试的准备过程也过于繁琐，而且还容易出错，比如某一个依赖的系统出问题就导致测试无法运行。而且使用真实的周边系统带来的另一个问题就是测试环境的清理问题，每次在数据库中可能会产生大量测试数据，这些数据都需要清理以便进行下一轮测试。

所以在自动化测试中，就像我们使用 Mock 对象一样，一般我们采用一个模拟的周边系统嵌入到待测试系统中。在我们的工程中使用了 Redis, ElasticSearch 和 MongoDB 。所以我们也引入三个嵌入式的服务： `embedded-redis` 、 `embedded-elasticsearch` 和 `de.flapdoodle.embed.mongo` ，其中嵌入式 MongoDB 没有指定版本号的原因是 Spring Boot 内建了这个依赖。

```groovy
testImplementation("com.github.kstyrc:embedded-redis:${embeddedRedisVersion}")
testImplementation("pl.allegro.tech:embedded-elasticsearch:${embeddedElasticsearchVersion}")
testImplementation("de.flapdoodle.embed:de.flapdoodle.embed.mongo")
```

引入依赖之后，我们还需要建立两个配置类，分别在测试启动时启动 Redis 和 ElasticSearch 服务，在测试关闭时关闭对应服务。MongoDB 的话， Spring Boot 会自动配置，所以就不需要我们手动写配置类了。

```java
package dev.local.gtm.api.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.io.IOException;

@Configuration
@Profile("test")
public class EmbeddedRedisTestConfig {

    private final redis.embedded.RedisServer redisServer;

    public EmbeddedRedisTestConfig(@Value("${spring.redis.port}") final int redisPort) throws IOException {
        this.redisServer = new redis.embedded.RedisServer(redisPort);
    }

    @PostConstruct
    public void startRedis() {
        this.redisServer.start();
    }

    @PreDestroy
    public void stopRedis() {
        this.redisServer.stop();
    }
}

```

在 Redis 配置中，我们通过 `@Value("${spring.redis.port}")` 读取配置文件 `application.yml` 中定义的 Redis 端口，然后使用 `@PostConstruct` 注解在构造函数之后启动该服务，使用 `@PreDestroy` 注解在该对象销毁之前停止服务。需要指出的是，测试配置文件，我一般会使用 `@Profile("test")` 注解规定只在测试环境下加载该配置类。

类似的，对于 ElasticSearch 我们创建一个配置类，ElasticSearch 的模拟服务甚至可以设置版本号，这一点非常方便。

```java
package dev.local.gtm.api.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import pl.allegro.tech.embeddedelasticsearch.EmbeddedElastic;
import pl.allegro.tech.embeddedelasticsearch.PopularProperties;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.io.IOException;

@Configuration
@Profile("test")
public class EmbeddedElasticsearchTestConfig {
    private final EmbeddedElastic embeddedElastic;

    public EmbeddedElasticsearchTestConfig() throws IOException {
        this.embeddedElastic = EmbeddedElastic.builder()
            .withElasticVersion("5.5.0")
            .withSetting(PopularProperties.TRANSPORT_TCP_PORT, 9300)
            .withSetting(PopularProperties.CLUSTER_NAME, "docker-cluster")
            .withPlugin("analysis-stempel")
            .build();
    }

    @PostConstruct
    public void startRedis() throws IOException, InterruptedException {
        this.embeddedElastic.start();
    }

    @PreDestroy
    public void stopRedis() {
        this.embeddedElastic.stop();
    }
}

```

对于集成测试来说，我们就不需要 Mock 各种对象了，直接进行各种依赖对象的注入，并执行测试即可。需要特别指出的一点是，默认情况下，Spring Boot 中的分页对象 `Pageable` 在测试中会引发异常，需要我们在构造 MockMvc 时设置一下 `.setCustomArgumentResolvers(new PageableHandlerMethodArgumentResolver())`

```java
package dev.local.gtm.api.rest;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.local.gtm.api.Application;
import dev.local.gtm.api.config.AppProperties;
import dev.local.gtm.api.repository.mongo.TaskRepository;
import dev.local.gtm.api.repository.mongo.UserRepository;
import dev.local.gtm.api.web.exception.ExceptionTranslator;
import dev.local.gtm.api.web.rest.TaskResource;
import lombok.val;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.web.PageableHandlerMethodArgumentResolver;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.context.web.WebAppConfiguration;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.view.json.MappingJackson2JsonView;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;


@RunWith(SpringRunner.class)
@WebAppConfiguration
@SpringBootTest(classes = Application.class)
@ActiveProfiles("test")
public class TaskResourceTest {

    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TaskRepository taskRepository;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private HttpMessageConverter[] httpMessageConverters;

    @Autowired
    private ExceptionTranslator exceptionTranslator;

    @Autowired
    private AppProperties appProperties;

    @Before
    public void setup() {
        taskRepository.deleteAll();
        userRepository.deleteAll();
        val taskResource = new TaskResource(taskRepository, userRepository);

        mockMvc = MockMvcBuilders.standaloneSetup(taskResource)
            .setMessageConverters(httpMessageConverters)
            .setControllerAdvice(exceptionTranslator)
            .setCustomArgumentResolvers(new PageableHandlerMethodArgumentResolver())
            .setViewResolvers((ViewResolver) (viewName, locale) -> new MappingJackson2JsonView()).build();
    }

    @Test
    @WithMockUser
    public void testGetTasks() throws Exception {
        mockMvc.perform(get("/api/tasks").accept(MediaType.APPLICATION_JSON)).andExpect(status().isOk());
    }

}

```

现在我们可以运行一下测试，既可以从 IDEA 中点击类或方法左边的图标启动测试，可以进行整个类的测试，也可以单独测试类中的某个测试方法。

![IDEA 中可以启动测试](/assets/2018-08-23-11-29-39.png)

当然也可以使用命令行进行测试，如果我们使用持续集成或持续发布系统的话，就可以在对应的脚本中使用命令进行测试了。

```bash
./gradlew test
```

执行命令后如果，没有错误就可以看到下面这样的输出了。

```txt
> Task :gtm-api:test
// 省略 log 输出
BUILD SUCCESSFUL in 1m 5s
9 actionable tasks: 9 executed

```

如果有测试失败的时候，在 IDEA 中，我们可以看到如下输出

![测试用例返回失败](/assets/2018-08-23-11-57-20.png)

其中 `java.lang.AssertionError: Expected an empty value at JSON path "$.id_token" but found: 'idToken'` 这句说明失败的原因是，我们期待第一层节点 `id_token` 是空 `jsonPath("$.id_token")` ，但实际返回的是 `idToken` 这个字符串。如果我们将断言改成 `jsonPath("$.id_token").isNotEmpty()` 那么测试就会通过了。
