# (十）soul网关的zookeeper同步数据

##  目标
* soul-admin怎么开启zookeeper和初始化相应类
* soul-bootstrap端怎么获取数据
* 官方提供的流程图




## soul-admin怎么开启zookeeper和初始化相应类
* 首先本地需要启动zookeeper服务

* soul-admin的配置，要把websocket注释掉
 ```Java   
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
  sync:
    zookeeper:
#    websocket:
#      enabled: true
      url: localhost:2181
      sessionTimeout: 5000
      connectionTimeout: 2000
 ```

* DataSyncConfiguration的内部类，ZookeeperListener初始化，ZookeeperDataChangedListener和ZookeeperDataInit都在这里初始化    
```Java   
    /**
     * The type Zookeeper listener.
     */
    @Configuration
    @ConditionalOnProperty(prefix = "soul.sync.zookeeper", name = "url")
    @Import(ZookeeperConfiguration.class)
    static class ZookeeperListener {

        /**
         * Config event listener data changed listener.
         *
         * @param zkClient the zk client
         * @return the data changed listener
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataChangedListener.class)
        public DataChangedListener zookeeperDataChangedListener(final ZkClient zkClient) {
            return new ZookeeperDataChangedListener(zkClient);
        }

        /**
         * Zookeeper data init zookeeper data init.
         *
         * @param zkClient        the zk client
         * @param syncDataService the sync data service
         * @return the zookeeper data init
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataInit.class)
        public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
            return new ZookeeperDataInit(zkClient, syncDataService);
        }
    }
```
>ZookeeperDataChangedListener和WebsocketDataChangedListener类似，同样实现了 DataChangedListener 接口，用于具体的事件监听，并处理数据，存在zkclient成员变量，通过zkclient进行操作数据
>ZookeeperDataInit实现了 CommandLineRunner 接口，项目启动后执行run方法：项目启动时把所以配置数据刷新到zookeeper上，

``` java
    @Override
    public void run(final String... args) {
        String pluginPath = ZkPathConstants.PLUGIN_PARENT;
        String authPath = ZkPathConstants.APP_AUTH_PARENT;
        String metaDataPath = ZkPathConstants.META_DATA;
        if (!zkClient.exists(pluginPath) && !zkClient.exists(authPath) && !zkClient.exists(metaDataPath)) {
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }
    
//SyncDataServiceImpl的syncAll,
    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
        appAuthService.syncData();
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        List<SelectorData> selectorDataList = selectorService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
        List<RuleData> ruleDataList = ruleService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
        metaDataService.syncData();
        return true;
    }


```
>通过spring发布事件，调用不同的具体的事件监听并处理数据 （spring发布事件和监听在第七小节有详细讲解）
>我们在admin的客户端修改了插件、选择器、规则、权限等通过spring事件监听来到这里ZookeeperDataChangedListener
>如下只贴了规则修改的代码
``` java
    @Override
    public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
        if (eventType == DataEventTypeEnum.REFRESH) {
            final String selectorParentPath = ZkPathConstants.buildRuleParentPath(changed.get(0).getPluginName());
            deleteZkPathRecursive(selectorParentPath);
        }
        for (RuleData data : changed) {
            final String ruleRealPath = ZkPathConstants.buildRulePath(data.getPluginName(), data.getSelectorId(), data.getId());
            if (eventType == DataEventTypeEnum.DELETE) {
                deleteZkPath(ruleRealPath);
                continue;
            }
            final String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getPluginName());
            createZkNode(ruleParentPath);
            //create or update
            upsertZkNode(ruleRealPath, data);
        }
    }

//createZkNode，upsertZkNode，deleteZkPath等zookeeper的操作
    private void upsertZkNode(final String path, final Object data) {
        if (!zkClient.exists(path)) {
            zkClient.createPersistent(path, true);
        }
        zkClient.writeData(path, data);
    }
```
* 下图是项目初始化后，在zookeeper创建的节点数据 
![在这里插入图片描述](https://img-blog.csdnimg.cn/202101250032423.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> ZookeeperListener的 @Import(ZookeeperConfiguration.class) ,ZookeeperConfiguration里面会初始化ZkClient，使用ZKClient这个Zookeeper客户端
>ZKClient存在api：1.创建会话
 2.创建节点
 3.读取数据
 4.更新数据
 5.删除节点
 6.检查节点是否存在

``` java
   @Bean
     @ConditionalOnMissingBean(ZkClient.class)
     public ZkClient zkClient(final ZookeeperProperties zookeeperProp) {
         return new ZkClient(zookeeperProp.getUrl(), zookeeperProp.getSessionTimeout(), zookeeperProp.getConnectionTimeout());
     }
```
## soul-bootstrap端怎么获取数据
* yml配置
 ```Java   
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
        zookeeper:
             url: localhost:2181
             sessionTimeout: 5000
             connectionTimeout: 2000
 ```
* 依赖
 ```Java   
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
        <version>${project.version}</version>
    </dependency>
 ```
* 初始化相应类
>根据配置信息，在ZookeeperSyncDataConfiguration初始化ZookeeperSyncDataService和ZkClient
 ```Java   
    @Bean
     public SyncDataService syncDataService(final ObjectProvider<ZkClient> zkClient, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                            final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
         log.info("you use zookeeper sync soul data.......");
         return new ZookeeperSyncDataService(zkClient.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                 metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
     }
 ```
* ZookeeperSyncDataService：对数据
> 构造函数：watcherData():刷新本地jvm缓存，其他两个方式类似
> watch机制，监听zookeeper数据的变化

 ```Java   
    public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        watcherData();
        watchAppAuth();
        watchMetaData();
    }

    private void watcherData() {
        final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
        // 获取插件列表，启动监听（在其中同时进行插件数据、选择器、规则的初始化）
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        for (String pluginName : pluginZKs) {
            watcherAll(pluginName);
        }
        
        // 启动监听，数据更新（新增和修改），对更新的插件进行监听
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    watcherAll(pluginName);
                }
            }
        });
    }

//watch机制，监听数据的变化
    private void watcherAll(final String pluginName) {
        watcherPlugin(pluginName);
        watcherSelector(pluginName);
        watcherRule(pluginName);
    }

 ```
## 官方提供的流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210125003406253.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210125003406249.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210125003406207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


## 总结
* zookeeper主要是依赖 watch 机制，soul-web 会监听配置的节点，soul-admin 在启动的时候，会将数据全量写入 zookeeper，后续数据发生变更时，会增量更新 zookeeper 的节点，与此同时，soul-web 会监听配置信息的节点，一旦有信息变更时，会更新本地缓存
* zookeeper相对http长轮训简单好理解，不是分组更新数据，
