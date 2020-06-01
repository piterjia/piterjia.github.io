---
layout: post
title: guava 限流方案
categories: Distributed
description: guava 限流方案
keywords: guava, distributed, limiting
---

Google开源工具包Guava提供了限流工具类RateLimiter，基于令牌桶算法实现。

# 常用方法：

- create（Double permitsPerSecond）方法根据给定的（令牌:单位时间（1s））比例为令牌生成速率
- tryAcquire（）方法尝试获取一个令牌，立即返回true/false，不阻塞，重载方法具备设置获取令牌个数、获取最大等待时间等参数
- acquire（）方法与tryAcquire类似，但是会阻塞，尝试获取一个令牌，没有时则阻塞直到获取成功


**分别举一个简单的例子如下：**

## 非阻塞限流
如果无法获取到指定个数的令牌，直接返回失败

```
    RateLimiter limiter = RateLimiter.create(2.0);

    // 非阻塞限流
    @GetMapping("/tryAcquire")
    public String tryAcquire(Integer count) {
        if (limiter.tryAcquire(count)) {
            log.info("success, rate is {}", limiter.getRate());
            return "success";
        } else {
            log.info("fail, rate is {}", limiter.getRate());
            return "fail";
        }
    }
```


## 限定时间的非阻塞限流
限定时间内，如果无法获取到指定个数的令牌，直接返回失败。否则返回成功。

```
    RateLimiter limiter = RateLimiter.create(2.0);

    // 
    @GetMapping("/tryAcquireWithTimeout")
    public String tryAcquireWithTimeout(Integer count, Integer timeout) {
        if (limiter.tryAcquire(count, timeout, TimeUnit.SECONDS)) {
            log.info("success, rate is {}", limiter.getRate());
            return "success";
        } else {
            log.info("fail, rate is {}", limiter.getRate());
            return "fail";
        }
    }
```


## 同步阻塞限流
一直等到获取到指定个数的令牌，再往下执行。
```
    RateLimiter limiter = RateLimiter.create(2.0);
    @GetMapping("/acquire")
    public String acquire(Integer count) {
        limiter.acquire(count);
        log.info("success, rate is {}", limiter.getRate());
        return "success";
    }
```


# 自定义注解应用

采用 SpringBoot + Interceptor + 自定义注解的方式，提供简介的guava限流应用方式。

## 1、maven依赖

```
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>18.0</version>
    </dependency>
```

## 2、自定义注解

```
import java.lang.annotation.*;
import java.util.concurrent.TimeUnit;

/**
 * RequestLimiter 自定义注解接口限流
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestLimiter {

    /**
     * 每秒创建令牌个数，默认:10
     */
    double QPS() default 10D;

    /**
     * 获取令牌等待超时时间 默认:500
     */
    long timeout() default 500;

    /**
     * 超时时间单位 默认:毫秒
     */
    TimeUnit timeunit() default TimeUnit.MILLISECONDS;

    /**
     * 无法获取令牌返回提示信息
     */
    String msg() default "亲，服务器快被挤爆了，请稍后再试！";
}
```

## 3、拦截器

```
import com.google.common.util.concurrent.RateLimiter;
import com.mowanka.framework.annotation.RequestLimiter;
import com.mowanka.framework.web.result.GenericResult;
import com.mowanka.framework.web.result.StateCode;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 请求限流拦截器
 */
@Component
public class RequestLimiterInterceptor extends GenericInterceptor {

    /**
     * 不同的方法存放不同的令牌桶
     */
    private final Map<String, RateLimiter> rateLimiterMap = new ConcurrentHashMap<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        try {
            if (handler instanceof HandlerMethod) {
                HandlerMethod handlerMethod = (HandlerMethod) handler;
                RequestLimiter rateLimit = handlerMethod.getMethodAnnotation(RequestLimiter.class);
                //判断是否有注解
                if (rateLimit != null) {
                    // 获取请求url
                    String url = request.getRequestURI();
                    RateLimiter rateLimiter;
                    // 判断map集合中是否有创建好的令牌桶
                    if (!rateLimiterMap.containsKey(url)) {
                        // 创建令牌桶,以n r/s往桶中放入令牌
                        rateLimiter = RateLimiter.create(rateLimit.QPS());
                        rateLimiterMap.put(url, rateLimiter);
                    }
                    rateLimiter = rateLimiterMap.get(url);
                    // 获取令牌
                    boolean acquire = rateLimiter.tryAcquire(rateLimit.timeout(), rateLimit.timeunit());
                    if (acquire) {
                        //获取令牌成功
                        return super.preHandle(request, response, handler);
                    } else {
                        log.warn("请求被限流,url:{}", request.getServletPath());
                        # 此处返回错误提示信息给请求者
                        return false;
                    }
                }
            }
            return true;
        } catch (Exception var6) {
            var6.printStackTrace();
            # 此处返回错误提示信息给请求者
            return false;
        }
    }

}

```


## 4、注册拦截器

```
/**
 * springboot - WebMvcConfig
 */
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    /**
     * 请求限流拦截器
     */
    @Autowired
    protected RequestLimiterInterceptor requestLimiterInterceptor;

    public WebMvcConfig() {}

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 请求限流
        registry.addInterceptor(requestLimiterInterceptor).addPathPatterns("/**");
    }

}
```

## 5、在需要的接口上配置自定义注解

```
@RequestLimiter(QPS = 5D, timeout = 200, timeunit = TimeUnit.MILLISECONDS,msg = "服务器繁忙,请稍后再试")
@GetMapping("/test")
@ResponseBody
public String test(){
      return "";
}
```


## 五.总结

本文首先简单介绍了下guava的基本使用。

然后基于它的基本使用，实现了自定义注解的方式，简化接口限流的代码。该代码只适于单个应用进行接口限流。

如果是分布式项目或者微服务项目可以采用redis来实现限流功能。




