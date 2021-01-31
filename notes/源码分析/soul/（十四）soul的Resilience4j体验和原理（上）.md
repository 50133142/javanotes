# （十四）sould的Resilience4j体验和原理

##  目标
* 什么是Resilience4j
* sould的Resilience4j体验



## 什么是Resilience4j
* Resilience4J是我们Spring Cloud G版本 推荐的容错方案，它是一个轻量级的容错库
* 借鉴了Hystrix而设计，并且采用JDK8 这个函数式编程，即lambda表达式
* 相比之下， Netflix Hystrix 对Archaius 具有编译依赖性，Resilience4j你无需引用全部依赖，可以根据自己需要的功能引用相关的模块即可
Hystrix很早就不更新了，Spring官方已经出现了Netflix Hystrix的替换方案，即Resilence4J
* Resilience4J 提供了一系列增强微服务的可用性功能：
    *  断路器 CircuitBreaker
    *  限流 RateLimiter
    *  基于信号量的隔离
    *  缓存
    *  限时 Timelimiter
    *  请求重启  Retry

* 官方提供的依赖包
 ```Java   
     <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-circuitbreaker</artifactId>
            <version>${resilience.version}</version>
     </dependency>
  ```

## sould的Resilience4j体验
* 首先在admin控制台的插件管理开始Resilience4j
* 在网关添加依赖
 ```Java   
       <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-ratelimiter</artifactId>
            <version>${project.version}</version>
        </dependency>
  ```
* 启动一个admin，一个soul-bootstrap，一个soulTtestHttp。三个服务

* 在admin控制台找到插件列表的Resilience4j，自定义配置，如下图，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130070424520.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)



> 阻塞问题了，Resilience4j和官网提供的文档配置属性不一致，一些数据对不上，
* Resilience4j的配置规则详解（摘自官网）：

    * timeoutDurationRate：等待获取令牌的超时时间，单位ms，默认值：5000。

    * limitRefreshPeriod：刷新令牌的时间间隔，单位ms，默认值：500。

    * limitForPeriod：每次刷新令牌的数量，默认值：50。

    * circuitEnable：是否开启熔断，0：关闭，1：开启，默认值：0。

    * timeoutDuration：熔断超时时间，单位ms，默认值：30000。

    * fallbackUri：降级处理的uri。

    * slidingWindowSize：滑动窗口大小，默认值：100。

    * slidingWindowType：滑动窗口类型，0：基于计数，1：基于时间，默认值：0。

    * minimumNumberOfCalls：开启熔断的最小请求数，超过这个请求数才开启熔断统计，默认值：100。

    * waitIntervalFunctionInOpenState：熔断器开启持续时间，单位ms，默认值：10。

    * permittedNumberOfCallsInHalfOpenState：半开状态下的环形缓冲区大小，必须达到此数量才会计算失败率，默认值：10。

    * failureRateThreshold：错误率百分比，达到这个阈值，熔断器才会开启，默认值50。

    * automaticTransitionFromOpenToHalfOpenEnabled：是否自动从open状态转换为half-open状态，,true：是，false：否，默认值：false。
    
> 当我的Resilience4j配置如上图时 ，请求服务，会调用服务fallback接口  
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130070530371.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

> Resilience4J插件包的信息
> Resilience4JPlugin核心源码
 ```Java   
    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        Resilience4JHandle resilience4JHandle = GsonUtils.getGson().fromJson(rule.getHandle(), Resilience4JHandle.class);
        if (resilience4JHandle.getCircuitEnable() == 1) {
            return combined(exchange, chain, rule);
        }
        return rateLimiter(exchange, chain, rule);
    }

    private Mono<Void> rateLimiter(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        return ratelimiterExecutor.run(
                chain.execute(exchange), fallback(ratelimiterExecutor, exchange, null), Resilience4JBuilder.build(rule))
                .onErrorResume(throwable -> ratelimiterExecutor.withoutFallback(exchange, throwable));
    }

    private Mono<Void> combined(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        Resilience4JConf conf = Resilience4JBuilder.build(rule);
        return combinedExecutor.run(
                chain.execute(exchange).doOnSuccess(v -> {
                    if (exchange.getResponse().getStatusCode() != HttpStatus.OK) {
                        HttpStatus status = exchange.getResponse().getStatusCode();
                        exchange.getResponse().setStatusCode(null);
                        throw new CircuitBreakerStatusCodeException(status);
                    }
                }), fallback(combinedExecutor, exchange, conf.getFallBackUri()), conf);
    }
  ```
## 总结
*  还需要补充Resilience4j怎么使用
