# （十五）sould的Resilience4j体验和原理(下)

##  目标
* Resilience4JPlugin核心源码解读

## Resilience4JPlugin核心源码
>1: doExecute：、从上下文中拿到配置信息，
>2:Resilience4JHandle是Resilience4J的参数对象类，根据rule值，json转化而来，即拿到我们在admin控制台配置的参数
>3:circuitEnable：是否开启熔断，配置0：关闭，直接走 rateLimiter方法：处理过程中存在依次则会执行fallback，没有配置fallback，则直接判处异常
>4：circuitEnable配置 1：开启，则走combined方法
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

 ```
* Resilience4JPlugin#combined解析
 ```Java   
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
>1: Resilience4JConf conf = Resilience4JBuilder.build(rule)：根据配置构建CircuitBreakerConfig（断路器）   TimeLimiterConfig（限时）  RateLimiterConfig（限流），在soul-plugin-resilience4j#pom.xml
有添加相关依赖
```Java   
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-circuitbreaker</artifactId>
            <version>${resilience.version}</version>
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-timelimiter</artifactId>
            <version>${resilience.version}</version>
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-ratelimiter</artifactId>
            <version>${resilience.version}</version>
        </dependency>
```

* CombinedExecutor#run，run是Resilience4插件的核心处理逻辑，在这里完成对soul的增强
> 1:RateLimiter和CircuitBreaker对象都有Resilience4JRegistryFactory工厂创建而来，Resilience4JRegistryFactory维护了两个属性RateLimiterRegistry和CircuitBreakerRegistry，
> 其中一个基于ConcurrentHashMap 的 CircuitBreakerRegistry ，CircuitBreakerRegistry 是线程安全的，并且是原子操作。开发者可以使用 CircuitBreakerRegistry 来创建和检索 CircuitBreaker 的实例，RateLimiterRegistry也是类似
> 2: run.transformDeferred:是响应式编程 Reactor转换的使用
> 3：依次断路器的操作，限流操作，设置超时时间，如果超时了抛出超时异常，

```Java   
    public <T> Mono<T> run(final Mono<T> run, final Function<Throwable, Mono<T>> fallback, final Resilience4JConf resilience4JConf) {
        RateLimiter rateLimiter = Resilience4JRegistryFactory.rateLimiter(resilience4JConf.getId(), resilience4JConf.getRateLimiterConfig());
        CircuitBreaker circuitBreaker = Resilience4JRegistryFactory.circuitBreaker(resilience4JConf.getId(), resilience4JConf.getCircuitBreakerConfig());
       //断路器的操作
        Mono<T> to = run.transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                //限流操作
                .transformDeferred(RateLimiterOperator.of(rateLimiter))
                //设置超时时间
                .timeout(resilience4JConf.getTimeLimiterConfig().getTimeoutDuration())
                //如果超时了抛出超时异常
                .doOnError(TimeoutException.class, t -> circuitBreaker.onError(
                        resilience4JConf.getTimeLimiterConfig().getTimeoutDuration().toMillis(),
                        TimeUnit.MILLISECONDS,
                        t));
        if (fallback != null) {
            to = to.onErrorResume(fallback);
        }
        return to;
    }
  ```

## 总结
*  
