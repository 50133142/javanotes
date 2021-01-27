# （（十三）sould的springcloud插件底层原理

##  目标
* divide插件底层原理
*




## springcloud插件底层原理
> springcloud插件的具体使用使用在小节三里有分析了，这里不解释
> 启动一个adminadmin和一个网关，两个order-service，一个eureka

* 我们在SpringCloudPlugin的doExecute打断点
> 从SoulWebHandler的execute， SoulWebHandler是插件链的处理器，依次按照order大小依次执行插件
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
>  plugins对象如下图

> plugin.execute(exchange, this)执行来到，AbstractSoulPlugin的doExecute
>   if (pluginData != null && pluginData.getEnabled()) ：有疑问，这代码明天看下，
 ```Java   
   public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
 ```
>  chain.execute(exchange)到cloud插件
> 1:exchange.getAttribute(Constants.CONTEXT):拿到我们的admin配置信息
> 2：realURL真实请求业务实例的ip
> loadBalancer.choose(selectorHandle.getServiceId())通过ribbon的负载均衡，来获取具体的哪个调用的实例ip信息
> 负载均衡的实例是因为依赖了包spring-cloud-starter-netflix-ribbon，netflix-ribbon包具体处理负载

 ```Java  
    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        if (Objects.isNull(rule)) {
            return Mono.empty();
        }
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final SpringCloudRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SpringCloudRuleHandle.class);
        final SpringCloudSelectorHandle selectorHandle = GsonUtils.getInstance().fromJson(selector.getHandle(), SpringCloudSelectorHandle.class);
        if (StringUtils.isBlank(selectorHandle.getServiceId()) || StringUtils.isBlank(ruleHandle.getPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getCode(), SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }

        final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final URI uri = loadBalancer.reconstructURI(serviceInstance, URI.create(soulContext.getRealUrl()));

        String realURL = buildRealURL(uri.toASCIIString(), soulContext.getHttpMethod(), exchange.getRequest().getURI().getQuery());

        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        //set time out.
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        return chain.execute(exchange);
    }
 ```
## 总结
*  今天下班太晚了，明天继续
