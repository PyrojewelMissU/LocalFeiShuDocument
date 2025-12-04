# &#x20;1.Nacos

***

## 1.1简介与安装

• **Nacos&#x20;**/nɑ:kəʊs/ 是 Dynamic **Na**ming and **Co**nfiguration **S**ervice的首&#x20;

字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。&#x20;

• 官网：[Nacos官网](https://nacos.io/)

• 安装：&#x20;

• 下载安装包【2.4.3】&#x20;

• 启动命令： `startup.cmd -m standalone`

下载好最近的安装包后，解压到非中文目录，进入 `bin` 目录，输入cmd，在cmd窗口中执行启动命令。

![](images/image-11.png)

## 1.2服务注册

1. 引入`spring-boot-starter-web`、`spring-cloud-starter-alibaba-nacos-discovery` 依赖

2. 编写主启动类，编写配置文件

3. 配置 Naocs 地址

```yaml
spring:
  cloud:
    nacos:
      # 配置 Nacos 地址
      server-addr: 127.0.0.1:8848
```

* 启动微服务

* 查看注册中心效果，访问 `http://localhost:8848/nacos/`

* 测试集群模式启动：单机情况下通过改变端口号模拟微服务集群，例如添加 Program arguments 信息为 `--server.port=8001`

* 访问nacos注册中心，如果能在服务管理的服务列表看到如下图片，则代表集群模式启动成功

![](images/image-13.png)

## 1.3服务发现

***

1. 开启服务发现，在主启动类上添加 `@EnableDiscoveryClient` 注解

1) 测试两款 API 的服务发现功能：`DiscoveryClient` 和 `NacosServiceDiscovery`。前者为 Spring 提供的服务发现标准接口，后者由 Nacos 提供。

```java
//测试服务发现Api
@Autowired
DiscoveryClient discoveryClient;

@Autowired
NacosServiceDiscovery nacosServiceDiscovery;

@Test
void nacosServiceDiscoveryTest() throws NacosException {
    for (String service : nacosServiceDiscovery.getServices()) {
        System.out.println("service = " + service);
        List<ServiceInstance> instances = nacosServiceDiscovery.getInstances(service);
        for (ServiceInstance instance : instances) {
            System.out.println("ip: " + instance.getHost() + " port:=" + instance.getPort());
        }
    }
}

@Test
void discoveryClientTest(){
    for (String service : discoveryClient.getServices()) {
        System.out.println("service = " + service);
        //获取ip + port
        List<ServiceInstance> instances = discoveryClient.getInstances(service);
        for (ServiceInstance instance : instances) {
            System.out.println("ip: " + instance.getHost() + " port:=" + instance.getPort());
        }
    }
}
```

## 1.4远程调用

***

远程调用基本流程：

![](images/diagram.png)

远程调用-**下单场景：**

![](images/diagram-1.png)

> 小知识：将项目中所用到的实体类都存放在model模块中，便于管理，并在Services模块中导入model层依赖，确保service-order和service-product都依赖了定义的公共模型model

![](images/image-6.png)



## 1.5负载均衡

* 使用`LoadBalancerClient`实现

1.首先一定要在`pom.xml`中导入`spring-cloud-starter-loadbalancer`

2.然后在`OrderServiceImpl`中注入`LoadBalancerClient`,调用其`choose()`方法，传入服务名，实现负载均衡。

```java
@Autowired //这里别导错包了 是import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
LoadBalancerClient loadBalancerClient;

//进阶2：完成负载均衡发送请求
private Product getProductFromRemoteWithLoadBalance(Long productId){
    //1 获取到商品服务所在的所有机器IP + port
    ServiceInstance choose = loadBalancerClient.choose("service-product");

    //远程URL
    String url = "http://" + choose.getHost() + ":" + choose.getPort() + "/product/" + productId;

    log.info("远程请求：{}",url);

    //2. 给远程发送请求
    Product product = restTemplate.getForObject(url, Product.class);
    return product;
}
```

* 使用`@LoadBalanced`注解实现

在配置类中向 Spring 容器添加 `RestTemplate` 的 Bean，在 Bean 方法上添加 `@LoadBalanced` 注解，使用 `RestTemplate` 进行远程调用时，修改传入的 URL 为服务名，比如：

```java
@Configuration
public class OrderConfig {
    @LoadBalanced//注解式负载均衡
    @Bean
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

```java
//进阶3：基于注解的负载均衡
private Product getProductFromRemoteWithLoadBalancerAnnotation(Long productId) {
    // 给远程发送请求：service-product 会被动态替换
    String url = "http://service-product/product/" + productId;
    log.info("远程请求: {}", url);
    // 给远程发送请求
    return restTemplate.getForObject(url, Product.class);
}
```

此时底层会将服务名替换为负载均衡后的目标 URL。

> 经典面试题：如果注册中心宕机，远程调用是否可以成功？

![](images/diagram-2.png)

**两种情况：**

* 如果从未调用过，此时注册中心宕机，调用会立即失败

* 如果调用过：

  * 此时注册中心宕机，会因为存在缓存的服务信息，调用会成功

  * 如果注册中心和对方服务都宕机，因为会缓存名单，调用会阻塞后失败（Connection Refused）

## 1.6配置中心

### 1.6.1创建配置

***

1. 由于每个微服务都需要配置中心，所以先要在Services层导入`nacos`作为配置中的依赖

```xml
<dependency> 
<groupId>com.alibaba.cloud</groupId> 
<artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId> 
</dependency>
```

* 再在`application.properties`配置

```properties
spring.cloud.nacos.server-addr=127.0.0.1:8848 
spring.config.import=nacos:service-order.properties
```

* 在nacos配置中心中创建data-id（数据集）

![](images/image-8.png)

![](images/image-9.png)

配置好后点击发送，即可看到

![](images/image-10.png)

启动服务之后访问，`localhost:port/config`

就可以获取到配置中心的值，创建配置成功。

![](images/image-12.png)

### 1.6.2配置刷新

***

配置中心的动态刷新步骤：

* `@Value("${xx}")` 获取配置 + `@RefreshScope` 实现动态刷新

配置创建好后，我们再在配置中心对配置进行修改，当配置改了之后再次访问

`localhost:port/config`，并没有获取到修改后的配置，原因在于：如果要

实现配置刷新，还需要配合一个注解`@RefreshScope`

```properties
spring.cloud.nacos.server-addr=127.0.0.1:8848
spring.config.import=nacos:service-order.properties
```

```java
@RefreshScope//自动刷新配置中心修改的配置
@RestController
public class OrderController {
    
    @Value("${order.timeout}")
    String orderTimeout;
    @Value("${order.auto-confirm}")
    String orderAutoConfirm;

    @Autowired
    OrderProperties orderProperties;

    @GetMapping("/config")
    public String config(){
        return "order.timeout" + orderTimeout
                + "：order.auto-confirm=" + orderAutoConfirm;
    }
}
```

此时会有一个小问题：一旦引用了配置中心，项目启动时还没有导入任何配置，就会报错，报错信息为：`No spring.config.import property has been defined`，就是说我们没有指定spring.config.import这个属性。此时可以在`proproperties`文件中设置将nacos的导入检查功能给禁用掉：`spring.cloud.nacos.config.import-check.enabled=false`

此时再启动服务，就没问题了

***

* `@ConfigurationProperties` 无感自动刷新

首先将`application.properties`文件中配置中心的配置都删掉，然后创建一个配置类`OrderProperties`，将配置中心的配置属性都添加到这个类中

```properties
@Component
//配置批量绑定在nacos下，可以无需@RefreshScope就能实现自动刷新
@ConfigurationProperties(prefix = "order") 
@Data
public class OrderProperties {
    String timeout;
    String autoConfirm;
    String dbUrl;
}
```

然后在`OrderController`中注入`OrderProperties`

```properties
@RestController
public class OrderController {

    @Autowired
    OrderProperties orderProperties;
    
    @GetMapping("/config")
    public String config(){
        return "order.timeout" + orderProperties.getTimeout()
                + "：order.auto-confirm=" + orderProperties.getAutoConfirm()
                + "order.db-url="+orderProperties.getDbUrl();
    }
}
```

* `NacosConfigManager` 监听配置变化

假设我们现在希望`nacos`里面`service-order-properties`这个配置中心的配置任何一项的配置属性发生了变化，去告诉我，这一项现在新的值是什么，然后发一个邮件

```properties
@EnableDiscoveryClient
@SpringBootApplication
public class OrderMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderMainApplication.class, args);
    }

    //1. 项目启动就监听配置文件变化
    //2. 发生变化后拿到变化值
    //3. 发送邮件

    @Bean
    ApplicationRunner applicationRunner(NacosConfigManager nacosConfigManager){
        return args -> {
            ConfigService configService = nacosConfigManager.getConfigService();
            configService.addListener("service-order.properties",
                    "DEFAULT_GROUP", new Listener() {
                        @Override
                        public Executor getExecutor() {
                            return Executors.newFixedThreadPool(4);
                        }

                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            System.out.println("变化的配置信息：" + configInfo);
                            System.out.println("邮件通知...");
                        }
                    });
        };
    }
}
```

这就是通过编码的方式，使用`NacosConfigManager` 实时监听指定数据集里面的配置变化

**总结：**

配置中心的动态刷新步骤：

* `@Value("${xx}")` 获取配置 + `@RefreshScope` 实现动态刷新

* `@ConfigurationProperties` 无感自动刷新

* `NacosConfigManager` 监听配置变化

### 1.6.3面试题

> Nacos中的数据集和application.properties有相同的配置项，哪个生效？

![](images/diagram-3.png)

* 先导入优先：下面一行代码导入了两个配置分别是`nacos:service-order.properties`和`nacos:common.properties`，此时配置生效的只有第一个`nacos:service-order.properties`

```properties
spring.config.import=nacos:service-order.properties,nacos:common.properties
```



## 1.7数据隔离

***

一个项目通常部署在多套环境上，比如dev(开发)，test(测试)，prod(生产)。

项目中每个微服务的配置信息在每套环境上的值可能不一样，要求项目可以通过切换环境，加载本环境的配置。

如果要完成以上需求，其中的难点是如何：

* 区分多套环境

* 区分多种微服务

* 区分多种配置

* 按需加载配置

![](images/diagram-4.png)

**`namespace`、`dataId`、`group`配合`spring.config.activate.on-profile`实现配置环境隔离**

**`spring.profiles.active: dev`**：默认激活 `dev` 环境（开发环境）。

**`spring.cloud.nacos.config.import-check.enabled: false`**：关闭 Spring Boot 3.1 新增的 “config import check”，避免启动时找不到配置报错。

**`spring.cloud.nacos.config.namespace`**：Nacos 配置的命名空间使用当前激活的 profile（`dev`/`test`/`prod`），如果没有就用 `public`。

```yaml
server:
  port: 8080                 # 应用服务端口，默认启动在 8080

spring:
  profiles:
    active: dev              # 默认激活 dev 环境（开发环境）
  application:
    name: service-order      # Spring 应用名称（注册到 Nacos 时的服务名）
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848   # Nacos 配置中心/注册中心地址
      config:
        import-check:
          enabled: false     # 关闭 Spring Boot 3.1+ 的 import 校验，避免启动时报错
        namespace: ${spring.profiles.active:public} # 根据当前环境选择 Nacos 命名空间，没有就用 public

# ---------------------------------------------------------
# 以下是多文档配置，根据不同的 profile 激活不同的 Nacos 配置
# ---------------------------------------------------------

---
spring:
  config:
    import:
      - nacos:common.properties?group=order    # 从 Nacos 的 order 组导入 common.properties
      - nacos:database.properties?group=order  # 从 Nacos 的 order 组导入 database.properties
    activate:
      on-profile: dev          # 当 spring.profiles.active = dev 时生效（开发环境）

---
spring:
  config:
    import:
      - nacos:common.properties?group=order    # 同上
      - nacos:database.properties?group=order  # 同上
      - nacos:haha.properties?group=order      # 仅测试环境多导入 haha.properties
    activate:
      on-profile: test         # 当 spring.profiles.active = test 时生效（测试环境）

---
spring:
  config:
    import:
      - nacos:common.properties?group=order    # 同上
      - nacos:database.properties?group=order  # 同上
      - nacos:hehehe.properties?group=order    # 仅生产环境多导入 hehehe.properties
    activate:
      on-profile: prod         # 当 spring.profiles.active = prod 时生效（生产环境）

```

Nacos 的解决方案：

* 用名称空间区分多套环境

* 用 Group 区分多种微服务

* 用 Data-id 区分多种配置

* 使用 SpringBoot 激活对应环境的配置

## 1.7Nacos总结

***

![](images/image-14.png)



# 2.OpenFeign

## 2.1 简介与使用

OpenFeign，是一种 Declarative REST Client，即声明式 Rest 客户端，与之对应的是编程式 Rest 客户端，比如 RestTemplate。

OpenFeign 由注解驱动：

* 指定远程地址：`@FeignClien`

* 指定请求方式：`@GetMapping`、`@PostMapping`、`@DeleteMapping`...

* 指定携带数据：`@RequestHeader`、`@RequestParam`、`@RequestBody`...

* 指定返回结果：响应模式

其中的 `@GetMapping` 等注解可以沿用 Spring MVC：

* 当它们标记在 Controller 上时，用于接收请求

* 当他们标记在 FeignClien 上时，用于发送请求

![](images/diagram-5.png)

> 远程调用注册中心中的服务 `ProductFeignClient`

```java
@FeignClient(value = "service-product")//feign客户端
public interface ProductFeignClient {
    //mvc注解的两套使用逻辑
    //1. 标注在Controller上，是接受这样的请求
    //2. 标注在FeignClient上，是发送这样的请求
    @GetMapping("/product/{id}")
    Product getProductById(@PathVariable("id") Long id);
}
```

```java
@Override
public Order createOrder(Long productId, Long userId) {
    //Product product = getProductFromRemoteWithLoadBalanceAnnotation(productId);
    //使用Feign完成远程调用
    Product product = productFeignClient.getProductById(productId);
    Order order = new Order();
    order.setId(1L);
    //总金额
    order.setTotalAmount(product.getPrice().multiply(new BigDecimal(product.getNum())));
    order.setUserId(userId);
    order.setNickName("zhangsan");
    order.setAddress("尚硅谷");
    //远程查询商品列表
    order.setProductList(Arrays.asList(product));

    return order;
}
```

> 远程调用指定URL `QWeatherClient` 访问的是和风天气

```typescript
@FeignClient(name = "qweatherClient", url = "https://nc5u9vnpd6.re.qweatherapi.com")
public interface QWeatherClient {

    @GetMapping("/v7/weather/now")
    String getNowWeather(@RequestParam("location") String location,
                         @RequestHeader("X-QW-Api-Key") String apiKey,
                         @RequestHeader(value = "Accept-Encoding", defaultValue = "gzip") String encoding);
}
```

```typescript
@SpringBootTest
public class WeatherTest {
    @Autowired
    private QWeatherClient qWeatherClient;
    @Test
    public void testNowWeather() {
        String apiKey = "21a144c7722e46d4a4524a07cd92fb53"; // 直接写死测试用
        String location = "101050910"; // 北京
        String result = qWeatherClient.getNowWeather(location, apiKey, "gzip");
        System.out.println(result);
    }
}
```

使用时引入以下依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

在主启动类上使用注解：`@EnableFeignClients`

![](images/diagram-6.png)

## 2.2 小技巧

如何编写好 OpenFeign 声明式的远程调用接口：

* 针对业务 API：直接复制对方的 Controller 签名即可；

* 第三方 API：根据接口文档确定请求如何发

## 2.3 一道面试题

客户端负载均衡与服务端负载均衡的区别：

![](images/diagram-7.png)

![](images/diagram-8.png)

## 2.4 进阶用法

### 2.4.1 日志

在配置文件中指定 feign 接口所在包的日志级别：

```xml
logging:
  level:
    #指定feign接口所在的包的日志级别为debug级别
    com.yan.order.feign: debug
```

向 Spring 容器中注册 `feign.Logger.Level` 对象：

```xml
@Bean
public Logger.Level feignlogLevel() {
    // 指定 OpenFeign 发请求时，日志级别为 FULL
    return Logger.Level.FULL;
}
```

### 2.4.2 超时控制

![](images/diagram-9.png)

连接超时（connectTimeout），默认 10 秒。

读取超时（readTimeout），默认 60 秒。

如果需要修改默认超时时间，在配置文件中进行如下配置：

```yaml
spring:
  profiles:
    active: dev
    # 该配置表示在当前应用启动时，会包含并加载名为 feign 的配置文件或配置集。
    # 用途：通常用于启用与 Feign 相关的配置。Feign 是 Spring Cloud 中用于声明式 REST 客户端的一个组件，
    # 通过 include: feign 可以加载 Feign 的相关设置，比如日志、超时时间等。
```

```yaml
连接超时（connectTimeout），默认 10 秒。
读取超时（readTimeout），默认 60 秒。
如果需要修改默认超时时间，在配置文件中进行如下配置：
spring:
  cloud:
    openfeign:
      client:
        config:
          # 默认配置default:
            logger-level: fullconnect-timeout: 1000read-timeout: 2000# 具体 feign 客户端的超时配置service-product:
            logger-level: full# 连接超时，3000 毫秒connect-timeout: 3000# 读取超时，5000 毫秒read-timeout: 5000
```

### 2.4.3 重试机制

远程调用超时失败后，还可以进行多次尝试，如果某次成功则返回 ok，如果多次尝试后依然失败则结束调用，返回错误。

OpenFeign 底层默认使用 `NEVER_RETRY`，即从不重试策略。

向 Spring 容器中添加 `Retryer` 类型的 Bean：

```java
/**
 * 配置Feign客户端的重试器
 * 默认情况下，Feign会使用Retryer.Default进行重试
 * 重试机制可以在服务调用失败时自动重试，提高系统的容错能力
 * @return Retryer实例
 */
@Bean
public Retryer retryer() {
    // 使用默认的重试策略，初始间隔为100ms，最大重试次数为5次
    return new Retryer.Default();
}
```

这里使用 OpenFeign 的默认实现 `Retryer.Default`，在这种默认实现下：

```java
public Default() {
    this(100L, TimeUnit.SECONDS.toMillis(1L), 5);
}
```

OpenFeign 的重试规则是：

* 重试间隔 100ms

* 最大重试间隔 1s。新一次重试间隔是上一次重试间隔的 1.5 倍，但不能超过最大重试间隔。

* 最多重试 5 次

### 2.4.4 拦截器

![](images/diagram-10.png)

以请求拦截器为例，自定义的请求拦截器需要实现 `RequestInterceptor` 接口，并重写 `apply()` 方法：

```typescript
/**
 * 请求拦截器，为每个请求添加X-Token头部
 * @param template 请求模板
 */
@Override
public void apply(RequestTemplate template) {
    // 生成一个随机UUID作为token值
    String token = UUID.randomUUID().toString();
    // 将X-Token头部添加到请求中
    template.header("X-Token", token);
}
```

要想要该拦截器生效有两种方法：

1. 在配置文件中配置对应 Feign 客户端的请求拦截器，此时该拦截器只对指定的 Feign 客户端生效

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          # 默认配置
          default:
            logger-level: full
            connect-timeout: 1000
            read-timeout: 2000
          # 具体 feign 客户端
          service-product:
            # 该请求拦截器仅对当前客户端有效
            request-interceptors:
              - com.yan.order.interceptor.XTokenRequestInterceptor
              
```

* 请求拦截器在项目中经常会被用到，用来在整个分布式调用链路中，每次远程调用之前都可以使用请求拦截器，把这个链路上想要共享的数据给它放到请求头或请求体中，然后再把请求发出去，那么这个链路的下游就能收到这个数据了，所以这个数据就能在整个链路中共享了。

### 2.4.5 Fallback

![](images/diagram-11.png)

Fallback，即兜底返回。兜底返回的目的就是为了在远程调用失败的时候能拿到一个默认数据，让业务能继续往下推进。我们把这个数据叫做兜底数据

注意，此功能需要整合 Sentinel 才能实现。

因此需要先导入 Sentinel 依赖：

```yaml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

并在需要进行 Fallback 的服务的配置文件中开启配置：

```yaml
feign:
  sentinel:
    enabled: true
```

现在需要对 Feign 客户端 `ProductFeignClient` 配置 Fallback，那么需要先实现 `ProductFeignClient` 编写兜底返回逻辑，并将其交由 Spring 管理：

```java
@Component
public class ProductFeignClientFallback implements ProductFeignClient {
    @Override
    public Product getProductById(Long id) {
        System.out.println("Fallback...");
        Product product = new Product();
        product.setId(id);
        product.setPrice(new BigDecimal("0"));
        product.setProductName("未知商品");
        product.setNum(0);
        return product;
    }
}
```

之后回到对应的 Feign 客户端，配置 Fallback：

```python
@FeignClient(value = "service-product", fallback = ProductFeignClientFallback.class)
public interface ProductFeignClient {

    @GetMapping("/product/{id}")
    Product getProductById(@PathVariable("id") Long id);
}
```

![](images/image-5.png)

# 3.Sentinel

官方文档：[Sentinel](https://sentinelguard.io/zh-cn/docs/introduction.html)

维基：<https://github.com/alibaba/Sentinel/wiki>

## 3.1 功能介绍

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Spring Cloud Alibaba Sentinel 以流量为切入点，从流量控制、流量路由、熔断降级、系统自适应过载保护、热点流量防护等多个维度保护服务的稳定性。

Sentinel 具有以下特征:

* **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。

* **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。

* **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Apache Dubbo、gRPC、Quarkus 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。同时 Sentinel 提供 Java/Go/C++ 等多语言的原生实现。

* **完善的 SPI 扩展机制**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

![](images/diagram-12.png)

**架构：**

![](images/diagram-13.png)



定义资源：

* 主流框架自动适配（Web Servlet、Dubbo、Spring Cloud、gRPC、Spring WebFlux、Reactor），所有 Web 接口均为资源

* 编程式：SphU API

* 声明式：`@SentinelResource`

定义规则：

* 流量控制（FlowRule）

* 熔断降级（DegradeRule）

* 系统保护（SystemRule）

* 来源访问控制（AuthorityRule）

* 热点参数（ParamFlowRule）

![](images/diagram-14.png)

## 3.2 整合 Sentinel

> 启动Dashboard

前往 Sentinel GitHub Realease 页下载 Sentinel Dashboard，这里选择 1.8.8 版本，因此下载 `sentinel-dashboard-1.8.8.jar`。

在 `sentinel-dashboard-1.8.8.jar` 所在的目录运行以下命令，启动 Dashboard：

```yaml
java -jar sentinel-dashboard-1.8.8.jar
```

启动完成后，浏览器访问 `http://localhost:8080/`，默认用户与密码均为 `sentinel`。

> 服务整合 Sentinel

引入依赖：

```yaml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

配置文件中添加：

```yaml
spring:
  application:
    name: service-product
  cloud:
    sentinel:
      transport:
        # 控制台地址
        dashboard: localhost:8080
      # 立即加载服务  
      eager: true
```

配置完成后启动对应服务，再前往 Sentinel Dashboard 查看，能够看到对应服务信息。

![](images/image-3.png)

可以在一个方法上使用 `@SentinelResource` 注解，将其标记为一个「资源」，当方法被调用时，能够在 Dashboard 的「簇点链路」上找到对应的资源，之后在界面上完成对资源的流控、熔断、热点、授权等操作。例如：

```java
@SentinelResource(value = "createOrder")
@Override
public Order createOrder(Long productId, Long userId) {
    //使用Feign完成远程调用
    Product product = productFeignClient.getProductById(productId);
    Order order = new Order();
    order.setId(1L);
    //总金额
    order.setTotalAmount(product.getPrice().multiply(new BigDecimal(product.getNum())));
    order.setUserId(userId);
    order.setNickName("zhangsan");
    order.setAddress("尚硅谷");
    //远程查询商品列表
    order.setProductList(Arrays.asList(product));

    return order;
}
```

此时访问该Web接口：`http://localhost:8000/create?userId=1&productId=777`

![](images/image-1.png)

## 3.3 异常处理

![](images/diagram-15.png)

我们来梳理一下这个 **Sentinel BlockException 异常处理流程图**：

***

1️⃣ 整体概览

* Sentinel 在做限流、熔断、降级时，会抛出一个 **BlockException**。

* 不同的使用场景（Web接口、注解、Feign调用、硬编码）会走不同的处理流程。

* 最终都可以通过自定义或默认的 handler / fallback 来处理。

***

2️⃣ 左边：Web 接口

* **Web 接口**触发限流 → 抛出 **BlockException**

* 经过 `SentinelWebInterceptor` 拦截器

* 先走 **默认的 BlockExceptionHandler**

* 你也可以自定义 **BlockExceptionHandler** 替换默认逻辑，比如返回自己的错误 JSON 或友好提示页面

***

3️⃣ 中间：注解方式 @SentinelResource

* 通过 `@SentinelResource` 注解定义资源

* Sentinel 会通过 `SentinelResourceAspect` AOP 切面来拦截方法

* 可以自定义：

  * **blockHandler**：处理限流、降级等 Sentinel 抛出的 BlockException

  * **fallback**：处理方法执行本身抛出的异常（非限流）

* 如果没有显式指定，会进入 **兜底回调**（Sentinel 内部默认行为）

* 最后如果还是没处理，就交给 Spring Boot 全局异常处理机制

***

4️⃣ 右边：OpenFeign 调用

* 使用 OpenFeign + Sentinel 时

* 通过 `SentinelFeign.builder()` 创建 Feign Client

* 可以配置 **fallback** 来处理 BlockException 或接口调用失败

* 这样 Feign 的熔断/限流异常不会直接抛到外层

***

5️⃣ 最右边：SphU 硬编码

* 如果是用 `SphU.entry()` 这种硬编码方式进行资源保护

* 手动用 **try-catch** 捕获 **BlockException** 来做处理

***

6️⃣ 总结成一张表：

***

在`IDEA`中搜索`BlockExcetion`后按Ctrl + H 可以看见它的实现。针对不同的规则会有不同的细节异常

* `FlowException`: 流控异常

* `ParamFlowException`：热点参数异常

* `DegradeException`: 熔断降级异常

* `AuthorityException`: 权限控制异常

* `SystemBlockException`: 系统阻塞异常

![](images/image-4.png)

1. Web接口

当 Web 接口作为资源被流控时，默认情况下会在页面显示：

```yaml
Blocked by Sentinel (flow limiting)
```

如果需要自定义异常处理，可以实现 `BlockExceptionHandler` 接口，并将实现类交给 Spring 管理：

以Sentinel 限流异常处理器为例：

```java
//这段代码是一个自定义的 Sentinel 限流异常处理器，用于处理被 Sentinel 限流或降级的请求。
@Component
public class MyBlockExceptionHandler implements BlockExceptionHandler {
    private ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse response,
                       String resourceName, BlockException e) throws Exception {
        // 设置响应的内容类型为JSON格式，字符编码为UTF-8
        response.setContentType("application/json;charset=utf-8");
        // 获取响应的打印输出流，用于向客户端输出数据
        PrintWriter writer = response.getWriter();
        // 创建一个包含错误信息的R对象
        // 错误码为500，错误信息包含被限制的资源名称和异常类型
        R error = R.error(500, resourceName + "被Sentinel限制了，原因：" + e.getClass().getSimpleName());
        // 将R对象转换为JSON字符串
        String json = objectMapper.writeValueAsString(error);
        // 将JSON字符串写入响应
        writer.write(json);
        // 刷新输出流，确保数据被发送
        writer.flush();
        // 关闭输出流
        writer.close();
    }
}
```

```typescript
@Data
@NoArgsConstructor
@AllArgsConstructor
public class R {
    private Integer code;
    private String msg;
    private Object data;

    public static R ok(Object data){
        return new R(200,"操作成功",data);
    }

    public static R ok(String msg,Object data){
        return new R(200,msg,data);
    }

    public static R error(String msg){
        return new R(500,msg,null);
    }

    public static R error(Integer code, String msg){
        return new R(code,msg,null);
    }

}
```

以 `/create` 接口为例，当其被流控时，此时再访问`http://localhost:8000/create?userId=1&productId=777`多次，会出现：

![](images/image.png)

***

* `@SentinelResource`

当 `@SentinelResource` 注解标记的资源被流控时，默认返回 500 错误页。

![](images/image-2.png)

如果需要自定义异常处理，一般可以增加 `@SentinelResource` 注解的以下任意配置：

* `blockHandler`

* `fallback`

* `defaultFallback`

以 `blockHandler` 为例：

```typescript
@SentinelResource(value = "createOrder", blockHandler = "createOrderFallback")
public Order createOrder(Long productId, Long userId) {
    // --snip-
}
```

在当前类中创建名称为 `blockHandler` 值的方法，并且返回值类型、参数信息与 `@SentinelResource` 标记的方法一致（可以额外增加一个 `BlockException` 类型的参数）：

```sql
/**
 * 指定兜底回调
 */
public Order createOrderFallback(Long productId, Long userId, BlockException e) {
    Order order = new Order();
    order.setId(0L);
    order.setTotalAmount(new BigDecimal("0"));
    order.setUserId(userId);
    order.setNickname("未知用户");
    order.setAddress("异常信息: " + e.getClass());
    return order;
}
```

当资源没有被流控时，则调用真实的业务逻辑去返回真实的数据

当资源被流控时，被Sentinel限制了，则执行 `blockHandler` 指定的方法并返回兜底数据：

```json
{
  "id": 0,
  "totalAmount": 0,
  "userId": 1,
  "nickName": "未知用户",
  "address": "异常信息: class com.alibaba.csp.sentinel.slots.block.flow.FlowException",
  "productList": null
}
```

**总结：**`@SentinelResource` 一般标注在非controller层，要给哪些方法加上保护就加上这个注解，一旦违反规则之后，如果业务规定有兜底回调的数据就使用blockHandler去指定兜底回调，如果业务没规定兜底回调，那么也可以不用任何一种回调机制，直接让异常抛给全局由项目的SpringBoot全局异常处理器处理即可

* OpenFeign - 兜底回调

当 Feign 接口作为资源并被流控时，如果调用的 Feign 接口指定了 `fallback`，那么就会使用 Feign 接口的 `fallback` 进行异常处理，否则由 SpringBoot 进行全局异常处理。

* Sphu 硬编码 ----了解即可

## 3.4 流控规则

***

流控，即流量控制（FlowRule），用于限制多余请求，从而保护系统资源不被耗尽。

![](images/diagram-16.png)



> 阈值类型

![](images/image-7.png)

Sentinel 的限流（FlowRule）主要由 **限流阈值 + 统计维度 + 控制效果** 组成。

1. Sentinel 的流控阈值规则有两种：

   1. QPS（每秒请求数）：Queries Per Second，用于限制资源每秒的请求次数，防止突发流量，应用于高频短时接口（如 API 网关）。当每秒的请求数超过设定的阈值时，就会触发流控。比如上图设置的 QPS = 5，就表示每秒最多允许 5 个请求。

   2. 并发线程数（Concurrent Threads）：用于限制同时处理该资源的线程数（即并发数），保护系统资源（线程池），应用于耗时操作（如数据库查询）。当处理该资源的线程数超过阈值时，就会触发流控。比如设置并发线程数为 5，表示最多允许 5 个线程同时处理该资源。

   Sentinel 在创建 FlowRule 时通过 `grade` 字段区分这两种规则：

   * `RuleConstant.FLOW_GRADE_QPS`（按 QPS 限流）

   * `RuleConstant.FLOW_GRADE_THREAD`（按线程数限流）

2. 流控维度（来源粒度）：

在来源粒度上，Sentinel 可以选择 **default（全部来源）** 或 **按调用来源限流**。
&#x20;默认模式下，所有调用方共享同一个阈值，不区分来源；而按调用来源限流则可以针对不同的调用方设置不同的阈值，例如不同微服务访问同一个接口时，可以根据来源区分限流策略。

* 流控效果（控制行为）

Sentinel 的流控效果主要有三种。



当勾选「是否集群」时，有两种集群阈值模式可供选择：

1. 单机均摊：将设置的「均摊阈值」均摊到每个节点。以上图为例，假设集群有 3 个节点，那么每个节点的阈值都是 5；

2. 总体阈值：整个集群共享设置的「均摊阈值」。假设集群有 3 个节点，这 3 个节点的的总阈值只有 5，比如按 `2-2-1` 的形式将阈值均摊到每个节点。



> 集群阈值模式

1️⃣ 单机均摊

* **意思**：给整个集群设定一个总的阈值，Sentinel 会自动把这个阈值平均分配到每台实例上。

* **效果**：每台机器各自按“均摊后的阈值”限流，不需要单独计算全局请求数。

* **场景**：各实例流量比较平均，不想搭建 Token Server 时最简单的方案。

例子：
&#x20;设定阈值 100 QPS，集群 5 台实例，Sentinel 自动把每台的阈值分成 20 QPS，单台超过 20 就限流。

***

2️⃣ 总体阈值

* **意思**：整个集群的请求总数共同遵守这个阈值，由 **Token Server** 统一分配令牌。

* **效果**：全局只有一个总限流值，所有实例按这个总值一起限流，可以做到真正的全局限流。

* **场景**：集群各节点流量差异大，或者希望整体严格控制集群总量，需部署 Token Server。

例子：
&#x20;设定阈值 100 QPS，集群 5 台实例，总体阈值模式下 5 台合计只能跑 100 QPS，多余的请求会被统一限流。

***

✅ **总结对比：**

***

💡 另外看到“失败退化”这个选项：如果 Token Server 不可用，勾选后 Sentinel 会自动退化为单机限流模式，防止 Token Server 挂掉导致整个集群不可用。



> 流控模式

![](images/image-22.png)

配置流控规则时，可以点击下方的「高级选项」，在这里可以配置「流控模式」，共有三种可选项：

1️⃣ 直接模式（默认）

* **意思**：对这个资源本身进行限流，和其它资源无关。

* **效果**：只要当前资源的 QPS / 并发线程数超过设定阈值，就直接触发限流。

* **场景**：普通接口最常用的限流方式。

***

2️⃣ 关联模式

* **意思**：当 **另外一个资源** 达到阈值时，限流当前资源。

* **效果**：资源 A 自己没超阈值，但关联的资源 B 超过了阈值，A 也会被限流。

* **场景**：比如一个支付接口（资源 A）关联下单接口（资源 B），当下单接口流量过高时连带支付接口一起限流，避免关键接口被压垮。

* **在配置时**：需要额外指定“关联资源”的名字。

***

3️⃣ 链路模式

* **意思**：按调用链路来限流，只对指定入口资源生效。

* **效果**：同一个资源从不同调用路径过来时，可以分别设限。

* **场景**：例如某个方法 `queryUser` 被不同业务调用，A 业务和 B 业务的访问量不同，你只想对 A 业务调用这个方法做限流，不影响 B 业务。

* **在配置时**：需要额外指定入口资源名（哪个入口链路下生效）。

✅ **总结对比**：

调用关系包括调用方、被调用方；一个方法又可能会调用其他方法，形成一个调用链路的层次关系；有了调用链路的统计信息，可以衍生出多种流量控制手段。

![](images/diagram-17.png)



> 流控效果

![](images/image-25.png)

“**流控效果**”下面有三个选项：**快速失败、Warm Up、排队等待**，分别代表不同的限流处理策略：

***

1️⃣ 快速失败（默认）

* **意思**：当请求数超过阈值时，直接拒绝多余的请求并抛出 `BlockException`（HTTP 接口会直接返回错误）。

* **特点**：实现简单、反应快，但有抛弃请求的风险。

* **适用场景**：普通接口、可接受直接拒绝的场景。

![](images/diagram-18.png)

***

2️⃣ Warm Up（预热模式）

* **意思**：先以较小的流量阈值放行请求，随着时间推移逐渐升到设定的最大阈值（类似“慢启动”）。

* **特点**：能防止冷启动时突然放入大量流量压垮系统。

* **适用场景**：秒杀、促销等瞬时流量激增场景，或系统初始化较慢的服务。

> 例子：设定阈值 100 QPS、预热时间 10 秒，刚启动时可能只允许 20 QPS，逐渐升到 100 QPS。

![](images/diagram-19.png)

***

3️⃣ 排队等待（匀速排队）

* **意思**：当请求数超过阈值时，不直接拒绝，而是把请求排队，按固定速率（漏桶算法）处理。

* **特点**：让请求以恒定速率进入系统，实现流量平滑，避免瞬间高峰。

* **适用场景**：希望请求都能处理，只要能稍微等待；对延迟敏感度低但要求稳定的接口。

> 例子：设定阈值 50 QPS，多出来的请求会排队等待，不会立刻被拒绝。

![](images/diagram-20.png)

***

✅ **总结对比：**

***

## 3.5 熔断规则

熔断规则，即DegradeRule。

使用熔断规则可以配置熔断降级，用于：

* 切断调用

* 快速返回不积压

* 避免雪崩效应

**支架：**

![](images/image-23.png)

**最佳实践**：熔断降级作为保护自身的手段，通常在客户端（调用端）进行配置。

熔断降级里的核心组件是「静止」，其工作原理如下：

![](images/diagram-21.png)

Sentinel提供了清晰的熔断策略：

1. 慢调用比例

2. 异常比例

3. 异常数

> 慢调用比例

![](images/image-24.png)

在5000ms内，有80%（0.8的比例阈值）的请求的最大响应时间超过1000ms，则进行30s的熔断。

如果5000ms内，请求数不超过5，即使达到熔断规则，也不进行熔断。

> 异常比例

在远程调用的目标接口里添加`int i = 1 / 0;`模拟远程调用异常。

此时尚未配置任何熔断规则，然后远程调用存在异常的接口，此时会触发使用OpenFeign配置的兜底回调。

那么，不配置任何熔断规则都可以触发兜底回调，而配置熔断规则也可以触发兜底回调，那不是配不配置熔断规则都可以吗？

![](images/diagram-22.png)

所以无熔断规则和有熔断规则的相同点都是远程一旦有问题都会执行兜底回调，但是有熔断规则的好处就是远程出问题，会有一段时间我就不理远程了，不给远程发请求了，这样就节约了远程调用时间，避免无意义的远程调用请求，节约了很多的资源

![](images/image-27.png)

在5000ms内，有80%（0.8的比例阈值）的请求产生了异常，则进行30s的熔断。

> 异常数

![](images/image-21.png)

「异常数」的熔断策略与「异常比例」很相似，只不过「异常数」是直接统计异常数，即使统计时长内产生百万个请求，但只要有 10 个请求出现了异常，就会触发熔断。

## 3.6 热点规则（热点参数限流）

***

### 3.6.1 概述

所谓热点，即经常访问的数据。很多时候希望统计某个热点数据中访问频率最高的前K个数据，并对其访问进行限制。比如：

* 商品ID为参数，统计时段内最常购买的商品ID并进行限制

* user ID 为参数，针对定时限制访问的用户 ID 进行一段时间

热点参数限流会统计参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。

热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

![](images/image-26.png)

Sentinel利用LRU策略统计最近最常访问的热点参数，结合令牌桶算法来进行参数级别的流控。热点参数限制流支持集群模式。

> 以秒杀为例

现有需求如下：

* 每个用户秒杀QPS不得超过1（秒杀下单时，userId等级）

* 6号用户是vvip，不限制QPS（例外情况）

* 666号商品为下架商品，不允许访问

在 Sentinel GitHub Wiki 中指出：

* 目前Sentinel自带的适配器只有Dubbo方法埋点带了热点参数，改装其他模块（如Web）默认不支持生成热点规则，可以通过自定义埋点方式指定新的资源名并创建的参数。注意自定义埋点的资源名不要和装备模块的资源名重复，否则会导致重复统计。

```java
@GetMapping("/seckill")
@SentinelResource(value = "seckill-order",fallback = "seckillFallback")
public Order seckill(@RequestParam(value = "userId",required = false) Long userId,
                     @RequestParam(value = "productId",defaultValue = "1000") Long productId){
    Order order = orderService.createOrder(productId, userId);
    order.setId(Long.MAX_VALUE);
    return order;
}

public Order seckillFallback(Long userId,Long productId, Throwable exception){
    System.out.println("seckillFallback....");
    Order order = new Order();
    order.setId(productId);
    order.setUserId(userId);
    order.setAddress("异常信息："+exception.getClass());
    return order;
}
```

### 3.6.2 热点参数规则

热点参数规则（`ParamFlowRule`）相似控制规则（`FlowRule`）：

对`seckill-order`资源进行如下热点规则配置：

这表示：访问`seckill-order`资源时，第一个参数（参数索引0）在1秒的统计窗口时长下，其阈值为1，那么QPS = 1。

需要注意：携带此参数，则参与流控；不携带则不流控。

```typescript
@GetMapping("/seckill")
@SentinelResource(value = "seckill-order", fallback = "seckillFallback")
public Order seckill(@RequestParam(value = "userId", defaultValue = "888") Long userId,
                     @RequestParam(value = "productId", defaultValue = "1000") Long productId) {
    // --snip--
}
```

其中代码中，`userId`的默认值为`888`，以`http://localhost:8000/seckill?productId=777`的形式进行访问时，`userId`的值为`888`，此时又建立了`userId`，重新触发流控。

```typescript
@GetMapping("/seckill")
@SentinelResource(value = "seckill-order", fallback = "seckillFallback")
public Order seckill(@RequestParam(value = "userId", required = false) Long userId,
                     @RequestParam(value = "productId", defaultValue = "1000") Long productId) {
    // --snip--
}
```

该代码中，`userId`可以不传递，当以`http://localhost:8000/seckill?productId=777`这种形式进行访问时，`userId`为`null`，不形成`userId`，不会触发流控。

经过上述配置，已经完成「每个用户杀QPS不得超过1」的需求，但「6号用户」是个例外：

![](images/image-20.png)

访问`seckill-order`资源时，第一个参数（参数索引0）的类型是`long`，当其值为`6`时，限流阈值为`1000000000`，变相不限制「6号用户」的QPS。

现在还有最后一个需求「`666`号商品是下架商品，不允许访问」，这实际上相当于：对`666`号商品进行流控（限流阈值为0，不允许访问），对其他商品不进行流控（或阈值非常大）。

![](images/image-16.png)

访问`seckill-order`资源时，第二个参数（参数索引 1）在 1 秒的统计窗口时长下，其阈值为 1000000，这是一个无法达到的值，不进行限流。但有一个例外：当其值为 666 时，限流阈值为 0，则不允许访问。

### 3.6.3 补充：fallback和blockHandler兜底回调

***

1️⃣ `blockHandler` —— **流控/熔断违规时的兜底处理**

* **触发场景**：请求被 Sentinel 的**流控**、**熔断降级**、**系统保护**等规则拦截时（即**规则触发**时）。

* **作用**：自定义当请求被 **限流 / 熔断** 时该怎么返回，而不是直接抛异常。

* **方法签名**：

* `@SentinelResource(
      value = "testResource",
      blockHandler = "handleBlock"  // 指定被限流时调用的方法
  )public String testResource() {return "正常业务逻辑";
  }
  `
  `// 注意 blockHandler 方法必须在同一个类或指定的类里
  public String handleBlock(BlockException e) {return "被限流/熔断了：" + e.getClass().getSimpleName();
  }`

* **总结**：`blockHandler` 专门处理 **Sentinel 规则触发**的情况。

***

2️⃣ `fallback` —— **业务异常时的兜底处理**

* **触发场景**：当业务方法本身 **抛出异常**（非流控异常）时。

* **作用**：在出现运行时异常、空指针、超时等 **业务异常**时，提供统一的降级返回，不影响主业务。

* **方法签名**：

* `@SentinelResource(
      value = "testResource",
      fallback = "handleFallback"  // 指定业务异常时调用的方法
  )public String testResource() {// 模拟业务异常
      int a = 1/0;return "正常业务逻辑";
  }
  `
  `public String handleFallback(Throwable ex) {return "业务异常降级处理：" + ex.getMessage();
  }`

* **总结**：`fallback` 处理 **业务异常**。

***

3️⃣ 同时配置 `blockHandler` + `fallback`

两者可以一起用，优先级：

* **规则触发（限流/熔断） → `blockHandler`**

* **业务抛异常 → `fallback`**

### 3.6.4 授权规则

![](images/image-17.png)

解释下这张图里 **Sentinel 的“授权规则”**：

***

1️⃣ 资源名

`/seckill`

> 这是你要保护的资源，一般是 URL、接口名、方法名。这里表示“秒杀接口”。

***

2️⃣ 流控应用

`order, product, pay`

> 这是**调用这个资源的来源应用**。
> &#x20;Sentinel 支持按**调用方**做限流或授权。

* 写上 `order, product, pay` 表示这几个应用在调用 `/seckill` 时，要按照下面的授权类型判断。

* 多个应用用逗号隔开。

***

3️⃣ 授权类型

* **白名单**：只允许列表里的应用访问，其它应用都不允许。

* **黑名单**：拒绝列表里的应用访问，其它应用都可以。

这里选的是 **白名单**：

> **只有 `order`、`product`、`pay` 这三个应用可以访问 `/seckill` 资源**，其它来源的调用都会被拦截。

🔑 总结

这张图的规则意思是：

> “对资源 `/seckill` 建立一个**授权规则**，只有 `order`、`product`、`pay` 这三个应用在调用 `/seckill` 时可以通过（白名单），其它应用访问都会被 Sentinel 拦截。”



### 3.6.5系统规则

### 3.6.6授权规则

这两个规则没什么用，了解即可

![](images/image-15.png)

# 4.Gateway

**官网**：<https://spring.io/projects/spring-cloud-gateway>

在`pom.xml`文件中添加依赖

```xml
<!--   网关依赖     -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!--   nacos注册中心依赖     -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!--   负载均衡依赖     -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

![](images/diagram-23.png)

## 4.1 路由

需求量：

1. 客户端发送`/api/order/**`转向`service-order`

2. 客户端发送`/api/product/**`转向`service-product`

3. 以上转发有负载均衡效果

![](images/diagram-24.png)

配置路由规则时，可直接在配置文件中完成：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: bing-route
          uri: https://cn.bing.com
          predicates:
            - Path=/**
          order: 10
          # id 全局唯一
        - id: order-route
          # 指定服务名称
          uri: lb://service-order
          # 指定断言规则，即路由匹配规则
          predicates:
            - Path=/api/order/**
          order: 1
        - id: product-route
          uri: lb://service-product
          predicates:
            - Path=/api/product/**
          order: 2
```

网关路由的工作原理如下：

![](images/diagram-25.png)

## 4.2 断言

官方文档：[路由谓词工厂](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/request-predicates-factories.html)

断言的两种书写方式：长断言，短断言

```yaml
spring:
  cloud:
    gateway:
      routes:
          # id 全局唯一
        - id: order-route
          # 指定服务名称
          uri: lb://service-order
          # 指定断言规则，即路由匹配规则
          # Fully Expanded Arguments
          predicates:
            - name: Path
              args:
                patterns: /api/order/**
                matchTrailingSlash: true
        - id: product-route
          uri: lb://service-product
          # Shortcut Configuration
          predicates:
            - Path=/api/product/**
```

在 Spring Cloud Gateway 的实现中，断言的实现都是`RoutePredicateFactory`接口的实现。

因此除了直接查看官方文档外确定有哪些断言形式外，还可以通过查看`RoutePredicateFactory`的实现：

* `HeaderRoutePredicateFactory`

* `PathRoutePredicateFactory`

* `ReadBodyRoutePredicateFactory`

* `BeforeRoutePredicateFactory`

* ...

![](images/image-18.png)

断言的名称可以通过去掉实现类名后来`RoutePredicateFactory`确定，比如`HeaderRoutePredicateFactory`对应称为`Header`的断言。

以`Query`为例：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: bing-route
          uri: https://cn.bing.com
          predicates:
            - name: Path
              args:
                patterns: /search
            - name: Query
              args:
                param: q
                regexp: haha
```

这表示：访问网关的`/search`地址，并且使用了名为`q`的请求参数，且满足`haha`，将会将请求转到`https://cn.bing.com`。

尽管网关内置了许多断言规则，但仍难以满足千变万化的需求。

在规则的基础上，再指定一个`Vip`称为断言的规则，要求存在称为`user`的请求参数，并且成立`mofan`时才将请求跳转到`https://cn.bing.com`：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: bing-route
          uri: https://cn.bing.com
          predicates:
            - name: Path
              args:
                patterns: /search
            - name: Query
              args:
                param: q
                regexp: haha
            - Vip=user,yjw
```

自定义`AbstractRoutePredicateFactory`实现类`VipRoutePredicateFactory`：

```java
@Component
public class VipRoutePredicateFactory extends AbstractRoutePredicateFactory<VipRoutePredicateFactory.Config> {


    public VipRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("param", "value");
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return (GatewayPredicate) serverWebExchange -> {
            // localhost/search?q=haha&user=yjw
            ServerHttpRequest request = serverWebExchange.getRequest();
            String first = request.getQueryParams().getFirst(config.param);
            return StringUtils.hasText(first) && first.equals(config.value);
        };
    }

    @Validated
    @Getter
    @Setter
    public static class Config {
        @NotEmpty
        private String param;
        @NotEmpty
        private String value;
    }
}
```

然后访问`http://localhost/search?q=haha&user=`yjw时，会跳转到 Bing 搜索`haha`。

## 4.3 过滤器

**官方文档**：[GatewayFilter Factory](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/gatewayfilter-factories.html)

![](images/diagram-26.png)

前面在网关中配置了将`/api/order/`底层的请求转向`service-order`服务，并要求在`service-order`服务中也存在的`/api/order/`请求路径，比如`/api/order/readDb`。如果该服务中并不存在`/api/order/`底层的请求，比如只有`/readDb`，那么在进行`/api/order/readDb`访问时就会出现404错误。

为了解决这个问题，可以在`service-order`服务对应的Controller上添加`@RequestMapping("/api/order")`注解，但这不是最佳方案，如果能直接在网关方面解决这个问题就好了，就像把`/api/order/readDb`重写为一样`/readDb`。就是网关判断完基准路径`/api/order/readDb`后把前面的`/api/order/`给删掉，只让`/readDb`传下去，让目的地最终感知到的请求路径只有后面的`/readDb`。这就是路径重写

网关中内置了许多过滤器，其中有一个常用的过滤器名为：`RewritePath`，即路径重写。

![](images/diagram-27.png)

```yaml
spring:
  cloud:
    gateway:
      routes:
          # id 全局唯一
        - id: order-route
          # 指定服务名称
          uri: lb://service-order
          # 指定断言规则，即路由匹配规则
          # Fully Expanded Arguments
          predicates:
            - name: Path
              args:
                patterns: /api/order/**
                matchTrailingSlash: true
          filters:
            # 类似把 /api/order/a/bc 重写为 /a/bc，移除路径前的 /api/order/
            - RewritePath=/api/order/?(?<segment>.*), /$\{segment}
          order: 1
        - id: product-route
          uri: lb://service-product
          # Shortcut Configuration
          predicates:
            - Path=/api/product/**
          filters:
            - RewritePath=/api/product/?(?<segment>.*), /$\{segment}
          order: 2
```

> 默认过滤器

如果需要为所有路由都添加同一个过滤器，则可以使用默认过滤器，比如：

```java
spring:
  cloud:
    gateway:
      default-filters:
        # 为所有路由添加响应头过滤器
        - AddResponseHeader=X-Response-Abc, 123
```

> 全局过滤器

除了默认过滤器之外，全局过滤器也中断所有匹配的路由添加一个过滤器，全局过滤器的配置修改配置文件。

实现`GlobalFilter`接口，把实现类交由Spring管理，即可实现全局过滤器。

还可以实现`Ordered`接口，调整多个全局过滤器的执行顺序。

```java
@Component
@Slf4j
public class RtGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        String uri = request.getURI().toString();
        long start = System.currentTimeMillis();
        log.info("请求【{}】开始：时间：{}",uri,start);
        //========================以上是前置逻辑=========================
        Mono<Void> filter = chain.filter(exchange)
                .doFinally((result)->{
                    //=======================以下是后置逻辑=========================
                    long end = System.currentTimeMillis();
                    log.info("请求【{}】结束：时间：{}，耗时：{}ms",uri,end,end-start);
                }); //放行   10s
        return filter;
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

> 自定义过滤器工厂

尽管网关内置了许多过滤器，但无法满足需求的情况，此时就需要自定义过滤器工厂。

与自定义断言类似，自定义过滤器工厂的类名也有限制，要求以`GatewayFilterFactory`结尾，而配置文件中配置的名称就是类名引用。

例如需要在配置文件中定义名为`OnceToken`的过滤器，那么需要添加`OnceTokenGatewayFilterFactory`：

```java
@Component
public class OnceTokenGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {
    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                //每次响应之前，添加一个一次性令牌，支持 uuid，jwt等各种格式
                return chain.filter(exchange).then(Mono.fromRunnable(()->{
                    ServerHttpResponse response = exchange.getResponse();
                    HttpHeaders headers = response.getHeaders();
                    String value = config.getValue();
                    if ("uuid".equalsIgnoreCase(value)){
                        value = UUID.randomUUID().toString();
                    }

                    if ("jwt".equalsIgnoreCase(value)){
                        value = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ";
                    }

                    headers.add(config.getName(),value);
                }));
            }
        };
    }
}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://service-order
          filters:
            # 自定义过滤器
            - OnceToken=X-Response-Token, uuid
```

此时再访问`localhost/api/order/readDb`，就会出现`readDb success...`

## 4.4 全局跨域

***

如果需要跨域配置，可以在Controller的类上添加`@CrossOrigin`注解。

如果有很多控制器，逐一添加注解太麻烦，可以在项目的配置类中添加`CorsFilter`类型的Bean。

上述方法只适用于纯化服务，那如果在微服务中呢？

借由 Gateway 的功能，可以在配置文件中轻松配置完成微服务的跨域：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origin-patterns: '*'
            allowed-headers: '*'
            allowedMethods: '*'
```

随后在请求的响应标头中会增加一些允许跨域的信息。

**微服务之间的调用是不经过网关的**

![](images/diagram-28.png)

**补：各种过滤器**

# 5.Seata

***

官方文档：[Seata](https://seata.apache.org/zh-cn/)&#x20;

[Seata](https://seata.apache.org/zh-cn/) 是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

在微服务项目中，一个操作往往会涉及多个不同的服务，每个服务又会连接不同的数据库：

Seata提供了在分布式系统下，保证多个数据库一起提交回滚，从而达到数据一致性状态的一站式解决方案

![](images/diagram-29.png)

## 5.1 环境准备

现有如下交易流程：

![](images/diagram-30.png)

发起采购流程后，需要扣库存、生成订单、从账户中扣除指定金额，任一流程发生异常时，整个流程应当回滚。

原理：

![](images/diagram-31.png)

* TC：Transaction Coordinator，即事务协调者。维护全局和分支事务的状态，驱动全局事务提交或回滚；

* TM：Transaction Manager，即事务管理器。定义全局事务的范围，开始全局事务、提交或回滚全局事务；

* RM：Resource Manager，即资源管理器。管理分支事务处理的资源，与 TC 交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

[下载](https://seata.apache.org/zh-cn/download/seata-server)并解压 Seata 后，进入 `seata\seata-server\bin`目录，然后使用 `seata-server.bat` 命令启动 Seata。

下载的 Seata 版本保证与 pom 文件中引入的 `spring-cloud-alibaba-dependencies` 依赖中的 Seata 版本一致。

在需要使用分布式事务的模块中添加依赖：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

文件导入:

![](images/image-19.png)

将这四个微服务导入到services层中，并在pom.xml中导入**依赖**

```xml
<modules>
    <module>service-product</module>
    <module>service-order</module>
    <module>seata-account</module>
    <module>seata-business</module>
    <module>seata-order</module>
    <module>seata-storage</module>
</modules>
```

配置

每个微服务创建 `file.conf`文件，完整内容如下；

```bash
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

transport {
  # tcp, unix-domain-socket
  type = "TCP"
  #NIO, NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  # the tm client batch send request enable
  enableTmClientBatchSendRequest = false
  # the rm client batch send request enable
  enableRmClientBatchSendRequest = true
   # the rm client rpc request timeout
  rpcRmRequestTimeout = 2000
  # the tm client rpc request timeout
  rpcTmRequestTimeout = 30000
  # the rm client rpc request timeout
  rpcRmRequestTimeout = 15000
  #thread factory for netty
  threadFactory {
    bossThreadPrefix = "NettyBoss"
    workerThreadPrefix = "NettyServerNIOWorker"
    serverExecutorThread-prefix = "NettyServerBizHandler"
    shareBossWorker = false
    clientSelectorThreadPrefix = "NettyClientSelector"
    clientSelectorThreadSize = 1
    clientWorkerThreadPrefix = "NettyClientWorkerThread"
    # netty boss thread size
    bossThreadSize = 1
    #auto default pin or 8
    workerThreadSize = "default"
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}
service {
  #transaction service group mapping
  vgroupMapping.default_tx_group = "default"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #degrade, current not support
  enableDegrade = false
  #disable seata
  disableGlobalTransaction = false
}

client {
  rm {
    asyncCommitBufferLimit = 10000
    lock {
      retryInterval = 10
      retryTimes = 30
      retryPolicyBranchRollbackOnConflict = true
    }
    reportRetryCount = 5
    tableMetaCheckEnable = false
    tableMetaCheckerInterval = 60000
    reportSuccessEnable = false
    sagaBranchRegisterEnable = false
    sagaJsonParser = "fastjson"
    sagaRetryPersistModeUpdate = false
    sagaCompensatePersistModeUpdate = false
    tccActionInterceptorOrder = -2147482648 #Ordered.HIGHEST_PRECEDENCE + 1000
    sqlParserType = "druid"
    branchExecutionTimeoutXA = 60000
    connectionTwoPhaseHoldTimeoutXA = 10000
  }
  tm {
    commitRetryCount = 5
    rollbackRetryCount = 5
    defaultGlobalTransactionTimeout = 60000
    degradeCheck = false
    degradeCheckPeriod = 2000
    degradeCheckAllowTimes = 10
    interceptorOrder = -2147482648 #Ordered.HIGHEST_PRECEDENCE + 1000
  }
  undo {
    dataValidation = true
    onlyCareUpdateColumns = true
    logSerialization = "jackson"
    logTable = "undo_log"
    compress {
      enable = true
      # allow zip, gzip, deflater, lz4, bzip2, zstd default is zip
      type = zip
      # if rollback info size > threshold, then will be compress
      # allow k m g t
      threshold = 64k
    }
  }
  loadBalance {
      type = "XID"
      virtualNodes = 10
  }
}
log {
  exceptionRate = 100
}
tcc {
  fence {
    # tcc fence log table name
    logTableName = tcc_fence_log
    # tcc fence log clean period
    cleanPeriod = 1h
  }
}
```

【微服务只需要复制 `service 块`配置即可】

即在需要使用 Seata 的模块中添加 Seata 的**配置**文件 `file.conf` ：

```bash
service {
  #transaction service group mapping
  vgroupMapping.default_tx_group = "default"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #degrade, current not support
  enableDegrade = false
  #disable seata
  disableGlobalTransaction = false
}
```

SQL

> 启动一个数据库，然后运行如下sql文件

```sql
CREATE DATABASE IF NOT EXISTS `storage_db`;
USE  `storage_db`;
DROP TABLE IF EXISTS `storage_tbl`;
CREATE TABLE `storage_tbl` (
                               `id` int(11) NOT NULL AUTO_INCREMENT,
                               `commodity_code` varchar(255) DEFAULT NULL,
                               `count` int(11) DEFAULT 0,
                               PRIMARY KEY (`id`),
                               UNIQUE KEY (`commodity_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
INSERT INTO storage_tbl (commodity_code, count) VALUES ('P0001', 100);
INSERT INTO storage_tbl (commodity_code, count) VALUES ('B1234', 10);

-- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
                            `id` bigint(20) NOT NULL AUTO_INCREMENT,
                            `branch_id` bigint(20) NOT NULL,
                            `xid` varchar(100) NOT NULL,
                            `context` varchar(128) NOT NULL,
                            `rollback_info` longblob NOT NULL,
                            `log_status` int(11) NOT NULL,
                            `log_created` datetime NOT NULL,
                            `log_modified` datetime NOT NULL,
                            `ext` varchar(100) DEFAULT NULL,
                            PRIMARY KEY (`id`),
                            UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE DATABASE IF NOT EXISTS `order_db`;
USE  `order_db`;
DROP TABLE IF EXISTS `order_tbl`;
CREATE TABLE `order_tbl` (
                             `id` int(11) NOT NULL AUTO_INCREMENT,
                             `user_id` varchar(255) DEFAULT NULL,
                             `commodity_code` varchar(255) DEFAULT NULL,
                             `count` int(11) DEFAULT 0,
                             `money` int(11) DEFAULT 0,
                             PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
-- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
                            `id` bigint(20) NOT NULL AUTO_INCREMENT,
                            `branch_id` bigint(20) NOT NULL,
                            `xid` varchar(100) NOT NULL,
                            `context` varchar(128) NOT NULL,
                            `rollback_info` longblob NOT NULL,
                            `log_status` int(11) NOT NULL,
                            `log_created` datetime NOT NULL,
                            `log_modified` datetime NOT NULL,
                            `ext` varchar(100) DEFAULT NULL,
                            PRIMARY KEY (`id`),
                            UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE DATABASE IF NOT EXISTS `account_db`;
USE  `account_db`;
DROP TABLE IF EXISTS `account_tbl`;
CREATE TABLE `account_tbl` (
                               `id` int(11) NOT NULL AUTO_INCREMENT,
                               `user_id` varchar(255) DEFAULT NULL,
                               `money` int(11) DEFAULT 0,
                               PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
INSERT INTO account_tbl (user_id, money) VALUES ('1', 10000);
-- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
                            `id` bigint(20) NOT NULL AUTO_INCREMENT,
                            `branch_id` bigint(20) NOT NULL,
                            `xid` varchar(100) NOT NULL,
                            `context` varchar(128) NOT NULL,
                            `rollback_info` longblob NOT NULL,
                            `log_status` int(11) NOT NULL,
                            `log_created` datetime NOT NULL,
                            `log_modified` datetime NOT NULL,
                            `ext` varchar(100) DEFAULT NULL,
                            PRIMARY KEY (`id`),
                            UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```



![](images/diagram-32.png)

&#x20;以上流程图表示Seata 在 AT 模式下的**二阶段提交协议**（2PC）是怎么工作的。

***

* 图的总体含义

图中展示了 Seata 在 **AT 模式**下的两阶段提交流程：

***

* 主要角色（图中块）

  1. **Business @GlobalTransactional**（绿色块）
     &#x20;发起全局事务的业务方法，用注解 `@GlobalTransactional` 开启。

  2. **TC（seata-server 事务协调器）**（蓝色长条）
     &#x20;管理全局事务状态，协调各个分支提交或回滚。

  3. **Storage / Account / Order 服务**（紫/绿/紫块）
     &#x20;这是三个参与全局事务的分支服务：库存、账户、订单。每个都有自己的本地数据库（storage\_db、account\_db、order\_db）。

  4. **Undo Log（回滚日志）**（黄色块）
     &#x20;第一阶段写入，用于第二阶段回滚时恢复数据。

***

* 第一阶段（左边竖长框 “本地事务”）

以 **库存服务**（storage）为例，执行步骤：

***

* 第二阶段（右下角绿色/红色块）

根据 TC 决策分支有两种情况：

（1）分支提交（绿色块）

（2）分支回滚（红色块）

这样保证所有分支事务回到一致的状态。

***

* 图里的“锁”部分

  * `order_tbl 1号记录全局锁`、`account_tbl 1号记录全局锁`、`storage_tbl 1号记录全局锁`
    &#x20;表示分支事务在第一阶段锁定这些资源，避免其它全局事务并发修改同一行。

***

1. 总结（图想表达的重点）

   1. **第一阶段**：本地执行业务逻辑 + 记录 Undo Log + 注册分支事务（但还没最终提交/清理 Undo Log）

   2. **第二阶段**：

      1. 如果全局提交：TC 通知各分支“提交”，分支异步删除 Undo Log

      2. 如果全局回滚：TC 通知各分支“回滚”，分支用 Undo Log 恢复数据&#x20;

这样既能保证分布式事务一致性，又减少资源锁定时间。















































































































