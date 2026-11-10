
## Nacos 和 Apollo，哪个更好用？

配置管理这个看似不起眼的环节，一旦出现问题，轻则功能异常，重则全站崩溃。

正因如此，配置中心成为了微服务架构中的标配。

而说到配置中心，目前国内最热门的两个选择非**Nacos**和**Apollo**莫属。

很多团队在选型时都会纠结：到底哪个更好用？

今天这篇文章就专门跟大家聊聊这个话题，希望对你会有所帮助。

## 01 设计理念

在深入技术细节之前，我们先看看这两个框架的“出身”和设计理念，这决定了它们的气质和边界。

**Nacos**（Naming and Configuration Service）是阿里巴巴2018年开源的项目，它的野心从一开始就不只是配置中心——它想成为微服务的“一站式”基础设施，同时提供服务发现和配置管理两大核心功能。

这意味着，如果你用了Nacos，你可能就无需再引入Eureka或Consul了。

它的口号是“一个更易于构建云原生应用的动态服务发现、配置和管理平台”。

**Apollo**（阿波罗）则来自携程，2016年开源。

它从一开始就**只专注做配置管理这一件事**，并在这个领域深耕细作。

携程作为一家大型在线旅游平台，内部有着极其复杂的配置治理需求（多环境、多集群、权限控制、灰度发布等），Apollo正是为了解决这些企业级痛点而生的。

> 一个简单的比喻：Nacos像一个功能丰富的“瑞士军刀”，既能切菜（配置）又能开瓶（服务发现）；Apollo则像一把顶级的“德国双立人厨师刀”，它只切菜，但在这个领域做到了极致。

## 02 谁更容易上手？

对于“好不好用”，最直观的感受就是**上手成本**。

我们先分别跑一个简单的Demo，看看两者的启动和集成难度。

### Nacos：极简启动

```bash
# 下载最新版Nacos Server
wget https://github.com/alibaba/nacos/releases/download/2.2.3/nacos-server-2.2.3.zip
unzip nacos-server-2.2.3.zip
cd nacos/bin

# 单机模式启动（Linux/Mac）
sh startup.sh -m standalone
# Windows
startup.cmd -m standalone
```

启动成功后，访问 `http://localhost:8848/nacos`，默认用户名密码都是 `nacos`。

然后创建一个Spring Boot项目，引入依赖：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2021.0.5.0</version>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

在 `bootstrap.properties` 中配置：

```properties
spring.application.name=my-service
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
spring.cloud.nacos.config.file-extension=properties
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
```

然后就可以在代码中使用了：

```java
@RestController
@RefreshScope // 配置自动刷新注解
public class DemoController {
    
    @Value("${user.name:default}")
    private String userName;
    
    @GetMapping("/config")
    public String getConfig() {
        return "Current user name: " + userName;
    }
    
    // 服务发现客户端注入
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @GetMapping("/services")
    public List<String> services() {
        return discoveryClient.getServices();
    }
}
```

**从下载到跑通第一个配置读取，熟练的话不超过10分钟**。

Nacos的一体化设计让配置和发现共用同一个服务端，客户端配置也非常简洁。

### Apollo：稍显复杂的部署

Apollo的部署稍微复杂一些，因为它本身就是一套分布式系统。它包含四个核心组件：

- **ConfigService**：提供配置读取和推送接口
- **AdminService**：提供配置管理接口
- **Portal**：Web管理界面
- **MetaServer**：服务发现（实际上是Eureka）

官方提供了Quick Start脚本，但依然需要几步操作：

```bash
# 下载Quick Start包
wget https://github.com/apolloconfig/apollo-quick-start/releases/download/v2.1.0/apollo-quick-start-2.1.0.zip
unzip apollo-quick-start-2.1.0.zip
cd apollo-quick-start
# 执行脚本（会自动启动MySQL和所有服务）
./demo.sh start
```

启动成功后，访问 `http://localhost:8070`，默认用户名/密码：apollo/admin。

Spring Boot集成：

```xml
<dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-client</artifactId>
    <version>2.1.0</version>
</dependency>
```

在 `application.properties` 中配置：

```properties
app.id=my-service
apollo.meta=http://localhost:8080
apollo.bootstrap.enabled=true
apollo.bootstrap.namespaces=application
```

代码中使用：

```java
@Component
public class ApolloConfigBean {
    
    // 使用@Value注解，需要添加@EnableApolloConfig注解
    @Value("${user.name:default}")
    private String userName;
    
    // 也可以直接使用API获取
    public String getConfig(String key) {
        Config config = ConfigService.getAppConfig();
        return config.getProperty(key, "default");
    }
    
    // 监听配置变化
    @ApolloConfigChangeListener
    private void onChange(ConfigChangeEvent changeEvent) {
        for (String key : changeEvent.changedKeys()) {
            System.out.println(key + " changed!");
        }
    }
}
```

**小结**：在“开箱即用”这个维度上，**Nacos完胜**。它的单机模式启动简单，客户端配置也少。Apollo的Quick Start虽然也提供了脚本，但因为组件多，启动稍慢，而且会占用更多端口。对于只想快速尝鲜的开发者来说，Nacos更友好。

## 03 核心功能

“好用”不只是启动快，更重要的是在复杂场景下能不能解决问题。

我们逐一对比关键功能点。

### 3.1 配置管理基础能力

**配置格式支持**：两者都支持properties、yaml、xml、json等常见格式。Nacos的控制台支持直接编辑yaml，并提供了语法高亮；Apollo也支持文本编辑，但默认按key-value展示，对于yaml来说体验稍差。

**版本管理**：两者都支持配置的历史版本和回滚。Apollo的版本管理粒度更细，可以精确到每次发布，并支持对比、查看变更详情。Nacos也有版本管理，但相对简单。

**环境与集群管理**：Apollo天生支持多环境（DEV、FAT、UAT、PRO）和多集群（数据中心），每个环境的配置完全隔离，这对于企业级应用非常友好。Nacos通过命名空间（Namespace）和分组（Group）来实现隔离，理论上可以模拟出环境，但需要自己规划命名策略，没有Apollo那么直观。

举个例子，在Apollo中，创建一个DEV环境的配置和一个PRO环境的配置，完全是两个独立的配置项，互不影响。而在Nacos中，你可能需要在不同的Namespace中分别创建。

### 3.2 配置推送的实时性

#### Nacos的长轮询（Long Polling）机制

Nacos采用**HTTP长轮询**实现配置推送。我来带你看看源码级别的实现：

```java
// Nacos客户端发起长轮询的核心逻辑
public void start() {
    executor.scheduleWithFixedDelay(new LongPollingRunnable(), 0, retryInterval, TimeUnit.SECONDS);
}

class LongPollingRunnable implements Runnable {
    @Override
    public void run() {
        // 构造订阅串，包含当前配置的md5值
        String listeningConfigs = buildListeningConfigs();
        
        Request request = new Request.Builder()
            .url(serverAddr + "/v1/cs/configs/listener")
            .post(body)
            .build();
        
        // 这里会阻塞最多30秒
        try (Response resp = httpClient.newCall(request).execute()) {
            if (resp.isSuccessful()) {
                String content = resp.body().string();
                // 如果有变更，立即拉取最新配置
                if (StringUtils.isNotEmpty(content)) {
                    pullNewConfig(content);
                }
            }
        }
    }
}
```

服务端的处理更精妙：
1. 客户端请求到达后，服务端先比较md5，如果有变更，立即返回
2. 如果没有变更，使用Servlet 3.0的`AsyncContext`挂起请求（默认29.5秒）
3. 同时启动一个定时任务，超时后返回空响应
4. 如果期间配置变更，触发`LocalDataChangeEvent`事件，从挂起队列中找出相关客户端，立即返回

```java
// 服务端挂起请求的核心代码
AsyncContext asyncContext = request.startAsync();
asyncContext.setTimeout(0);  // 禁用容器默认超时

ClientSubscription sub = new ClientSubscription(keys, asyncContext);
// 加到订阅列表
subscribers.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>()).add(sub);

// 定时29.5秒后超时返回
ScheduledFuture<?> timeoutTask = scheduler.schedule(() -> {
    asyncContext.getResponse().getWriter().write("");
    asyncContext.complete();
}, 29500, TimeUnit.MILLISECONDS);
```

这种设计的精妙之处在于：**既实现了准实时推送（秒级），又避免了WebSocket长连接的资源消耗，同时兼容所有HTTP中间件**。

#### Apollo的定时轮询机制

Apollo客户端默认每**60秒**拉取一次最新配置。虽然可以通过`apollo.refreshInterval`调整，但本质上还是轮询。源码层面，Apollo使用Spring的`DeferredResult`实现异步化：

```java
// Apollo服务端关键类：NotificationControllerV2
// 使用DeferredResult实现异步响应
@RequestMapping(method = RequestMethod.GET)
public DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> pollNotification(
    @RequestParam(value = "appId") String appId,
    @RequestParam(value = "cluster") String cluster,
    @RequestParam(value = "notifications") String notificationsAsString,
    @RequestParam(value = "dataCenter", required = false) String dataCenter,
    @RequestParam(value = "ip", required = false) String clientIp) {
    
    DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> deferredResult = 
        new DeferredResult<>(timeout);
    
    // 将deferredResult存入队列，等待配置变更时唤醒
    deferredResults.add(deferredResult);
    
    return deferredResult;
}
```

**两种机制的对比**：

| 维度 | Nacos长轮询 | Apollo定时轮询 |
|------|------------|----------------|
| 推送延迟 | 秒级（1-2秒） | 依赖轮询间隔，默认60秒 |
| 服务端压力 | 中等（需维护挂起队列） | 较低（无状态） |
| 连接数 | 与客户端数量正相关 | 与轮询频率相关 |
| 穿透性 | 好（纯HTTP） | 好（纯HTTP） |

如果业务对配置变更实时性要求高（如秒杀开关、黑白名单），Nacos的长轮询更有优势；如果能接受分钟级延迟，Apollo也能满足大多数场景。


### 3.3 灰度发布与权限控制

这是Apollo的绝对强项，也是很多大厂选择它的核心原因。

**Apollo的灰度发布**：支持按IP、按机器、按用户标签进行配置的灰度发布。你可以先发布给一台机器验证，没问题再逐步推送到全量。同时支持配置的**全量回滚**和**灰度回滚**，操作界面非常友好。

**Apollo的权限控制**：提供应用级、环境级、命名空间级的多维度权限管理。你可以指定某个用户只能修改DEV环境的配置，不能碰PRO；或者只允许运维人员执行发布操作。所有的操作都有审计日志，谁在什么时间改了哪个配置，一目了然。

相比之下，Nacos的权限控制比较基础，虽然也支持RBAC，但粒度较粗。灰度发布需要自己通过多Namespace或多Group模拟，或者结合配置中心的元数据功能手动实现，对于大多数团队来说门槛较高。

### 3.4 服务发现功能

这是Nacos的独有优势，Apollo**完全不支持**服务发现。

Nacos的服务发现功能非常强大，支持健康检查、临时/持久实例、权重路由、监听查询等，完全可以替代Eureka或Consul。如果你的项目既需要配置中心，又需要服务发现，使用Nacos可以少维护一套系统，架构更简洁。

```java
// Nacos服务发现客户端使用示例
@RestController
public class ConsumerController {
    
    @Autowired
    private LoadBalancerClient loadBalancerClient;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/call")
    public String call() {
        // 从Nacos获取服务实例
        ServiceInstance instance = loadBalancerClient.choose("provider-service");
        String url = "http://" + instance.getHost() + ":" + instance.getPort() + "/hello";
        return restTemplate.getForObject(url, String.class);
    }
}
```

如果选择Apollo，你需要额外引入一套服务发现组件，比如Eureka、Nacos Discovery或Consul，增加了系统的复杂度。

## 04 数据模型对比

很多小伙伴在工作中分不清两者的数据模型，特别是“Namespace”这个概念，两者含义完全不同。

### 4.1 Nacos的三层模型

Nacos采用 **Namespace + Group + DataId** 三层结构：

- **Namespace**：代表**环境隔离**，如dev、test、prod。客户端通过指定Namespace来访问不同环境的配置。
- **Group**：代表**逻辑分组**，如DEFAULT_GROUP、DATABASE_GROUP。
- **DataId**：代表**具体的配置文件**，通常格式为 `${serviceName}.properties`。

```yaml
# Nacos配置示例
spring.cloud.nacos.config.namespace=dev          # 开发环境
spring.cloud.nacos.config.group=DEFAULT_GROUP    # 默认分组
spring.cloud.nacos.config.dataId=order-service.properties
```

这种模型的好处是结构清晰，与Kubernetes的命名空间概念类似，理解成本低。但坏处是，如果你需要管理多个环境，必须在每个环境部署不同的Nacos集群，或者通过权限隔离来控制访问。

### 4.2 Apollo的四层模型

Apollo的数据模型更复杂：**Environment + AppId + Cluster + Namespace**。

- **Environment**：环境，如DEV、FAT、UAT、PRO。
- **AppId**：应用唯一标识。
- **Cluster**：集群，可用于区分数据中心或逻辑分组。
- **Namespace**：在这里Namespace是**配置文件**的概念，分为私有（private）和公共（public）两种。

```properties
# Apollo配置示例
app.id=order-service
apollo.meta=http://config-server:8080
apollo.bootstrap.namespaces=application,TEST1.public
apollo.cluster=default
```

Apollo的模型粒度更细，天然支持多环境、多集群，适合复杂的企业部署架构。

但缺点也很明显——配置项过多，初次上手容易迷失。


## 05 架构对比

我们用两张图来直观对比两者的架构。

### 5.1 Nacos架构


![Nacos架构 示意图1](https://web-1259428203.cos.ap-chengdu.myqcloud.com/1af2a31a-a55c-4544-af43-686470eca2ba.png)


Nacos集群中各节点是对等的，通过共享存储（MySQL）来保证数据一致性。

客户端可以配置多个服务端地址，或者通过负载均衡器接入。架构相对简单。

### 5.2 Apollo架构


![Apollo架构 示意图2](https://web-1259428203.cos.ap-chengdu.myqcloud.com/3c75ec1b-3b62-412c-bf17-e93a78c875f8.png)


Apollo的架构组件较多，职责分离清晰：
- **ConfigService**：负责配置的读取和推送
- **AdminService**：负责配置的修改和发布
- **Portal**：Web管理界面
- **MetaServer**：基于Eureka的服务发现，帮助客户端找到ConfigService

这种架构的优点是每个组件都可以独立扩展，比如ConfigService可以水平扩展，而AdminService可以单独隔离。

缺点是部署和维护成本高，需要同时管理多个组件和数据库。

## 06 性能与扩展性

在性能方面，两者都能满足绝大多数场景。

我们曾做过压测，单台Nacos Server在8核16G配置下，可以稳定支撑5000+客户端的长轮询连接。

Apollo的ConfigService也是无状态的，扩展性很好。

但Apollo的一个潜在瓶颈是数据库。

因为所有的配置拉取都会查询数据库（虽然有缓存），当客户端数量非常大时，数据库压力会上升。

Nacos同样依赖数据库，但它在服务端有一层缓存，可以减少数据库查询。

在数据一致性方面，两者都依赖于数据库事务，保证了配置发布的原子性。

## 07 运维成本与社区生态

**部署运维**：Nacos明显更简单。单机模式一条命令，集群模式也只需要配置数据库和几个节点。Apollo最少需要启动三个组件（ConfigService、AdminService、Portal），且需要两个数据库（ConfigDB和PortalDB），对运维人员的要求更高。

**监控告警**：Nacos自带了简易的监控页面，可以查看集群状态、健康检查等。Apollo也有监控，但需要整合到企业监控系统。

**社区生态**：
- **Nacos**：背靠阿里云，社区非常活跃，几乎每个月都有新版本，GitHub Star已超过27k。文档齐全，有中文社区。与Spring Cloud Alibaba、Dubbo等框架集成良好。
- **Apollo**：携程开源，社区也很成熟，Star数约28k，但发布频率相对较低（约半年一个大版本）。文档详细，有企业微信交流群。

## 08 如何选择？

结合以上对比，我给出一些选型建议：

### 推荐选择 Nacos 的场景
1. **新项目，技术栈为Spring Cloud Alibaba**：天然集成，配置+发现一体，开发效率高。
2. **中小团队，运维人力有限**：部署简单，维护成本低。
3. **需要服务发现功能**：用Nacos可以省掉一套Eureka或Consul。
4. **对配置变更实时性要求较高**：长轮询机制秒级生效。

### 推荐选择 Apollo 的场景
1. **大型企业，有复杂的配置治理需求**：多环境、多集群、严格的权限控制和灰度发布。
2. **配置中心需要与现有CMDB、监控系统深度集成**：Apollo的开放API和丰富事件机制更适合。
3. **对配置变更的安全性要求极高**（金融、政务）：完整的审计日志和细粒度权限控制。
4. **团队已经有一套成熟的服务发现方案**（如自研或Consul），只需要配置中心。

### 折中方案
也有一些团队采用**混合架构**：服务发现用Nacos（因为它轻量），配置管理用Apollo（因为它专业）。

这种方案虽然增加了组件，但可以发挥各自优势，适合大型项目。

## 09 如何实现配置动态刷新？

最后，给出一个示例，展示在两种框架下如何实现配置的动态刷新。

### Nacos 动态刷新

```java
@RestController
@RefreshScope
public class NacosConfigController {
    
    @Value("${my.config:default}")
    private String myConfig;
    
    @GetMapping("/nacos/config")
    public String getConfig() {
        return "Current config: " + myConfig;
    }
}
```

在Nacos控制台修改配置后，无需重启，调用接口就会看到新值，`@RefreshScope`是关键。

### Apollo 动态刷新

```java
@RestController
@RefreshScope
public class ApolloConfigController {
    
    @Value("${my.config:default}")
    private String myConfig;
    
    @GetMapping("/apollo/config")
    public String getConfig() {
        return "Current config: " + myConfig;
    }
}
```

看起来几乎一样，但需要注意两点：
1. 需要`@EnableApolloConfig`注解（通常在启动类或配置类上加）。
2. Apollo默认的刷新时机是60秒轮询，如果希望立即生效，可以调用`ConfigService`的API强制刷新，或集成消息总线。

## 总结

Nacos和Apollo没有绝对的“更好”，只有“更适合”。

如果让我用一个词形容它们，我会说：

- **Nacos**：**轻盈、全能、云原生**
- **Apollo**：**专业、严谨、企业级**

有些小伙伴可能会说：“我们团队小，用哪个都行。” 

确实，对于很多初创项目，两者都能满足需求。

但作为架构师，我们需要考虑未来3-5年的发展：业务会变复杂，团队会扩大，配置治理的挑战也会随之而来。选型时多思考一步，未来会少踩很多坑。

回到最初的问题：Nacos和Apollo，哪个更好用？

我的答案是：

- **如果你追求简单高效、配置和服务发现一体化**，选Nacos，它真的很好用。
- **如果你身处大型组织，需要精细化的配置治理和安全性**，选Apollo，它在复杂场景下的好用程度远超你的想象。

<!-- article-related-reading:start -->
## 相关阅读

- [为什么越来越多人用Nacos？](./why-more-people-use-nacos.html)
- [Nacos已全面拥抱AI！](./nacos-embraces-ai.html)
- [Milvus 和 PGVector，哪个更好？](./milvus-vs-pgvector.html)

<!-- article-related-reading:end -->
