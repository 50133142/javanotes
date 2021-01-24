# （九）soul网关的http长轮询同步数据

##  目标
* soul-admin怎么开启http长轮训和初始化相应类
* soul-bootstrap的配置和初始化
* http怎么长时间轮询
* soul-bootstrap请求admin的接口/configs/listener，具体做了什么



## soul-admin怎么开启http长轮训和初始化相应类
* 怎么开启http长轮训
>soul-admin的配置
 ```Java   
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
  sync:
      http:
        enabled: true
 ```

>http长轮训初始化在DataSyncConfiguration的内部类，ZookeeperListener，NacosListener，WebsocketListener也是在DataSyncConfiguration初始化的


 ```Java   
    @Configuration
    @ConditionalOnProperty(name = "soul.sync.http.enabled", havingValue = "true")
    @EnableConfigurationProperties(HttpSyncProperties.class)
    static class HttpLongPollingListener {

        @Bean
        @ConditionalOnMissingBean(HttpLongPollingDataChangedListener.class)
        public HttpLongPollingDataChangedListener httpLongPollingDataChangedListener(final HttpSyncProperties httpSyncProperties) {
            return new HttpLongPollingDataChangedListener(httpSyncProperties);
        }

    }
 ```
* HttpLongPollingDataChangedListener源码分析
> HttpLongPollingDataChangedListener继承AbstractDataChangedListener
> 而AbstractDataChangedListener implements DataChangedListener, InitializingBean ，所以HttpLongPollingDataChangedListener的afterPropertiesSet会在spring初始化当前类后调用此方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021012421533723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> AbstractDataChangedListener的afterPropertiesSet方法
 ```Java   
    @Override
    public final void afterPropertiesSet() {
        updateAppAuthCache();
        updatePluginCache();
        updateRuleCache();
        updateSelectorCache();
        updateMetaDataCache();
        afterInitialize();
    }
 ```

>updateAppAuthCache ，updatePluginCache，updateRuleCache，updateSelectorCache，updateSelectorCache都是先获取mysql数据，然后刷新本地缓存
>afterInitialize该方法中构建一个延迟任务，每隔5分钟刷新下本地缓存，调用 this.refreshLocalCache();
 ```Java   
    @Override
    protected void afterInitialize() {
        long syncInterval = httpSyncProperties.getRefreshInterval().toMillis();
        // Periodically check the data for changes and update the cache
        scheduler.scheduleWithFixedDelay(() -> {
            log.info("http sync strategy refresh config start.");
            try {
                this.refreshLocalCache();
                log.info("http sync strategy refresh config success.");
            } catch (Exception e) {
                log.error("http sync strategy refresh config error!", e);
            }
        }, syncInterval, syncInterval, TimeUnit.MILLISECONDS);
        log.info("http sync strategy refresh interval: {}ms", syncInterval);
    }

    private void refreshLocalCache() {
        this.updateAppAuthCache();
        this.updatePluginCache();
        this.updateRuleCache();
        this.updateSelectorCache();
        this.updateMetaDataCache();
    }
 ```

## soul-bootstrap的配置和初始化
 ```Java   
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
      http:
        url : http://localhost:9095
 ```
>依赖
 ```Java   
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
            <version>${project.version}</version>
        </dependency>
 ```
*  HttpSyncDataService的初始化
>HttpSyncDataService需要注入HttpConfig，PluginDataSubscriber，MetaDataSubscriber，AuthDataSubscriber
 ```Java   
@Configuration
@ConditionalOnClass(HttpSyncDataService.class)
@ConditionalOnProperty(prefix = "soul.sync.http", name = "url")
@Slf4j
public class HttpSyncDataConfiguration {

    /**
     * Http sync data service.
     *
     * @param httpConfig        the http config
     * @param pluginSubscriber the plugin subscriber
     * @param metaSubscribers   the meta subscribers
     * @param authSubscribers   the auth subscribers
     * @return the sync data service
     */
    @Bean
    public SyncDataService httpSyncDataService(final ObjectProvider<HttpConfig> httpConfig, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use http long pull sync soul data");
        return new HttpSyncDataService(Objects.requireNonNull(httpConfig.getIfAvailable()), Objects.requireNonNull(pluginSubscriber.getIfAvailable()),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }
 ```
* HttpSyncDataService构造函数
>初始化对象用于初始化了各个数据类型的刷新类， EnumMap<ConfigGroupEnum, DataRefresh> ENUM_MAP = new EnumMap<>(ConfigGroupEnum.class);
>核心方法this.start()
 ```Java   
    public HttpSyncDataService(final HttpConfig httpConfig, final PluginDataSubscriber pluginDataSubscriber,
                               final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        //初始化对象用于初始化了各个数据类型的刷新类， EnumMap<ConfigGroupEnum, DataRefresh> ENUM_MAP = new EnumMap<>(ConfigGroupEnum.class);
        this.factory = new DataRefreshFactory(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        this.httpConfig = httpConfig;
        this.serverList = Lists.newArrayList(Splitter.on(",").split(httpConfig.getUrl()));
        this.httpClient = createRestTemplate();
        //核心方法
        this.start();
    }
 ```
* 我们进到this.start()方法
 ```Java   
    private void start() {
        // It could be initialized multiple times, so you need to control that.
        if (RUNNING.compareAndSet(false, true)) {
            // fetch all group configs.
            this.fetchGroupConfig(ConfigGroupEnum.values());
            int threadSize = serverList.size();
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // start long polling, each server creates a thread to listen for changes.
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
        } else {
            log.info("soul http long polling was started, executor=[{}]", executor);
        }
    }

 ```
>核心代码：this.fetchGroupConfig(ConfigGroupEnum.values())，
* 根据yml配置拼接接口地址 url:http://localhost:9095/configs/fetch?groupKeys=APP_AUTH&groupKeys=PLUGIN&groupKeys=RULE&groupKeys=SELECTOR&groupKeys=META_DATA
* 请求soul-admin后拿到配置数据后，this.updateCacheWithJson(json)方法全量刷新本地jvm数据,    ENUM_MAP.values().parallelStream().forEach(dataRefresh -> success[0] = dataRefresh.refresh(data));
* ConfigGroupEnum.values(),参数我们知道第一次他会全量去admin获取配置信息 
```Java   
    private void doFetchGroupConfig(final String server, final ConfigGroupEnum... groups) {
        StringBuilder params = new StringBuilder();
        for (ConfigGroupEnum groupKey : groups) {
            params.append("groupKeys").append("=").append(groupKey.name()).append("&");
        }
        String url = server + "/configs/fetch?" + StringUtils.removeEnd(params.toString(), "&");
        log.info("request configs: [{}]", url);
        String json = null;
        try {
            json = this.httpClient.getForObject(url, String.class);
        } catch (RestClientException e) {
            String message = String.format("fetch config fail from server[%s], %s", url, e.getMessage());
            log.warn(message);
            throw new SoulException(message, e);
        }
        // update local cache
        boolean updated = this.updateCacheWithJson(json);
        if (updated) {
            log.info("get latest configs: [{}]", json);
            return;
        }
        // not updated. it is likely that the current config server has not been updated yet. wait a moment.
        log.info("The config of the server[{}] has not been updated or is out of date. Wait for 30s to listen for changes again.", server);
        ThreadUtils.sleep(TimeUnit.SECONDS, 30);
    }

 ```
## http怎么长时间轮询
* 长时间轮询处理流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210124215414976.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> 每个服务器都会创建一个线程来侦听更改， while (RUNNING.get()):会不停的执行doLongPolling方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210124215601441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


>方法this.executor.execute(new HttpLongPollingTask(server)的execute执行HttpLongPollingTask的run()
 ```Java   
        @Override
        public void run() {
            while (RUNNING.get()) {
                for (int time = 1; time <= retryTimes; time++) {
                    try {
                        doLongPolling(server);
                    } catch (Exception e) {
                        // print warnning log.
                        if (time < retryTimes) {
                            log.warn("Long polling failed, tried {} times, {} times left, will be suspended for a while! {}",
                                    time, retryTimes - time, e.getMessage());
                            ThreadUtils.sleep(TimeUnit.SECONDS, 5);
                            continue;
                        }
                        // print error, then suspended for a while.
                        log.error("Long polling failed, try again after 5 minutes!", e);
                        ThreadUtils.sleep(TimeUnit.MINUTES, 5);
                    }
                }
            }
            log.warn("Stop http long polling.");
        }
    }

 ```


>由上面的HttpLongPollingTask的run()，接着走到HttpSyncDataService的doLongPolling


> 上图解析：根据yml配置拼接接口地址 listenerUrl:http://localhost:9095/configs/listener，soul网关请求admin，如果获取到了ConfigGroupEnum，则再发起请求接口configs/fetch，获取admin更新的数据，然后刷新网关jvm缓存配置信息
* soul-web 网关请求 admin 的配置服务,读取超时时间为 90s，意味着网关层请求配置服务最多会等待 90s，这样便于 admin 配置服务及时响应变更数据，从而实现准实时推送


## soul-bootstrap请求admin的接口/configs/listener，具体做什么

* 接口configs/listener会请求到admin的doLongPolling方法
> http 请求到达 sou-admin 之后，并非立马响应数据，而是利用 Servlet3.0 的异步机制，异步响应数据

 ```Java   
  public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {

        // compare group md5
        // 因为soul-web可能未收到某个配置变更的通知，因此MD5值可能不一致，则立即响应
        List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
        String clientIp = getRemoteIp(request);

        // response immediately.
        if (CollectionUtils.isNotEmpty(changedGroup)) {
            this.generateResponse(response, changedGroup);
            log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
            return;
        }

        // listen for configuration changed.
        // Servlet3.0异步响应http请求
        final AsyncContext asyncContext = request.startAsync();

        // AsyncContext.settimeout() does not timeout properly, so you have to control it yourself
        asyncContext.setTimeout(0L);

        // block client's thread.
        scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
    }

 ```
* LongPollingClientd的run方法
> scheduler对象是ScheduledExecutorService定时任务线程池，60s之后执行响应接口configs/listener的请求
> clients.add(this)：clients是阻塞队列BlockingQueue<LongPollingClient>，保存了来自soul-web的请求信息，
 ```Java   
        @Override
        public void run() {
           // 加入定时任务，如果60s之内没有配置变更，则60s后执行，响应http请求
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                clients.remove(LongPollingClient.this);
                // clients是阻塞队列，保存了来自soul-web的请求信息
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            clients.add(this);
        }

 ```
>在这里我们可能有问题， 每次一个发起请求configs/listener，难道在admin后世60s后才响应数据吗（响应内容：告知是哪个 Group 的数据发生了变更我们将插件、规则、选择器、权限、元数据数据分成不同的组），
>这里的60S才响应数据，应对 while (RUNNING.get())这里它就不会无限循环向admin发起请求，然后60s后继续发起 http 请求，反复同样的请求
>60s还有疑惑问题：这样60s中间我们对admin操作很多配置信息，60s之后才响应，那网关在做代理转发时就严重滞后了，
>上面的疑惑请看下面的分析

* DataChangeTask核心类
> admin在对插件、规则、流量配置、用户配置更变时都会发布spring事件eventPublisher.publishEvent，最终在DataChangedEventDispatcher的onApplicationEvent监听处理，
>然后使用策略模式根据不同的类型处理不同的数据源,则我们来到HttpLongPollingDataChangedListener，下面贴了规则执行代码，其他插件、选择器、权限、元数据都类似
``` Java
    @Override
    protected void afterRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.RULE));
    }
 ```
> 如下是DataChangeTask类的核心代码
>遍历所有在阻塞队列的请求，然后当前请求移除阻塞队列，把修改的配置组消息响应回去（configs/listener），在这里解疑了前面60s的问题，有变更，就不会阻塞了，里面响应回去，没有才会在admin阻塞60s
 ```Java   
    class DataChangeTask implements Runnable {

        /**
         * The Group where the data has changed.
         */
        private final ConfigGroupEnum groupKey;

        /**
         * The Change time.
         */
        private final long changeTime = System.currentTimeMillis();

        /**
         * Instantiates a new Data change task.
         *
         * @param groupKey the group key
         */
        DataChangeTask(final ConfigGroupEnum groupKey) {
            this.groupKey = groupKey;
        }

        @Override
        public void run() {
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
                log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
            }
        }
    }

 ```


## 总结
*  soul-admin和soul-bootstrap需要同时开始http配置，才会生效
*  http长轮询同步: http的实现过程较为复杂，比较轻量，但是时效性低，其根据分组groupKey来拉取，如果数据量过大，过多，会有一定的影响。 也就是一个组groupKey下面的一个小地方更改，会拉取整个的组数据
*  从第八小节我们知道websocket知道，admin 会推送一次全量数据，后续如果配置数据发生变更，对比http根据分组拉取数据相对来说优化了些