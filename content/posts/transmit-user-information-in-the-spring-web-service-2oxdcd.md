---
title: 在 Spring Web 服务中传递用户信息
slug: transmit-user-information-in-the-spring-web-service-2oxdcd
url: /post/transmit-user-information-in-the-spring-web-service-2oxdcd.html
date: '2025-01-11 08:26:30+08:00'
lastmod: '2025-01-11 08:51:55+08:00'
tags:
  - SpringBoot
categories:
  - 快乐开发一点通
keywords: SpringBoot
toc: true
isCJKLanguage: true
---



面向 C 端的服务结构大致如下：

<iframe src="/widgets/widget-excalidraw/" data-src="/widgets/widget-excalidraw/" data-subtype="widget" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

​![image](https://raw.githubusercontent.com/luckypeak-xyz/images/main/image-20250111111501-w5l763d.png)​

用户信息以 token 的形式从 客户端 经过 网关（或者 BFF）完成鉴权后，以 account_id 的形式（放在 header 中）在后端服务内部及后端服务之间传递。

## 实验设定

通过一个小实验来实践在 Spring Web 服务中传递用户信息：

1. 访问 服务A 的 info 接口

    1. 假设鉴权已完成，header 中有 accountId=1 的用户信息
2. 服务A 打印用户信息（accountId=1，模拟对用户信息的使用）
3. 服务A 通过 feign 调用B 的 info 接口

    1. 同样将 accountId=1 用户信息放在 header 中传递
4. 服务B 打印用户信息
5. 服务B 并发调用 服务C 和 服务D 的 info 接口
6. 服务C 和 服务D 打印 用户信息

## 基于 ThreadLocal 在代码内传递用户信息

在 服务A 中新建 User 类用户存储用户信息：

```java
@Data
public class User {

    private String accountId;

}
```

新建 UserContext 类，提供设置、访问和删除 ThreadLocal 中用户信息的途经：

```java
public class UserContext {

    private static final ThreadLocal<User> USER_CONTEXT_HOLDER = new ThreadLocal<>();


    public static void set(User user) {
        USER_CONTEXT_HOLDER.set(user);
    }

    public static User get() {
        return USER_CONTEXT_HOLDER.get();
    }

    public static void remove() {
        USER_CONTEXT_HOLDER.remove();
    }

}
```

实现 `HandlerInterceptor`​ 处理用户上下文信息：

```java
public class UserContextInterceptor implements HandlerInterceptor {

    public static final String ACCOUNT_ID = "X-Account-Id";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String accountId = request.getHeader(ACCOUNT_ID);

        UserContext.set(new User().setAccountId(accountId));

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        UserContext.remove();
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```

注册 UserContextHandler：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Bean
    public UserContextInterceptor userContextInterceptor() {
        return new UserContextInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(userContextInterceptor());
    }
}
```

新增 info 接口：

```java
@Slf4j
@RestController
@RequestMapping("/info")
public class InfoController {

    @GetMapping
    public String info() {
        log.info("UserContext: {}", UserContext.get());
        return "a";
    }

}
```

访问 info 接口，查看日志：

```java
x.luckypeak.playground.a.InfoController  : UserContext: User(accountId=1)
```

至此便实现了在 服务A 代码内通过 UserContext.get() 随时随地访问用户信息的需求。

## 基于 `feign.RequestInterceptor`​ 在服务间传递用户信息

添加 openfeing 依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>3.1.5</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-okhttp</artifactId>
            <version>11.10</version>
            <scope>compile</scope>
        </dependency>
```

新建默认 feign 配置，并在其中注册 RequestInterceptor：

```java
@Configuration
public class DefaultFeignConfig {

    public static final String ACCOUNT_ID = "X-Account-Id";

    @Bean
    public RequestInterceptor userContextRequestInterceptor(){
        return template -> {
            User user = UserContext.get();
            String accountId = Optional.ofNullable(user).map(User::getAccountId).orElse(null);
            template.header(ACCOUNT_ID, accountId);
        };
    }

}
```

新建 BClient 接口作为 服务B 的 feign client：

```java
@FeignClient(value="b", url = "http://localhost:8081")
public interface BClient {

    @GetMapping("/info")
    String info();

}
```

复制 服务A 的代码新建 服务B，修改端口，和 info 实现：

```java
// application.properties
spring.application.name=b
server.port=8081


@Slf4j
@RestController
@RequestMapping("/info")
public class InfoController {

    @GetMapping
    public String info() {
        log.info("UserContext: {}", UserContext.get());
        return "b";
    }

}
```

在 AApplication.InfoController#info 中通过 feign 访问 服务B 的 info 接口：

```java
@Slf4j
@RestController
@RequestMapping("/info")
@RequiredArgsConstructor
public class InfoController {

    private final BClient bClient;

    @GetMapping
    public String info() {
        log.info("UserContext: {}", UserContext.get());

        bClient.info();

        return "a";
    }

}
```

分别启动 服务B 和 服务 A，访问 服务A 的 info 接口：

```shell
# 服务A
x.luckypeak.playground.a.InfoController  : UserContext: User(accountId=1)

# 服务B
x.luckypeak.playground.b.InfoController  : UserContext: User(accountId=1)
```

至此并实现在服务间通过 feign 调用时传递用户信息的需求。

## 基于 [transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local) 在多线程环境下传递用户信息

通过复制 服务B 的代码，并对 端口配置 和 info 做相应修改后新建 服务C 和 服务D。

在 服务B 的 info 接口中并发调用 服务C 和 服务D 的 info 接口：

```java
@Slf4j
@RestController
@RequestMapping("/info")
@RequiredArgsConstructor
public class InfoController {

    private final CClient cClient;
    private final DClient dClient;
    private static final Executor executor = Executors.newFixedThreadPool(5);

    @GetMapping
    public String info() {
        log.info("UserContext: {}", UserContext.get());

        CompletableFuture<Void> cfC = CompletableFuture.runAsync(cClient::info, executor);
        CompletableFuture<Void> cfD = CompletableFuture.runAsync(dClient::info, executor);

        CompletableFuture.allOf(cfC, cfD).join();

        return "b";
    }

}
```

启动 服务A、B、C、D，调用 服务A 的 info 接口：

```shell
# a
x.luckypeak.playground.a.InfoController  : UserContext: User(accountId=1)

# b
x.luckypeak.playground.b.InfoController  : UserContext: User(accountId=1)

# c
x.luckypeak.playground.c.InfoController  : UserContext: User(accountId=null)

# d
x.luckypeak.playground.d.InfoController  : UserContext: User(accountId=null)
```

可以看到 服务C 和 服务D 并未正确拿到用户信息。可以引入 [transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local) 来解决该问题。

引入依赖：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.5</version>
</dependency>
```

修改 服务B 的 UserContext 类，用 TransmittableThreadLocal 替换 ThreadLocal：

```java
public class UserContext {

    private static final TransmittableThreadLocal<User> USER_CONTEXT_HOLDER = new TransmittableThreadLocal<>();


  	// ... 不变

}
```

修改 服务B 的 InfoController 类，替换创建线程池的代码：

```java
@Slf4j
@RestController
@RequestMapping("/info")
@RequiredArgsConstructor
public class InfoController {

    private final CClient cClient;
    private final DClient dClient;
    private static final Executor executor = TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(5));

   // ... 不变
}
```

重新启动 服务B，调用 服务A 的 info 接口：

```shell
# a
x.luckypeak.playground.a.InfoController  : UserContext: User(accountId=1)

# b
x.luckypeak.playground.b.InfoController  : UserContext: User(accountId=1)

# c
x.luckypeak.playground.c.InfoController  : UserContext: User(accountId=1)

# d
x.luckypeak.playground.d.InfoController  : UserContext: User(accountId=1)
```

可以看到四个服务都可以正确拿到用户信息了。
