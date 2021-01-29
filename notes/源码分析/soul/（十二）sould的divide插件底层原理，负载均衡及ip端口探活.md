# （十二）sould的divide插件底层原理，负载均衡及ip端口探活

##  目标
*  divide插件的类信息
*  divide插件底层原理
*  soul-bootstrap端ip端口探活




## divide插件的类信息
>  divide的具体使用使用在小节一里有分析了，这里不阐述
>  soul-booostrap依赖包soul-spring-boot-starter-plugin-divide，里面DividePlugin（divide插件），DividePluginDataHandler（divide插件数据处理器），WebSocketPlugin，ReactorNettyWebSocketClient的初始化
>  DividePlugin继承AbstractSoulPlugin (AbstractSoulPlugin实现SoulPlugin)
> soul-spring-boot-starter-plugin-divide 包同时依赖了包soul-plugin-divide，看如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129073828510.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> balance负载均衡，UpstreamCacheManager探活
 ```Java   
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
      websocket :
        urls: ws://localhost:9095/websocket,ws://localhost:9096/websocket,ws://localhost:9097/websocket

 ```

##  divide插件底层原理
*  divide插件核心使用责任链模式和模板方法模式
> 责任链模式
 ```Java   
        @Override
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }

 ```
> 模板方法模式 在小节十三中，对模板方法做了详细阐述，
 ```Java   
         @Override
         public Mono<Void> execute(final ServerWebExchange exchange) {
             return Mono.defer(() -> {
                 if (this.index < plugins.size()) {
                     SoulPlugin plugin = plugins.get(this.index++);
                     Boolean skip = plugin.skip(exchange);
                     if (skip) {
                         return this.execute(exchange);
                     }
                     return plugin.execute(exchange, this);
                 }
                 return Mono.empty();
             });
         }
 
  ```

*  DividePlugin#doExecute核心处理代码
> 1:soulContext拿到从上下文拿到配置信息，即admin的配置同步打到soul-bootstrap
> 2:ruleHandle 获取负载均衡的配置规则，重试次数，超时时间
> 3：UpstreamCacheManager从获取可用调用ip信息，DivideUpstream带权重
> 4：LoadBalanceUtils.selector通过负载选择哪个具体调用ip  
> 5：realURL真实的请求地址
> 6：httpUrl，重试次数，超时时间添加到上下文中
> 7: chain.execute(exchange)：继续执行下一个插件链，则直接回到SoulWebHandler#execute
 ```Java   
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url
        String domain = buildDomain(divideUpstream);
        String realURL = buildRealURL(domain, soulContext, exchange);
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        // set the http timeout
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }

  ```
## divide负载均衡
> DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip)
> Divide插件是通过SPI的方式将负载均衡的类加载进来，然后通过getJoin方法真正选择出使用的具体算法实现类，soul-plugin-divide根目录存在org.dromara.soul.plugin.divide.balance.LoadBalance

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129073917775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)

> AbstractLoadBalanced的继承类有三个，HashLoadBalance，RandomLoadBalance，RoundRobinLoadBalance，
>具体通过 doSelect调用到具体哪个继承类处理，此处也是模板方法运用
 ```Java   
    @Override
    public DivideUpstream select(final List<DivideUpstream> upstreamList, final String ip) {
        if (CollectionUtils.isEmpty(upstreamList)) {
            return null;
        }
        if (upstreamList.size() == 1) {
            return upstreamList.get(0);
        }
        return doSelect(upstreamList, ip);
    }
 ```
## soul-bootstrap端ip端口探活
*  具体在UpstreamCacheManager类，UpstreamCacheManager初始化时，会新建ScheduledThreadPoolExecutor一个定时线程池，每间隔30秒调用当前类的scheduled方法
 ```Java   
    private UpstreamCacheManager() {
        boolean check = Boolean.parseBoolean(System.getProperty("soul.upstream.check", "false"));
        if (check) {
            new ScheduledThreadPoolExecutor(1, SoulThreadFactory.create("scheduled-upstream-task", false))
                    .scheduleWithFixedDelay(this::scheduled,
                            30, Integer.parseInt(System.getProperty("soul.upstream.scheduledTime", "30")), TimeUnit.SECONDS);
        }
    }
  ```
* scheduled方法  
> UPSTREAM_MAP全部的ip信息 ，遍历 UPSTREAM_MAP值，检查如果存在则放入UPSTREAM_MAP_TEMP，不存在则移除，UPSTREAM_MAP_TEMP可用ip信息。
> check(v)：检查是否可用的核心方法
 ```Java   
    private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            UPSTREAM_MAP.forEach((k, v) -> {
                List<DivideUpstream> result = check(v);
                if (result.size() > 0) {
                    UPSTREAM_MAP_TEMP.put(k, result);
                } else {
                    UPSTREAM_MAP_TEMP.remove(k);
                }
            });
        }
    }
  ```
* check(v)方法:先checkIP(url))检查ip的正确性
> isHostConnector
 ```Java   
    public static boolean checkUrl(final String url) {
        if (StringUtils.isBlank(url)) {
            return false;
        }
        if (checkIP(url)) {
            String[] hostPort;
            if (url.startsWith(HTTP)) {
                final String[] http = StringUtils.split(url, "\\/\\/");
                hostPort = StringUtils.split(http[1], Constants.COLONS);
            } else {
                hostPort = StringUtils.split(url, Constants.COLONS);
            }
            return isHostConnector(hostPort[0], Integer.parseInt(hostPort[1]));
        } else {
            return isHostReachable(url);
        }
    }
  ```
> 新建Socket类，Socket使用同步IO的方式直接尝试连接，判断ip是否可用，到处里即是我们soul-bootstrap端探活
 ```Java   
    private static boolean isHostConnector(final String host, final int port) {
        try (Socket socket = new Socket()) {
            socket.connect(new InetSocketAddress(host, port));
        } catch (IOException e) {
            return false;
        }
        return true;
    }
  ```

## 总结
*  目前只分析soul-bootstrap端ip端口探活，admin端探活缺失，后续继续分析