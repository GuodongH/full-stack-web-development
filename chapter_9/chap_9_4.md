# 微服务的远程调用

## Feign Client

Feign <https://github.com/OpenFeign/feign> 是 Netflix 出品的一个开源的声明式 HTTP 客户端， Feign 的目标是简化 HTTP 客户端，简单来说，开发者仅需要声明和注解一个接口，真正的接口实现会在运行时提供。我们在前面的章节中使用过 `RestTemplate` ，那么 Feign 可以看作是一个可以感知服务注册信息的 `RestTemplate` 。

举例来说，如果没有 Feign ，我们使用 RestTemplate 来实现微服务之间的调用的话，需要在 Controller 中注入 `eurekaClient` 以便我们可以取得要调用的微服务的地址和端口信息。

```java
@Autowired
private EurekaClient eurekaClient;

public void someAPI() {
    Application application = eurekaClient.getApplication("api-service");
    InstanceInfo instanceInfo = application.getInstances().get(0);
    String hostname = instanceInfo.getHostName();
    int port = instanceInfo.getPort();
    // ...
}
```

但显然这样做有点麻烦， Feign 可以大大的简化微服务中的这种调用。我们只需要添加一个注解 `@FeignClient("api-service")` ，这样就意味着我们要调用 Eureka 中注册的 `api-service` 的方法。其余的就可以按照标准的 RestController 去写了，这个接口的具体实现会在运行时提供：

```java
@FeignClient("api-service")
public interface ApiServiceClient {
    @RequestMapping("/api/users")
    String getUsers();
}
```

是的， 就这么简单，而且除了对于微服务内部的调用之外， Feign 在实现第三方接口调用时也非常好用。尤其对于标准的 RESTful 接口来说，比 RestTemplate 要简洁很多。但是对于较老的一些接口，尤其是不太规范的接口，支持的灵活度和可定制性上比 RestTemplate 要弱一些。

### 调用第三方 API -- LeanCloud

如果同样是调用 LeanCloud 的图形验证码和短信验证服务 API ，我们使用 Feign 的话，就只需定义下面的接口，就可以完成 API 的接口定义了。这种书写形式比 RestTemplate 要清晰和简洁。

```java
package dev.local.smartoffice.api.service.feign;

import dev.local.smartoffice.api.config.feign.LeanCloudConfiguration;

import dev.local.smartoffice.api.domain.Captcha;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

import lombok.*;

/**
 * LeanCloudService
 */
@FeignClient(name = "leanCloud", url = "${app.leanCloud.baseUrl}", configuration = LeanCloudConfiguration.class)
public interface LeanCloudFeignClient {

    @PostMapping("/requestSmsCode")
    void requestSmsCode(@RequestBody RequestSmsCodeParam requestSmsCodeParam);

    @PostMapping("/verifySmsCode/{code}")
    void verifySmsCode(@RequestBody VerifySmsCodeParam verifySmsCodeParam, @PathVariable String code);

    @GetMapping("/requestCaptcha")
    Captcha requestCaptcha();

    @PostMapping("/verifyCaptcha")
    String verifyCaptcha(@RequestBody VerifyCaptchaParam verifyCaptchaParam);

    @Getter
    @Setter
    @AllArgsConstructor
    @NoArgsConstructor
    public static class RequestSmsCodeParam {
        private String mobilePhoneNumber;
        private String validate_token;
    }

    @Getter
    @Setter
    @AllArgsConstructor
    @NoArgsConstructor
    public static class VerifySmsCodeParam {
        private String mobilePhoneNumber;
    }

    @Getter
    @Setter
    @AllArgsConstructor
    @NoArgsConstructor
    public static class VerifyCaptchaParam {
        private String captcha_code;
        private String captcha_token;
    }
}

```

注意到上面代码中的 `url = "${app.leanCloud.baseUrl}"` 规定第三方的 API 的 URL ，而这个 URL 是可以定义在 `.properties` 或者 `.yml` 中的。在上面例子中我们只需要在 `applicaiton.yml` 中定义 `app.leanCloud.baseUrl` 的值为 LeanCloud 分配的应用域名即可。

当然，同样的我们也需要提供截断器、自定义错误处理等，这些我们可以都在一个 `LeanCloudConfiguration` 中定义，在上面的代码中只需要指定配置的类型即可 `configuration = LeanCloudConfiguration.class` 。这个配置类中，我们可以定制化以下内容

* `Logger.Level` -- 设置日志等级
  * `NONE` -- 没有日志
  * `BASIC` -- 记录请求的方法、 URL 和返回的响应码以及响应时间
  * `HEADERS` -- 除了 `BASIC` 提供的日志之外，还提供请求（ Request ）和响应（ Response ）的头 ( Headers )
  * `FULL` -- 完整日志，包括请求和响应的 Header 和 Body 以及其他元数据。
* `Retryer` -- 定义重试逻辑
* `ErrorDecoder` -- 可以进行自定义的错误处理
* `Request.Options` -- 定义请求的连接超时和读取超时
* `Collection<RequestInterceptor>` -- 截断器，可以对请求进行拦截，一般可以写入较通用的 Header ，比如授权信息等。

```java
package dev.local.smartoffice.api.config.feign;

import dev.local.smartoffice.api.config.AppProperties;
import feign.Logger;
import feign.codec.Encoder;
import feign.codec.ErrorDecoder;
import lombok.RequiredArgsConstructor;

import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@RequiredArgsConstructor
public class LeanCloudConfiguration {

    private final ObjectFactory<HttpMessageConverters> messageConverters;
    private final AppProperties appProperties;

    @Bean
    public Logger.Level feignLogger() {
        return Logger.Level.FULL;
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new LeanCloudErrorDecoder();
    }

    @Bean
    public Encoder feignEncoder() {
        return new SpringEncoder(messageConverters);
    }

    @Bean
    public LeanCloudAuthHeaderInterceptor leanCloudAuthHeaderInterceptor() {
        return new LeanCloudAuthHeaderInterceptor(appProperties);
    }
}

```

一个典型的 Feign 拦截器，如下面的代码所示，需要实现 `RequestInterceptor` ，在 Interceptor 中我们在所有请求的头部添加 LeanCloud 要求的 `AppId` 和 `AppKey` ，以及设置 `Content-Type` 为 `application/json` 。

```java
package dev.local.smartoffice.api.config.feign;

import org.springframework.http.MediaType;

import dev.local.smartoffice.api.config.AppProperties;
import feign.RequestInterceptor;
import feign.RequestTemplate;
import lombok.RequiredArgsConstructor;

/**
 * LeanCloudAuthHeaderInterceptor
 */
@RequiredArgsConstructor
public class LeanCloudAuthHeaderInterceptor implements RequestInterceptor {

    private final AppProperties appProperties;

    @Override
    public void apply(RequestTemplate template) {
        template.header("X-LC-Id", appProperties.getLeanCloud().getAppId());
        template.header("X-LC-Key", appProperties.getLeanCloud().getAppKey());
        template.header("Content-Type", MediaType.APPLICATION_JSON_UTF8_VALUE);
    }
}

```

和 RestTemplate 类似的，我们也可以添加自定义的错误处理机制，在连接第三方服务时，这是很有必要的。因为第三方的服务往往采用各不相同的错误处理，有必要在系统中将第三方返回的错误统一处理成系统内部规范的机制。

```java
package dev.local.smartoffice.api.config.feign;

import com.fasterxml.jackson.core.JsonFactory;
import com.fasterxml.jackson.core.JsonParseException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.local.smartoffice.api.util.ErrorResponseUtil;
import dev.local.smartoffice.api.web.exception.OutgoingBadRequestException;
import feign.Response;
import feign.codec.ErrorDecoder;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import lombok.val;

import java.io.IOException;

@Slf4j
public class LeanCloudErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() != 200) {
            return new ErrorDecoder.Default().decode(methodKey, response);
        }
        String json;
        try {
            json = ErrorResponseUtil.readFully(response.body().asInputStream());
            log.debug("[LeanCloud] 解析返回错误 {}", json);
            ObjectMapper mapper = new ObjectMapper(new JsonFactory());
            JsonNode jsonNode = mapper.readValue(json, JsonNode.class);
            Integer code = jsonNode.has("code") ? jsonNode.get("code").intValue() : null;
            String err = jsonNode.has("error") ? jsonNode.get("error").asText() : null;

            val error = new LeanCloudError(code, err);
            log.debug("[LeanCloud] 错误: ");
            log.debug("   代码        : {}", error.getCode());
            log.debug("   信息        : {}", error.getError());
            return new OutgoingBadRequestException(error.getError());
        } catch (JsonParseException e) {
            return new OutgoingBadRequestException(e.getMessage());
        } catch (IOException e) {
            return new OutgoingBadRequestException(e.getMessage());
        }
    }

    @Data
    @AllArgsConstructor
    private final class LeanCloudError {
        private Integer code;
        private String error;
    }
}

```

利用 Feign Client 的好处除了代码简洁流畅之外，还可以利用 Ribbon 进行负载均衡以及利用 Hystrix 进行熔断处理。

## 负载均衡

负载平衡自动在两台或多台计算机之间分配传入的应用程序流量。它让我们能够在应用程序中实现容错，无缝地提供路由应用程序流量所需的负载平衡容量。负载平衡旨在优化资源使用，最大化吞吐量，最小化响应时间，并避免任何单个资源的过载。使用具有负载平衡的多个组件可以通过冗余提高可靠性和可用性。

### Ribbon

Ribbon是一个客户端负载均衡器，可以让我们对 HTTP 和 TCP 客户端的行为进行控制。Ribbon 的客户端组件提供了很多好用的配置选项，例如连接超时，重试，重试算法（指数，边界退回）等。Ribbon 内置了可插拔和可自定义的负载平衡组件。

下面列出了一些提供的负载平衡策略：

* 简单的循环负载均衡
* 加权响应时间负载均衡
* 基于区域的轮询负载均衡
* 随机负载均衡

### 客户端负载均衡

负载平衡的实现方法是向客户端提供服务器 IP 列表，然后让客户端从每个连接的列表中随机选择 IP 。一般来说，客户端随机负载平衡往往比循环 DNS 提供更好的负载分配。利用这种方法，向客户端传送 IP 列表的方法可以有多种变化形式，甚至可以实现为 DNS 列表（在没有任何循环的情况下传递给所有客户端），或者通过将其硬编码到列表中。如果使用“智能客户端”，检测到随机选择的服务器已关闭并再次随机连接，则还提供容错功能。

Ribbon 提供以下功能：

* 负载均衡
* 容错
* 异步和反应模型中的多协议（ HTTP，TCP，UDP ）支持
* 缓存和批处理

Ribbon 中的一个核心概念是指定客户端的概念。每个负载均衡器都是微服务架构中多个组件的一部分，这些组件一起工作以按需联系远程服务器，并且该集合可以使用一个对开发人员友好的名称。

如果不使用 Eureka 的情况下，我们可以通过配置文件进行负载均衡 server 的配置。下面例子中 `listOfServers` 列出了用于负载均衡的三个服务地址和端口。

```yml
api-service:
  ribbon:
    eureka:
      enabled: false
    listOfServers: localhost:8091,localhost:8092,localhost:8093
    ServerListRefreshInterval: 15000
```

但如果我们使用了 Eureka 的话，我们就无需任何配置，当然 Eureka 的配置还是需要的。Spring Cloud 集成了 Ribbon 和 Eureka ，可在使用 Feign 时提供负载均衡的 HTTP 客户端。此时默认的无需指定 `listOfServers` ，因为 Feign 会自动从 Eureka 拉取 Server 列表，并利用集成的 Ribbon 进行负载均衡。
