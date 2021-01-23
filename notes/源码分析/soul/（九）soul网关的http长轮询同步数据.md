# （九）soul网关的http长轮询同步数据

##  目标
* 怎么开启http长轮训和初始化相应类
*



## soul-admin怎么开启http长轮训和初始化相应类
* 怎么开启http长轮训
>soul-admin的配置
 ``` Java   
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
  sync:
      http:
        enabled: true
 ```

>http长轮训初始化在DataSyncConfiguration的内部类，ZookeeperListener，NacosListener，WebsocketListener也是在DataSyncConfiguration初始化的


 ``` Java   
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
![afterPropertiesSet.png](../soul/png/afterPropertiesSet.png "afterPropertiesSet")

> AbstractDataChangedListener的afterPropertiesSet方法
 ``` Java   
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
 ``` Java   
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
 ``` Java   
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
 ``` Java   
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
            <version>${project.version}</version>
        </dependency>
 ```
*  HttpSyncDataService的初始化
>HttpSyncDataService需要注入HttpConfig，PluginDataSubscriber，MetaDataSubscriber，AuthDataSubscriber
 ``` Java   
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
 ``` Java   
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
>开始长时间轮询时，每个服务器都会创建一个线程来侦听更改
 ``` Java   
  this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
          new LinkedBlockingQueue<>(),
          SoulThreadFactory.create("http-long-polling", true));
  // start long polling, each server creates a thread to listen for changes.
  this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));

 ``` 

>execute执行HttpLongPollingTask的run()
 ``` Java   
  this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
          new LinkedBlockingQueue<>(),
          SoulThreadFactory.create("http-long-polling", true));
  // start long polling, each server creates a thread to listen for changes.
  this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));

 ``` 


>开始长时间轮询时，每个服务器都会创建一个线程来侦听更改
 ``` Java   
  this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
          new LinkedBlockingQueue<>(),
          SoulThreadFactory.create("http-long-polling", true));
  // start long polling, each server creates a thread to listen for changes.
  this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));

 ``` 
## 总结
*  soul-admin和soul-bootstrap需要同时开始http配置，才会生效
*
*