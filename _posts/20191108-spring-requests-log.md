---
title: 如何在spring boot中打rest请求日志
tags:
  - JAVA
  - SPRING
  - SPRING BOOT
date: 2019-11-08 11:42:27
category: CODING
---

这个问题就是之前说的碰到过好多次的问题，这里记录一下

## HandlerInterceptorAdapter

这个玩意是spring专门用来拦截处理各种http请求的适配器，其中有四个方法：

* preHandle 请求来的时候做的预处理
* postHandle 请求出去的时候做的预处理
* afterCompletion 请求完成之后的处理
* afterConcurrentHandlingStarted 如果handler处于同步化执行时，测个函数会替代postHandle和afterCompletion，使用这个函数时尽量不要改变reqest和response,典型的用法是清除threadLocal变量。这个我没用过我不知道是干什么的，上面那段话是我抄官方文档的

比如说我们的controller长这样：

```java
@RestController
public class TestController {

    @Autowired
    private TestService testService;

    @GetMapping("/test")
    public String test() {
        return testService.test();
    }
}
```

之后我们为她添加一个HandlerInterceptorAdapter

```java
@Component
@Slf4j
public class AccessLogInterceptor extends HandlerInterceptorAdapter {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        log.info("in preHandle.");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
                           @Nullable ModelAndView modelAndView) throws Exception {
        log.info("in postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
                                @Nullable Exception ex) throws Exception {
        log.info("in afterCompletion.");
    }
}
```

除此之外我们还需要把这个intercreptor添加到config里面

```java
@Component
@EnableWebMvc
@Configuration
public class RestAdapter implements WebMvcConfigurer {
    @Bean
    public AccessLogInterceptor accessLog() {
        return new AccessLogInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        InterceptorRegistration registration = registry.addInterceptor(accessLog());
        registration.addPathPatterns("/api/**");
    }
}
```

试试结果吧

```log
2019-11-08 15:27:05.225  INFO 23683 --- [  XNIO-1 task-1] m.s.y.rest.config.AccessLogInterceptor   : in preHandle.
2019-11-08 15:27:05.542  INFO 23683 --- [  XNIO-1 task-1] m.s.y.rest.config.AccessLogInterceptor   : in postHandle
2019-11-08 15:27:05.543  INFO 23683 --- [  XNIO-1 task-1] m.s.y.rest.config.AccessLogInterceptor   : in afterCompletion.
```

完美

## postHandle 和 afterCompletion 的区别

postHandle 会在handler处理完成之后执行，而afterCompletion会在整个请求处理完毕之后执行。也就是执行

一个典型的例子是如果在controller中抛错，那么postHandle不会执行，而afterCompletion会执行

## ControllerAdvice

上面说到了exception，大多数情况下，如果你默认把报错信息暴露给用户，你的系统就很容易被人扒光了。所以一般的错误信息我们都会包一层

那么怎么处理错误信息呢？当然就是用我们神奇的ControllerAdvice了

```java
@ControllerAdvice
public class ErrorHandler {
    @ExceptionHandler({RuntimeException.class})
    public ResponseEntity<String> exceptionHandle(Exception e) {
        return new ResponseEntity<>("Internal error", HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

之后如果你的代码抛错就会显示你的自定义内容而不是默认的错误信息了。

## 如何正确地打日志

一个神奇的东西：

### MDC

[具体看这里](http://logback.qos.ch/manual/mdc.html)

简而言之就是一个多线程处理上下文的东西，每个request是一个线程，这样每个request里面的MDC拿到的都是一样的东西。

之后要考虑我们要记录的东西了。按照我之前的经验来说，每个request需要记录：

* requestId: 自动生成且每个request唯一
* ip
* uri
* params
* method
* 开始时间
* 结束时间
* 执行结果以及报错信息
* user-agent
* 用户身份或者其他重要信息（如果必要的话）

这样的话我们可以适当的该一下我们的intercerptor

preHandle:

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {
    String requestId = UUID.randomUUID().toString();
    String startTime = Long.toString(new Date().getTime());
    String uri = request.getRequestURI();
    String params = request.getParameterMap().entrySet().stream()
            .map(e -> Arrays.stream(e.getValue()).map(v -> e.getKey() + "=" + v)
                    .reduce((v1, v2) ->v1 + "&" + v2).orElse("")
            )
            .reduce((p1, p2) -> p1 + "&" + p2)
            .orElse("");
    String method = request.getMethod();
    String ip = StringUtils.isEmpty(request.getHeader("X-FORWARDED-FOR")) ? request.getLocalAddr() : request.getHeader("X-FORWARDED-FOR");
    String userAgent = request.getHeader("user-agent");
    log.info("new request: requestId: {}, startTime:{}, uri: {}, params: {}, method: {}, ip: {}, user-agent: {}",
            requestId, startTime, uri, params, method, ip, userAgent);
    MDC.put("requestId", requestId);
    MDC.put("startTime", startTime);
    MDC.put("uri", uri);
    MDC.put("params", params);
    MDC.put("method", method);
    MDC.put("ip", ip);
    MDC.put("userAgent", userAgent);
    return true;
}
```

ControllerAdvice:

```java
@ControllerAdvice
@Slf4j
public class ErrorHandler {
    @ExceptionHandler({RuntimeException.class})
    public ResponseEntity<String> exceptionHandle(Exception e) {
        log.error("error: ", e);
        MDC.put("result", "INTERNAL_SERVER_ERROR");
        return new ResponseEntity<>("Internal error", HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

postHandle & afterCompletion 别忘了最后把MDC里面的东西清空释放内存

```java
@Override
public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
                        @Nullable ModelAndView modelAndView) throws Exception {
    MDC.put("result", "OK");
}

@Override
public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
                            @Nullable Exception ex) throws Exception {
    String endTime = Long.toString(new Date().getTime());
    log.info("request end: requestId: {}, endTime:{}, result: {}", MDC.get("requestId"), endTime, MDC.get("result"));

    MDC.remove("requestId");
    MDC.remove("startTime");
    MDC.remove("uri");
    MDC.remove("params");
    MDC.remove("method");
    MDC.remove("ip");
    MDC.remove("userAgent");
    MDC.remove("result");
}
```

## 如何给request body 打日志

你会发现我上面没有记request body的内容，为什么呢？因为安全性和效率问题，http request body在HttpRequestServlet里面是一个流，只能读取一次，所以直接拿出来在interceptor里面是没办法处理的

这个时候我们需要用到一个新东西

### RequestBodyAdvice

和字面上的意思一样，这个东西是用来处理request body转换到object这一过程中可以自定义一些处理

RequestBodyAdvice里有四个方法：

* supports: 这个advice什么时候生效，返回true生效，false不生效
* handleEmptyBody: 处理空body
* beforeBodyRead: body转换成object前做的处理
* afterBodyRead: body转换成object之后的处理

有了这个工具后面的事情就很简单啦，我们自己写个bodyAdvice

```java
@ControllerAdvice
@Slf4j
public class BodyAdvice extends RequestBodyAdviceAdapter {
    @Override
    public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
                                Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        log.info("request body: {}", body);
        return super.afterBodyRead(body, inputMessage, parameter, targetType, converterType);
    }
}
```

完成
