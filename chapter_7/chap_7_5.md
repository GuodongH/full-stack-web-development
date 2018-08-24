# WebSocket 实时通讯服务

WebSocket 协议（ RFC 6455 ）提供了一种标准化的方式来进行服务器和客户端之间进行双工、双向的基于 TCP 连接的通讯方式。它既不同于 HTTP ，又构建于 HTTP 之上，它可以使用 HTTP 的协议端口，比如 `80` 或 `443` 。

一个 WebSocket 交互是由一个 HTTP 请求开始的，这个 HTTP 请求使用 "Upgrade" 头去转换到 WebSocket 协议：

```txt
GET ws://localhost:8080/websocket/app HTTP/1.1
Host: localhost:8080
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: chrome-extension://heeinifidncnmblmddfajlikkiihfgai
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: _ga=GA1.1.13036277.1526997915; m=
Sec-WebSocket-Key: I88buTTIKcJE1QKxHAMpnA==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Protocol: v10.stomp, v11.stomp, v12.stomp
```

支持 WebSocket 的服务器端收到这个请求后，会返回

```txt
HTTP/1.1 101 Switching Protocols
Expires: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
X-XSS-Protection: 1; mode=block
Origin: chrome-extension://heeinifidncnmblmddfajlikkiihfgai
Upgrade: WebSocket
Pragma: no-cache
Sec-WebSocket-Accept: G+NHta8H3Mjiol8xeT3T3Tilabw=
Date: Thu, 07 Jun 2018 04:45:14 GMT
Connection: Upgrade
Sec-WebSocket-Location: ws://localhost:8080/websocket/app
X-Content-Type-Options: nosniff
Sec-WebSocket-Protocol: v10.stomp
```

在成功的握手之后，TCP socket 会保持对服务器和客户端打开以便发送和接收消息。注意如果 WebSocket 服务器在 Web 服务器后面，比如有时我们采用 Web 服务器是 nginx ，通过反向代理把某些请求指向我们的 API 服务，这种时候就需要配置 nginx 将 WebSocket 的 Upgrade 请求发送到 WebSocket 服务器。

## HTTP 和 WebSocket 的区别和联系

我们在考虑使用 WebSocket 的时候要注意，通常的 Rest API 编程模型和 WebSocket 编程模型的区别是巨大的，这一点无论对服务端还是客户端来说都是如此。在 Rest API 模型中，我们把资源划分成若干 URL ，客户端需要使用 Request/Response 的机制去访问这些 URL ，服务端也会根据这些 Request 的 URL 、方法和请求头来决定对应的处理方式。

而对于 WebSocket 来说，通常是建立一个连接后就通过这个连接发送后继的消息，这个是一个一直保持连接的模型，而不是像 REST 方式那样每次建立新的连接。一般来说，这种模式要求的编程模型是事件驱动类型的，和 Request/Response 的模型区别较大。

另外 WebSocket 是一个较基础的传输协议，没有像 HTTP 那样规定内容的格式、语义，我们一般需要在其之上再封装一层协议来处理内容、格式等，比如我们下面要谈到的 STOMP。

## 何时使用 WebSocket？

初学者往往觉得既然可以实时，那就所有业务都实时化好了，但实时是有成本的，而且大多数场景不需要实时，对这些场景进行实时化处理代价高昂，却收益甚微。

WebSockets 主要用于处理实时性较高的需求，但它并不是这类问题的唯一解决方案，在很多时候，利用 Ajax 或者 HTTP 流或者 HTTP 长连接等方式可以获得类似的效果和体验，甚至很多场景下，用其他方案会比 WebSocket 更好。

并非所有应用都要求实时性，多人在线游戏和股票类应用需要更高的实时性，而新闻类，邮件类或者社交类应用往往只需要阶段性的更新即可。除了时延，数据的传输量也是一个考虑因素，如果数据量不大的情况下，使用 HTTP 长链接（ Long Pooling ）应该是更好的选择。此外网络环境也是一个考虑因素，如果我们的 Web 代理并不支持，或者使用的云服务不支持 Upgrade 到一个需要常驻的连接的话，那么我们也就无法使用 WebSocket 了。

## STOMP

WebSocket 协议定义了两种类型的消息，文本和二进制，但它们的内容格式是未定义的。这些是通过客户端和服务器协商子协议的机制 - 即更高级别的消息传递协议来完成的。在 WebSocket 之上使用子协议来定义每个消息可以发送什么类型的消息，每个消息的格式和内容是什么等等。子协议的使用是可选的，但无论是客户端还是服务器都需要就定义消息内容的某些协议达成一致。

和 `Node.js` 生态中的 `Socket.js` 类似， `Spring` 采用了一个叫做 `STOMP` 的子协议。 `STOMP` 是由 Google 开发的一套基于 WebSocket 之上的消息通讯协议，和传统的 AMQP 协议以及 JMS 的作用类似，但不像传统的协议那么复杂，只包含最常见的一些操作。

STOMP 是一种简单的，面向文本的消息传递协议，最初是为 Ruby ， Python 和 Perl 等脚本语言创建的，用于连接企业消息代理。它旨在解决常用消息传递模式的最小子集。 STOMP 可用于任何可靠的双向流网络协议，如 TCP 和 WebSocket 。虽然 STOMP 是面向文本的协议，但消息是可以携带文本或二进制内容的。

STOMP 客户端可以使用 `SEND` 或 `SUBSCRIBE` 命令发送或订阅消息以及 `destination header` ，这个 `destination header` 描述消息的内容以及应由谁接收消息。这启用了一个简单的“发布 - 订阅”机制，可用于通过代理将消息发送到其他连接的客户端，或者向服务器发送消息以请求执行某些工作。

使用 Spring 的 STOMP 支持时， Spring WebSocket 应用充当客户端的 STOMP 代理。消息被路由到 `@Controller` 注解的消息处理方法或者路由到一个简单的内存代理，该代理跟踪订阅并向订阅用户广播消息。

使用 STOMP 作为子协议能够提供比 WebSocket 更丰富的编程模型：

* 无自己定义自定义消息传递协议和消息格式
* 可以使用支持 STOMP 的客户端
* 可以使用像 RabbitMQ ， ActiveMQ 等消息代理软件来管理订阅和广播消息。
* 应用程序逻辑组织成若干由 `@Controller` 注解的方法中，可以根据 STOMP `destination header` 将对应的消息路由给这些方法。
* 使用 Spring Security 进行安全保障。

## WebSocket 配置

在 Spring Boot 中加入 WebSocket 的支持非常简单，我们只需要创建一个配置类，这个类需要实现 `WebSocketMessageBrokerConfigurer` 这个接口。

```java
package dev.local.smartoffice.api.config;

import dev.local.smartoffice.api.config.websocket.ChannelRegistrationInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

import java.util.Optional;

@Order(Ordered.HIGHEST_PRECEDENCE + 99)
@RequiredArgsConstructor
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final AppProperties appProperties;
    private final ChannelRegistrationInterceptor channelRegistationInterceptor;

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(channelRegistationInterceptor);
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        String[] allowedOrigins = Optional.ofNullable(appProperties.getCors().getAllowedOrigins())
            .map(origins -> origins.toArray(new String[0]))
            .orElse(new String[0]);
        registry.addEndpoint("/websocket/tracker")
            .setAllowedOrigins(allowedOrigins);
        registry.addEndpoint("/websocket/tracker")
            .setAllowedOrigins(allowedOrigins)
            .withSockJS();
    }
}

```

`WebSocketConfig` 使用 `@Configuration` 注解，表明它是 `Spring` 配置类。它还使用了注解`@EnableWebSocketMessageBroker` 。顾名思义，`@EnableWebSocketMessageBroker` 支持 `WebSocket` 消息处理，而这个处理是交由消息的代理进行的。

`configureMessageBroker()` 方法在 `WebSocketMessageBrokerConfigurer` 中配置消息代理。它首先调用`enableSimpleBroker()` 启用一个简单的基于内存的消息代理，以在前缀为 `/topic` 的目标 URL 上将消息传回客户端。它还为使用 `@MessageMapping` 注解的方法中的消息指定了 `/app` 前缀，这个前缀会用于定义所有消息映射。

`registerStompEndpoints()` 方法注册 `/websocket/tracker` 这个 `Endpoint`，也就是建立连接的地址。我们通过 `withSockJS()` 启用 `SockJS` 做为后备选项，以便在 WebSocket 不可用时可以使用备用传输协议。 `SockJS` 客户端将尝试连接到 <ws://localhost:8091/websocket/tracker> 并使用可用的最佳的、可用的传输协议（ `websocket` ， `xhr-streaming` ， `xhr-polling` 等）。

请注意我们注册了两个 `Endpoints` 但它们的 URL 是一样的，区别仅仅是是否配置成启用 `SockJS` 。这么做的原因是一般在前端我们当然可以使用 `SockJS` ，但在 App 客户端，比如 Android 或 iOS 上可能会采用其他支持 STOMP 协议的类库，这种情况下是无法访问启用了 `SockJS` 的 URL 的。所以这里面我们注册了两个 `Endpoints` ，分别对应不同的客户端。

`configureClientInboundChannel()` 方法中我们配置了拦截器，这个拦截器主要是为 WebSocket 的安全连接服务的，下面的章节会讲到。

## WebScoket 安全

WebSocket 消息传递会话中的每个 STOMP 都以 HTTP 请求开始 - 可以是升级到 WebSockets 的请求（即 WebSocket 握手），也可以是在 SockJS 回退使用 HTTP 传输请求的情况下。

Web 应用程序已经具有用于保护 HTTP 请求的身份验证和授权。通常，用户通过 Spring Security 使用某种机制（例如登录页面， HTTP 基本身份验证等）进行身份验证。经过身份验证的用户的安全上下文保存在 HTTP 会话中。 Spring Security 提供 WebSocket 子协议授权，该授权使用 ChannelInterceptor 根据其中的用户头来授权消息。

请注意，STOMP 协议在 CONNECT 帧上确实有“登录”和“密码”标头。这些最初设计用于并且仍然需要例如用于 TCP 上的 STOMP。但是，对于 STOMP over WebSocket ，Spring 默认忽略 STOMP 协议级别的授权标头，并假定用户已在 HTTP 传输级别进行了身份验证，并期望 WebSocket 或 SockJS 会话包含经过身份验证的用户。

但是我们的应用使用并不是基于 Session 的，而是基于令牌的，因此使用 Token 的应用无法在 HTTP 协议级别进行身份验证。他们可能更喜欢在 STOMP 消息传递协议级别使用标头进行身份验证：

* 使用 STOMP 客户端在连接时传递身份验证标头。
* 使用 ChannelInterceptor 处理身份验证标头。

如果我们想和 REST API 中一样采用 token 进行鉴权处理的话，就需要使用截断器对 STOMP 的 CONNECT 命令进行 `header` 进行处理。这里我们构建一个 `ChannelRegistrationInterceptor` ：

```java
package dev.local.smartoffice.api.config.websocket;

import dev.local.smartoffice.api.config.AppProperties;
import dev.local.smartoffice.api.security.jwt.TokenProvider;
import lombok.RequiredArgsConstructor;
import lombok.val;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.security.access.AuthorizationServiceException;
import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

/**
 * 使用截断器对 WebSocket 连接进行鉴权检验
 */
@RequiredArgsConstructor
@Component
public class ChannelRegistrationInterceptor implements ChannelInterceptor {

    private final AppProperties appProperties;
    private final TokenProvider tokenProvider;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
        assert accessor != null;
        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            val bearerTokenHeader = accessor.getNativeHeader(appProperties.getSecurity().getAuthorization().getHeader());
            if(bearerTokenHeader == null) {
                throw new AuthenticationCredentialsNotFoundException("没有找到鉴权头信息");
            }
            val bearerToken = bearerTokenHeader.get(0);
            String prefix = appProperties.getSecurity().getJwt().getTokenPrefix();
            if (!StringUtils.hasText(bearerToken) || !bearerToken.startsWith(prefix)) {
                throw new AuthorizationServiceException("鉴权信息不存在或格式错误");
            }
            String token = bearerToken.substring(prefix.length());
            Authentication user = tokenProvider.getAuthentication(token); // access authentication header(s)
            accessor.setUser(user);
        }
        return message;
    }
}

```

然后在 `WebSocketConfig` 中配置该截断器， Spring 将记录并保存经过身份验证的用户，并将其与同一会话中的后续 STOMP 消息相关联：

```java
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    private final ChannelRegistrationInterceptor channelRegistationInterceptor;

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(channelRegistationInterceptor);
    }
    // 省略其他
}
```

此外我们还可以对 STOMP 中的端点和代理进行权限控制，非常类似我们在 SecurityConfig 中做的配置，我们建立一个 `WebsocketSecurityConfiguration` ：

```java
package dev.local.smartoffice.api.config;

import dev.local.smartoffice.api.security.AuthoritiesConstants;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.SimpMessageType;
import org.springframework.security.config.annotation.web.messaging.MessageSecurityMetadataSourceRegistry;
import org.springframework.security.config.annotation.web.socket.AbstractSecurityWebSocketMessageBrokerConfigurer;

@RequiredArgsConstructor
@Configuration
public class WebsocketSecurityConfiguration extends AbstractSecurityWebSocketMessageBrokerConfigurer {

    @Override
    protected void configureInbound(MessageSecurityMetadataSourceRegistry messages) {
        messages
            .nullDestMatcher().authenticated()
            .simpDestMatchers("/websocket/tracker").hasAuthority(AuthoritiesConstants.USER)
            // 对于以 /topic/ 开头的所有目标位置要求鉴权
            .simpDestMatchers("/topic/**").authenticated()
            .simpDestMatchers("/app/**").authenticated()
            // 对于消息类型为 MESSAGE and SUBSCRIBE 之外的消息禁止访问
            .simpTypeMatchers(SimpMessageType.MESSAGE, SimpMessageType.SUBSCRIBE).denyAll()
            // 其他情况全部禁止访问
            .anyMessage().denyAll();
    }

    /**
     * 对于 WebSockets 禁用 CSRF
     */
    @Override
    protected boolean sameOriginDisabled() {
        return true;
    }
}

```

## 建立一个实时消息 Controller

为简单起见，我们建立一个简单的回声服务，在收到客户端发出消息后将该消息广播到 `/topic/echo` ，此时所有订阅了 `/topic/echo` 也会收到这个消息。

```java
package dev.local.smartoffice.api.web.websocket;

import lombok.RequiredArgsConstructor;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

/**
 * JdProfileController
 */
@RequiredArgsConstructor
@Controller
public class EchoController {

    private final SimpMessagingTemplate template;

    @MessageMapping("/echo")
    @SendTo("/topic/echo")
    public String echoMessage(@Payload String message) {
        return "reply: " + message;
    }
}

```

## 测试 WebSocket

为了测试 WebSocket ，我们需要有一个可以支持 STOMP 的客户端，这里我们采用一个 Chrome 扩展插件叫做 Websocket STOMP Client ，这个插件需要科学上网到 Chrome 商店去下载。安装好插件后，我们就可以在 `URL` 中填 <ws://localhost:8080/websocket/tracker> ，然后在 `Connection Headers` 中填写一个鉴权头，和之前讲过的 Web 鉴权方式一样，是一个 `Authorization: Bearer <token>` 这样的格式。此时我们点击 `Connect` 就会发现 Status 变成了 `CONNECTED` ，表示连接成功了。

![使用 Stomp Client 测试连接](/assets/2018-08-23-13-00-59.png)

连接上之后，我们在 `Subscribe` 文本框中输入 `/topic/echo` 以订阅这个主题。然后在 `Send to topic` 的 `Topic` 中填写 `/app/echo` （还记得我们定义的前缀 `/app` 吗？），在 `Message` 中填写 `hello`

![订阅主题和向主题发送消息](/assets/2018-08-23-20-51-05.png)

我们可以从下面的文本中看到连接、订阅、发送、接收消息的全过程。

```txt
// 连接消息
CONNECTED version:1.2 heart-beat:0,0 user-name:admin
// 订阅 /topic/echo 主题
Subscribing to topic /topic/echo with headers {}
// 向 app/echo 发送 hello 这个消息
Sent 'hello' to topic /app/echo with headers {}
// 订阅的 /topic/echo 收到了服务器返回的消息
MESSAGE destination:/topic/echo
content-type:text/plain;charset=UTF-8
subscription:sub-0
message-id:riBaUTME9ePS2rFH_b2TvIAYNBmgmbMvPRa9M4U7-0
content-length:12
reply: hello
```

我们还可以同时打开多个 Websocket STOMP Client ，订阅同一个主题，看看是否所有 Client 都会收到实时消息。
