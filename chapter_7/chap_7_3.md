# 使用 ElasticSearch 提升搜索性能

Elasticsearch 也是一个著名的 NoSQL 数据库，它是基于原来社区中大名鼎鼎的 `lucence` 文本搜索引擎的基础发展而来，所以它的强项就是搜索。那么问题来了，为什么我们在项目中要采用两个 NoSQL 数据库？既然有了 MongoDB 而且 MongoDB 的查询能力也很强的情况下为什么又引入了 Elasticsearch ？

这个话题其实同样可以应用到 SQL 和 NoSQL ，其实没有任何一种技术框架是万能的，都有各自适合的应用场景。一个现实世界的复杂项目中，可能有些模块或功能适合 SQL 数据库，而有些适合 NoSQL 数据库，没有一个规则说你只能使用一种类型的数据库。相反如果我们可以在不同场景使用不同的数据库才会让我们的工程更加灵活强大。那么 Elasticsearch 在全文搜索方面几乎无人可敌，所以我们对于搜索或全文索引就应该使用 Elasticsearch ，而 MongoDB 是一个更通用的文档数据库，在一般性的增删改查操作上显然更为合适。其实细心的读者会发现前一章我们还用到了一个 NoSQL 数据库 -- Redis ，它擅长以键值对的形式在内存中高速的读写，所以我们的缓存框架采用了 Redis 。

## 配置

### 安装 ElasticSearch

Elasticsearch 的安装很简单，我们还是通过 docker 来进行。Elasticsearch 官方提供的镜像现在改到了 elastic 自己的网站发行，如果想从官网拉下来镜像的话，需要使用下面的命令。

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:5.5.0
```

但这个镜像是含有 `x-pack` 这个扩展包的，这个扩展包提供了很多功能，包括一些付费才能使用的商业特性。在我们的例子中，我们还是希望有一个相对纯净的 elasticsearch ，所以我们自己写一个 Dockerfile 将 `x-pack` 移除掉。请注意，如果你决定保留 `x-pack` 的话，需要自定义配置用来支持 x-pack 的特性。我们在根目录下建立 `docker/elasticsearch` 目录，在此目录下建立 `Dockerfile`

```Dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:5.5.0
RUN elasticsearch-plugin remove x-pack --purge
```

但一般情况下，你会觉得下载速度好慢，所以最好使用像阿里云之类的云服务建立一个自己的仓库，以便提高下载的速度。

### 集成 Spring Data Elasticsearch

对于一个 Spring Boot 应用，如果可以使用 Spring Data 的话，给生产效率带来的提升实在是非常大的，好在 Spring Data 有针对 Elasticsearch 的子项目，也就是 Spring Data Elasticsearch 了。在 `api/build.gradle` 中添加 `spring-boot-starter-data-elasticsearch` 依赖

```groovy
// 省略
dependencies {
    //省略
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch") // <--- 这里
    testImplementation("org.springframework.security:spring-security-test")
}
```

需要注意的是，也许是 Elasticsearch 版本管理的问题，也许是 Spring Data Elasticsearch 的成熟度不高的问题，但总之两者配合时需要注意的细节配置问题比使用其他 Spring Data 的子项目时遇到的要多。

其中最明显的一个需要细心配置的地方是：使用的 Spring Data Elasticsearch 版本号和 Elasticsearch 的版本号有对应关系，从我个人的经验来看，最好和官方推荐的版本一致，否则在使用的过程中会出现很多问题。在成书的时间点 Spring Data Elasticsearch 3.1 还处于 Milestone 阶段，所以我们采用的 Elasticsearch 版本是 `5.5.0` 。

| Spring Boot 版本 (x) | Spring Data Elasticsearch 版本 (y) | Elasticsearch 版本 (z) |
| -------------------- | ---------------------------------- | ---------------------- |
| x <= 1.3.5           | y <= 1.3.4                         | z <= 1.7.2*            |
| 2.x> x >= 1.4.x      | 2.0.0 <=y < 3.0.0**                | 2.0.0 <= z < 3.0.0**   |
| x >= 2.x             | 3.1.0 > y >= 3.0.0                 | 5.5.0                  |
| x >= 2.x             | y >= 3.1.0                         | 6.2.2                  |

### 添加配置类

对于 Elasticsearch 的 Java 配置类，我们只需简单的提供一个 `ElasticsearchTemplate` ，使用了一个自定义的实体映射转换类。这里需要说明的是，对于 Elasticsearch 5.x 来说， `TransportClient` 是框架推荐的客户端，但到了 Elasticsearch 6.x 之后，官方推荐使用新的 `HighLevelRestClient` 以 Http 协议访问服务器。

```java
package dev.local.gtm.api.config;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.elasticsearch.client.transport.TransportClient;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.EntityMapper;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;

import java.io.IOException;

@RequiredArgsConstructor
@Configuration
@EnableConfigurationProperties(ElasticsearchProperties.class)
@ConditionalOnProperty("spring.data.elasticsearch.cluster-nodes")
public class ElasticsearchConfig {

    private final TransportClient transportClient;

    @Bean
    public ElasticsearchTemplate elasticsearchTemplate(Jackson2ObjectMapperBuilder jackson2ObjectMapperBuilder) {
        return new ElasticsearchTemplate(transportClient, new CustomEntityMapper(jackson2ObjectMapperBuilder.createXmlMapper(false).build()));
    }

    public class CustomEntityMapper implements EntityMapper {

        private ObjectMapper objectMapper;

        public CustomEntityMapper(ObjectMapper objectMapper) {
            this.objectMapper = objectMapper;
            objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
            objectMapper.configure(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true);
        }

        @Override
        public String mapToString(Object object) throws IOException {
            return objectMapper.writeValueAsString(object);
        }

        @Override
        public <T> T mapToObject(String source, Class<T> clazz) throws IOException {
            return objectMapper.readValue(source, clazz);
        }
    }
}
```

但是一定要注意，上面的 `@ConditionalOnProperty("spring.data.elasticsearch.cluster-nodes")` 这个注解要求在配置文件中必须指定 `cluster-nodes` 。在 `application.yml` 中添加 `elasticsearch` 的属性 `cluster-name` 和 `cluster-nodes` 。 Elasticsearch 是一个分布式的数据库，所以一般情况下有多个节点构成一个集群， `cluster-name` 是集群的名称，而 `cluster-nodes` 是集群的节点集合，比如 `cluster-nodes: xxx.xxx.xxx.xx:9300, yyy.yy.yyy.yy:9300` 。但是这里我们演示的是一个单机环境，所以写一个节点，也就是本机 `localhost:9300` 即可。

```yml
spring:
  application:
    name: api-service
  # 省略
  data:
    elasticsearch:
      cluster-name: docker-cluster
      cluster-nodes: localhost:9300
  # 省略
```

这个集群名称以及单节点的设置可以在 Elasticsearch 的配置文件中指定，我们在 `docker/elasticsearch` 中建立一个子文件夹 `config` ，然后新建一个 `elasticsearch.yml`

```yml
## Elasticsearch 的默认配置
## 摘自 https://github.com/elastic/elasticsearch-docker/blob/master/build/elasticsearch/elasticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0

# 如果配置在一个公网 IP 时， minimum_master_nodes 需要显性配置
# 设置为 1 允许单节点集群
# 相关细节讨论见: https://github.com/elastic/elasticsearch/pull/17288
discovery.zen.minimum_master_nodes: 1

## 使用单节点发现策略，避免启动时的检查
## 见 https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
#
discovery.type: single-node
```

然后在 `docker-compose.yml` 中指定这个配置文件，在下面的配置中我们建立 `volumes` 的映射将配置文件 `./docker/elasticsearch/config/elasticsearch.yml` 映射到容器内 Elasticsearch 的配置文件 `/usr/share/elasticsearch/config/elasticsearch.yml` ，这样就可以让容器启动后使用我们的配置文件了。

```yml
version: '3.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.6.8
    volumes:
      - esdata:/usr/share/elasticsearch/data
      - ./docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"
      - "9300:9300"
# 省略其他配置
volumes:
  api_db: {}
  redis-data: {}
  esdata: {}
```

我们这里可以简单的使用 `docker-compose up -d elasticsearch` 启动 Elasticsearch 服务，如果打开浏览器访问 <http://localhost:9200> 的话，可以看到类似如下的 `json` 输出，那就是说明 Elasticsearch 启动正常。 `9200` 是 Elasticsearch 的监控端口，而 `9300` 是客户端连接 Elasticsearch 服务的端口，这两个不要混淆了，在 Spring Boot 中的 `application.yml` 配置时一定要使用 `9300` 端口。

![Elasticsearch 的监控端口返回结果](/assets/2018-05-20-00-26-01.png)

那么有了这些基础配置之后，我们可以利用 Elasticsearch 做什么呢？当然是利用它的强项 -- 基于索引的强大数据查询能力。

## 构建用户查询 API

在 `domain` 下新建一个 `search` 包，然后在 `domain/search` 下新建 `UserSearch.java`

```java
package dev.local.gtm.api.domain.search;

import dev.local.gtm.api.domain.Authority;
import dev.local.gtm.api.domain.User;
import lombok.Data;
import lombok.NoArgsConstructor;

import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;

import java.io.Serializable;
import java.util.Set;
import java.util.stream.Collectors;

@Data
@NoArgsConstructor
@Document(indexName = "users", type = "user")
public class UserSearch implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    private String id;

    private String login;

    private String mobile;

    private String name;

    private String email;

    private String avatar;

    private boolean activated;

    private Set<String> authorities;

    public UserSearch(User user) {
        this.id = user.getId();
        this.activated = user.isActivated();
        this.avatar = user.getAvatar();
        this.email = user.getEmail();
        this.login = user.getLogin();
        this.mobile = user.getMobile();
        this.name = user.getName();
        this.authorities = user.getAuthorities().stream()
                .map(Authority::getName)
                .collect(Collectors.toSet());
    }
}
```

这个实体类和 `User` 很像，为什么要重建一个而不是复用 `User` 呢？因为无论从数据库角度还是实际的业务角度，这都是两个相对独立的实体，对于搜索来说，我们有些字段是不希望被搜索的，比如密码，有些字段可能对于搜索来说我们不必以嵌套对象的形式处理，比如权限 `authorities` 等。

接下来，我们就需要为这个领域对象建立一个 Respository ，用于它的增删改查。在 `repository` 下面建立一个子 `package` 叫做 `search` ，然后在其中建立一个新的文件 `UserSearchRepository.java`

```java
package dev.local.gtm.api.repository.search;

import dev.local.gtm.api.domain.search.UserSearch;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserSearchRepository extends ElasticsearchRepository<UserSearch, String> {
}
```

这时候需要注意的是，我们在同一个项目中，我们使用了两个 Spring Data 的 Repository 。而 Spring Boot 的依赖注入通过接口类型去匹配和实例化的，尽管我们没有复用领域对象，但最好还是限定一下两种 Repository 的类型的组件扫描范围。

对于 MongoDB 类型的处理在 `DatabaseConfig` 中增加 `@EnableMongoRepositories` 注解

```java
// 对于 MongoDB 的 Repository 请限定在 dev.local.gtm.api.repository.mongo 包中
@RequiredArgsConstructor
@Configuration
@EnableMongoRepositories(basePackages = "dev.local.gtm.api.repository.mongo")
public class DatabaseConfig {
    // 省略
}
```

而对于 Elasticsearch 类型的处理在 `ElasticConfig` 中增加 `@EnableElasticsearchRepositories` 注解。

```java
// 对于 Elasticsearch 的 Repository 请限定在 dev.local.gtm.api.repository.search 包中
@RequiredArgsConstructor
@EnableElasticsearchRepositories(basePackages = "dev.local.gtm.api.repository.search")
@EnableConfigurationProperties(ElasticsearchProperties.class)
@ConditionalOnProperty("spring.data.elasticsearch.cluster-nodes")
@Configuration
public class ElasticConfig {
    // 省略
}
```

### 改造 Service

我们需要考虑在何时进行 `UserSearch` 的增、删、改、查。要注意的是，并不是 `User` 保存的时候一定会存储 `UserSearch` ，因为 `UserSearch` 对于有些字段并不关心，比如重置密码的方法中就不必涉及 `UserSearch` 。

```java
public class AuthServiceImpl implements AuthService {

    private final UserRepository userRepository;

    private final UserSearchRepository userSearchRepository;

    // 省略其他成员变量

    @Override
    public void registerUser(UserDTO userDTO, String password) {

        // 省略

        log.debug("用户 {} 即将创建", newUser);
        userRepository.save(newUser);
        userSearchRepository.save(new UserSearch(newUser)); // <--- 这里
        this.clearUserCaches(newUser);
        log.debug("用户 {} 创建成功", newUser);
    }
    // 省略
}
```

同样的，在 `UserServiceImpl` 中，我们需要对增加用户 `createUser` 和更改用户信息 `updateUser` 中保存 `UserSearch` ，而在删除用户 `deleteUser` 中之后也需要删除 `UserSearch` ，此外需要添加一个搜索方法 `search` ，这里我们采用了 Spring Data Elasticsearch 提供的 `queryStringQuery` 方法对传入的字符串转换成 Elasticsearch 的查询语句。

```java
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserSearchRepository userSearchRepository;
    // 省略

    @Override
    public User createUser(UserDTO userDTO) {
        // 省略
        userRepository.save(user);
        userSearchRepository.save(new UserSearch(user));
        this.clearUserCaches(user);
        log.debug("用户: {}", user);
        return user;
    }

    @Override
    public Optional<UserDTO> updateUser(UserDTO userDTO) {
        return Optional.of(userRepository.findById(userDTO.getId())).filter(Optional::isPresent).map(Optional::get)
                .map(user -> {
                    // 省略
                    userRepository.save(user);
                    userSearchRepository.save(new UserSearch(user));
                    this.clearUserCaches(user);
                    log.debug("用户: {} 的数据已更新", user);
                    return user;
                }).map(UserDTO::new);
    }

    @Override
    public void deleteUser(String login) {
        userRepository.findOneByLoginIgnoreCase(login).ifPresent(user -> {
            userRepository.delete(user);
            userSearchRepository.delete(new UserSearch(user));
            this.clearUserCaches(user);
            log.debug("已删除用户: {}", user);
        });
    }

    @Override
    public Page<UserSearch> search(String query, Pageable pageable) {
        return userSearchRepository.search(queryStringQuery(query), pageable);
    }

    // 省略
}
```

接下来，就是改造一下 `UserResource` ，我们把搜索类的路径统一放在 `/api/_search/` 下面，把查询的字符串作为路径变量。

```java
@RequiredArgsConstructor
@RequestMapping("/api")
@RestController
public class UserResource {
    private final UserService userService;

    @GetMapping("/_search/users/{query}")
    public Page<UserSearch> search(@PathVariable String query, final Pageable pageable) {
        return userService.search(query, pageable);
    }
}
```

启动应用后，在 <http://localhost:8091/swagger-ui.html#/> 我们可以看到 `/api/_search/users/{query}` 的接口

![在 Swagger 中尝试 Elasticsearch ](/assets/2018-08-07-11-28-40.png)

我们可以在 `query` 中尝试一下 Elasticsearch 的 `QueryString Query` 的威力，当然要实验之前，请先建立一些必要的数据，新建几个 User ，这个可以通过 Swagger 很快的完成。

假设我们的几个用户数据如下

```json
[
    {
      "id": "5b6919202873f9fe26e24752",
      "login": "lisi",
      "mobile": "13000000002",
      "name": "李四",
      "email": "lisi@local.dev",
      "avatar": "avatars:svg-2",
      "activated": true,
      "authorities": [
        "ROLE_USER"
      ]
    },
    {
      "id": "5b69193c2873f9fe26e24754",
      "login": "wangwu",
      "mobile": "13000000003",
      "name": "王五",
      "email": "wangwu@local.dev",
      "avatar": "avatars:svg-1",
      "activated": true,
      "authorities": [
        "ROLE_USER",
        "ROLE_ADMIN"
      ]
    },
    {
      "id": "5b6919042873f9fe26e24750",
      "login": "zhangsan",
      "mobile": "13000000001",
      "name": "张三",
      "email": "zhangsan@local.dev",
      "avatar": "avatars:svg-3",
      "activated": true,
      "authorities": [
        "ROLE_USER"
      ]
    }
]
```

如果我们在 `query` 中输入 `zhangsan` 或者 `张三` 或者甚至 `张` 都可以看到如下的输出，我们搜索到了这个姓名为 `张三` 的用户。

```json
{
  "content": [
    {
      "id": "5b6919042873f9fe26e24750",
      "login": "zhangsan",
      "mobile": "13000000001",
      "name": "张三",
      "pinyinNameInitials": "zs",
      "email": "zhangsan@local.dev",
      "avatar": "avatars:svg-3",
      "activated": true,
      "authorities": [
        "ROLE_USER"
      ],
      "createdBy": "admin",
      "createdDate": "2018-08-07T03:59:00.503Z",
      "lastModifiedBy": "admin",
      "lastModifiedDate": "2018-08-07T03:59:00.503Z"
    }
  ],
  "pageable": {
    "sort": {
      "sorted": false,
      "unsorted": true
    },
    "offset": 0,
    "pageSize": 20,
    "pageNumber": 0,
    "paged": true,
    "unpaged": false
  },
  "facets": [],
  "aggregations": null,
  "scrollId": null,
  "totalElements": 1,
  "totalPages": 1,
  "size": 20,
  "number": 0,
  "numberOfElements": 1,
  "sort": {
    "sorted": false,
    "unsorted": true
  },
  "first": true,
  "last": true
}
```

但如果我们搜索 `zhang` 或者 `san` 结果却会返回空，这个主要是用户名的 `zhangsan` 是一个词，而姓名的 `张三` 这个中文词其实是两个 `word` ，所以 `张` 可以被匹配而 `zhang` 却没有匹配到。那么如果我们想要匹配 `zhang` 怎么办呢？可以使用通配符 `*` ，我们如果在 `query` 输入框中输入 `zhang*` 就有可以搜索到同样的结果了。

有的同学可能看到这里会觉得有点奇怪，因为我们没有指定那个字段，只是给出了要搜索的值，为什么能搜索到结果呢？一般来说我们在做常规 SQL 查询的时候应该是 `name = "value"` 或者模糊查询像 `name like %value%` ，但我们在上面的例子中完全没有指定一个字段。这是由于 Elasticsearch 是一个全文搜索引擎，也就是说它会对所有字段建立索引，从行为上说，它非常类似于 Google 这种网络搜索引擎。

那么如果我们希望只对某个字段进行搜索可不可以呢？当然没问题，我们如果将搜索条件改成 `name:张` ，那么这个条件就是只应用于 `name` 这个字段了。你可以试试如果条件改成 `name:zhang*` 会返回什么？

另一个问题是如果多个条件怎么办？比如说我们希望找到姓名为 `张三` 或者 `李四` 的用户，那么我们可以试试这样的条件 `张三 OR 李四` ，注意那个 `OR` 需要大写，而且之前和之后都有空格。这样的条件会搜索到下面的结果：

```json
{
  "content": [
    {
      "id": "5b6919042873f9fe26e24750",
      "login": "zhangsan",
      "mobile": "13000000001",
      "name": "张三",
      "pinyinNameInitials": "zs",
      "email": "zhangsan@local.dev",
      "avatar": "avatars:svg-3",
      "activated": true,
      "authorities": [
        "ROLE_USER"
      ],
      "createdBy": "admin",
      "createdDate": "2018-08-07T03:59:00.503Z",
      "lastModifiedBy": "admin",
      "lastModifiedDate": "2018-08-07T03:59:00.503Z"
    },
    {
      "id": "5b6919202873f9fe26e24752",
      "login": "lisi",
      "mobile": "13000000002",
      "name": "李四",
      "pinyinNameInitials": "ls",
      "email": "lisi@local.dev",
      "avatar": "avatars:svg-2",
      "activated": true,
      "authorities": [
        "ROLE_USER"
      ],
      "createdBy": "admin",
      "createdDate": "2018-08-07T03:59:28.788Z",
      "lastModifiedBy": "admin",
      "lastModifiedDate": "2018-08-07T03:59:28.788Z"
    }
  ],
  "pageable": {
    "sort": {
      "sorted": false,
      "unsorted": true
    },
    "offset": 0,
    "pageSize": 20,
    "pageNumber": 0,
    "paged": true,
    "unpaged": false
  },
  "facets": [],
  "aggregations": null,
  "scrollId": null,
  "totalElements": 2,
  "totalPages": 1,
  "size": 20,
  "number": 0,
  "numberOfElements": 2,
  "sort": {
    "sorted": false,
    "unsorted": true
  },
  "first": true,
  "last": true
}
```

当然你也可以指定字段 `name:张三 OR name:李四` ，得到的是同样的结果。

### 查询语法

`QueryString Query` 可以使用一套“迷你语言”， `QueryString` 被解析为一系列术语（ term ）和运算符（ operator ）。术语可以是单个单词或短语，也可以用双引号括起来，系统以相同的顺序搜索短语中的所有单词。运算符允许自定义搜索。

#### 字段名称

我们下面会使用一系列的例子来说明：

字段 `name` 包含 `张三`

```json
name:张三
```

`name` 字段包含“张三”或“李四”，如果省略 OR 运算符，将使用默认运算符。

```json
name:(张三 OR 李四)
name:(张三 李四)
```

`name` 字段包含完全匹配的 `张三` 这个词，使用双引号表示要完全匹配。

```json
name:"张三"
```

字段也可以使用通配符，下面的意思是以 `na` 开头的字段包含“张三”或“李四”。

```json
na*:(张三 李四)
```

字段 `name` 包含非空值

```json
_exists_:name
```

#### 通配符

可以使用单个术语运行通配符搜索 `?` 替换单个字符， `*` 替换零个或多个字符：

```js
zha?g s*
```

请注意，通配符查询可能会占用大量内存并执行性能非常差，只需要想想看我们得需要查询多少个术语才能匹配查询字符串 `a * b * c *` ，就会明白为什么了。尤其是在单词的开头使用通配符（例如 `*san` ）的话，会严重影响性能，因为索引中的所有术语都需要检查是否匹配。

#### 模糊匹配

我们可以使用“模糊”运算符 -- `~` -- 搜索与我们的搜索字词类似但不完全相同的字词。

```txt
zhagnsan~
```

上面的表达式如果你仔细观察的话，会发现我们故意把 `zhang` 写成了 `zhagn` ，其中 `g` 和 `n` 颠倒了顺序。但在模糊匹配中这是可以匹配到的，这个功能在有时需要容忍拼写错误时尤其有用。

Elasticsearch 使用 `Damerau-Levenshtein distance` <https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance> 来查找最多包含两个更改的所有项，其中更改是单个字符的插入，删除或替换，或两个相邻字符的转置。默认编辑距离为2，但编辑距离为1应足以捕获所有人为拼写错误的 80％ 。编辑距离可以在 `~` 后跟一个数字来表示

```txt
zhagnsan~1
```

#### 范围

可以为日期，数字或字符串字段指定范围。包含范围边界的使用方括号 `[min TO max]` 和不含范围边界的使用大括号 `{min TO max}` 。

下面的表达式是 2018 年 8 月

```json
date:[2018-08-01 TO 2018-08-31]
```

1 - 9 的数字

```json
num:[1 TO 9]
```

大于等于 10 的数字，下面使用 `*` 表示开区间，也可以使用 `>=` 或 `<=` 表示开区间。

```json
num:[10 TO *]
num:>=10
```

大于等于 1 小于 5 的数字，注意到大括号和中括号可以混合使用

```json
num:[1 TO 5}
```

