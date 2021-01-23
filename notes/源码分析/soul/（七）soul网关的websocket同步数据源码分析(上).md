# （七）soul网关的websocket同步数据源码分析(上)

##  目标
* SoulClientController的内容
* WebsocketDataChangedListener分析

> 如下时soul整体架构图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123182931181.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> 如下时soul-admin的数据同步流程图 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123182940990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


## SoulClientController的内容

*  1：一共六个接口：springmvc-register，springcloud-register，springcloud-register，dubbo-register，sofa-register，tars-register
    
*  2：六个接口全部用于各个插件通过okhttp方式调用接口：
*  3:处理被代理服务器的注册信息新增到： meta_data表
    
 ```Java   
            MetaDataDO metaDataDO = MetaDataDO.builder()
                    .appName(dto.getAppName())
                    .path(dto.getContext() + "/**")
                    .pathDesc(dto.getAppName() + "spring cloud meta data info")
                    .serviceName(dto.getAppName())
                    .methodName(dto.getContext())
                    .rpcType(dto.getRpcType())
                    .enabled(dto.isEnabled())
                    .id(UUIDUtils.getInstance().generateShortUuid())
                    .dateCreated(currentTime)
                    .dateUpdated(currentTime)
                    .build();
            metaDataMapper.insert(metaDataDO);

```

* 4: 通过发布spring事件


``` Java
 eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.META_DATA, DataEventTypeEnum.CREATE,
                Collections.singletonList(MetaDataTransfer.INSTANCE.mapToData(metaDataDO))));    

```

*  如下5个controller的update和insert之后最终发布DataChangedEvent 事件
    * MetaDataController
    * PluginController
    * SelectorController
    * RuleController
    * SoulClientController，
![](https://img-blog.csdnimg.cn/20210123183001805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)
> 都会发送一个数据变更事件DataChangedEvent 事件，最终调用publishEvent方法，发布spring事件出去


* 5: spring事件监具体处理代码
>使用了策略设计模式：通过event.getGroupKey()属性值配比ConfigGroupEnum，同时根据yml的配置信息（websocket，zookeeoer，necos）初始化的具体DataChangedListener，默认是：WebsocketDataChangedListener，： APP_AUTH 鉴权,PLUGIN 插件,RULE 规则,META_DATA 元数据都在onApplicationEvent方法接受各自spring事件的监听
  ```Java  
        public void onApplicationEvent(final DataChangedEvent event) {
            for (DataChangedListener listener : listeners) {
                switch (event.getGroupKey()) {
                    case APP_AUTH:
                        listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                        break;
                    case PLUGIN:
                        listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                        break;
                    case RULE:
                        listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                        break;
                    case SELECTOR:
                        listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                        break;
                    case META_DATA:
                        listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                        break;
                    default:
                        throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
                }
            }
        }

```

* 6: 事件监听接口DataChangedListener的实现类，有四个，如下图，今天我们分析默认的初始化事件监听实现类：WebsocketDataChangedListener
>WebsocketDataChangedListener
>HttpLongPollingDataChangedListener
>NacosDataChangedListener
>ZookeeperDataChangedListener
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123183101362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


## WebsocketDataChangedListener分析

* soul-admin 开启websocket,在onApplicationEvent方法处理时就会调到WebsocketDataChangedListener类  
、
```Java  
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
  sync:
    websocket:
      enabled: true

  ```
* WebsocketDataChangedListener类
> 把数据转化为json，通过WebsocketCollector发送出去
>
```Java  
       WebsocketData<MetaData> configData =
                 new WebsocketData<>(ConfigGroupEnum.META_DATA.name(), eventType.name(), metaDataList);
         WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
```

##  总结
> 充分使用了Spring的容器内事件发布监听机制： ApplicationEventPublisher发布， WebsocketDataChangedListener监听eventType);
> 发布-订阅模式：单一职责原则、代码耦合降低、事件的处理人员只需关注处理的代码、发布人员只需关注发布的代码