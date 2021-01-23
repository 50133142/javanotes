# （八） soul网关的websocket同步数据源码分析(下)

##  目标
> 第七小节主要分析了soul-admin通过websocket发送数据
> 本小节主要分析soul-bootstrap怎么处理websocket发来的数据



## soul-sync-data-websocket包
>如下图 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123183728507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


*  SoulWebsocketClient
*  WebsocketConfig
*  DataHandler的实现类和抽象类
*  WebsocketSyncDataService
    
##  websocket接收发来的数据
>websocketDataHandler接收发来的消息onMessage，再传递到handleResult方法，

``` Java
    private void handleResult(final String result) {
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
```
## websocketDataHandler  
> 默认的构造函数把AbstractDataHandler继承类初始化存在EnumMap对象中
*  AbstractDataHandler的继承类：PluginDataHandler 插件处理器，  SelectorDataHandler选择器处理器， RuleDataHandler 规则处理器，AuthDataHandler权限处理器 ，MetaDataHandler
* ENUM_MAP.get(type).handle(json, eventType):根据ENUM_MAP类型去调用不同的处理器，

``` Java
    private static final EnumMap<ConfigGroupEnum, DataHandler> ENUM_MAP = new EnumMap<>(ConfigGroupEnum.class);

    /**
     * Instantiates a new Websocket data handler.
     *
     * @param pluginDataSubscriber the plugin data subscriber
     * @param metaDataSubscribers  the meta data subscribers
     * @param authDataSubscribers  the auth data subscribers
     */
    public WebsocketDataHandler(final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers,
                                final List<AuthDataSubscriber> authDataSubscribers) {
        ENUM_MAP.put(ConfigGroupEnum.PLUGIN, new PluginDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.SELECTOR, new SelectorDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.RULE, new RuleDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.APP_AUTH, new AuthDataHandler(authDataSubscribers));
        ENUM_MAP.put(ConfigGroupEnum.META_DATA, new MetaDataHandler(metaDataSubscribers));
    }
 
   //根据type具体哪个模板方法
   public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
```

## 接着上一步分析：我们走到AbstractDataHandler类的handler方法，由于AbstractDataHandlerdo是个抽象类，具体调用的它的继承类处理方式(doRefresh，doUpdate，doDelete),这里巧妙的使用模板设计模式 
* doRefresh，缓存初始化或更新
* doUpdate， 更新缓存或具体协议处理
* doDelete   清空缓存
``` Java
    public void handle(final String json, final String eventType) {
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList);
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
    }
```

## 这里我们分析：doUpdate(dataList)
>SelectorDataHandler
``` Java
    @Override
    protected void doUpdate(final List<SelectorData> dataList) {
        dataList.forEach(pluginDataSubscriber::onSelectorSubscribe);
    }
```
> pluginDataSubscriber::onSelectorSubscribe
> CommonPluginDataSubscriber的onSelectorSubscribe

``` Java

  @Override
     public void onSelectorSubscribe(final SelectorData selectorData) {
         subscribeDataHandler(selectorData, DataEventTypeEnum.UPDATE);
     }
```
     
> CommonPluginDataSubscriber的subscribeDataHandler  
>  subscribeDataHandler方法是核心： 修改缓存 BaseDataCache： Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
> BaseDataCache: 中存放了三个ConcurrentMap，缓存了注册上来的插件、选择器、规则的数据，这就是为什么soul高性能的其中一个重要优化，配置不从数据库、redis、其他配置中心拉去数据，
>配置基于JVM缓存操作，性能非常快

``` Java
    private static final ConcurrentMap<String, PluginData> PLUGIN_MAP = Maps.newConcurrentMap();
    
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    private static final ConcurrentMap<String, List<RuleData>> RULE_MAP = Maps.newConcurrentMap();
```
> 根据插件名称，找到插件处理器 PluginDataHandler 处理选择器的更新数据 
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-764CB2p0-1611398167199)(../soul/png/handlePlugin.png "handlePlugin")]


``` Java

    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 修改缓存 BaseDataCache： Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    // 根据插件名称，找到插件处理器 PluginDataHandler 处理选择器的更新数据
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }

```
*

## 使用websocket同步的时候，特别要注意断线重连，也叫保持心跳
>多个soul-admin实例，初始化多个WebSocketClient，onOpen和onMessage接收websocket发了来的数据,每个soul-admin实例，存在一个调度线程30秒进行一次断线重连

``` Java
    public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        //多个soul-admin实例，初始化多个WebSocketClient，onOpen和onMessage接收websocket发了来的数据

        for (String url : urls) {
            try {
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
            //每个soul-admin实例，存在一个调度线程，去进行断线重连
            for (WebSocketClient client : clients) {
                //进行连接
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
                //使用调度线程池进行断线重连，30秒进行一次
                executor.scheduleAtFixedRate(() -> {
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }
```
> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit)
>上面的四个参数进行讲解：
 第一个command参数是任务实例，
 第二个initialDelay参数是初始化延迟时间，
 第三个period参数是间隔时间，
 第四个unit参数是时间单元。

## 总结
* soul的websocket接收数据处理，
* 使用模板设计模式AbstractDataHandler，去处理不同handler
* 使用了JVM缓存管理配置信息 
* websocket存在一个调度线程30秒进行一次断线重连