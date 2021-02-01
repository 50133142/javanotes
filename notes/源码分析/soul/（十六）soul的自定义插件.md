# （（十六）soul的自定义插件

##  目标
* soul的本身提供的插件

## soul的本身提供的插件
###  说明
* 插件是 soul 网关的核心执行者，每个插件在开启的情况下，都会对匹配的流量，进行自己的处理。

* 在soul 网关里面，插件其实分为2 类：

    * 一类是单一职责的调用链，不能对流量进行自定义的筛选。

    * 另一类，能对匹配的流量，执行自己的职责调用链。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201065259295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

### 插件类别
### 单一职责原插件
> 首先需要实现org.dromara.soul.plugin.api.SoulPlugin
 ```Java   
    Mono<Void> execute(ServerWebExchange exchange, SoulPluginChain chain);

    int getOrder();

    default String named() {
        return "";
    }
    default Boolean skip(ServerWebExchange exchange) {
        return false;
    }
 ```
* 接口方法详细说明
    * execute() 方法为核心的执行方法，用户可以在里面自由的实现自己想要的功能。 
    * getOrder() 指定插件的排序。    
    * named() 指定插件的名称。    
    * skip() 在特定的条件下，该插件是否被跳过。
    
### 比配流程处理插件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201071237144.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

> 首先需要继承org.dromara.soul.plugin.base.AbstractSoulPlugin
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201071445766.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* 详细讲解：
    * 继承该类的插件，插件会进行选择器规则匹配，那如何来设置呢？   
    * 首先在 soul-admin 后台 –>插件管理中，新增一个插件，注意 名称与 你自定义插件的 named() 方法要一致。   
    * 重新登陆 soul-admin 后台 ，你会发现在插件列表就多了一个你刚新增的插件，你就可以进行选择器规则匹配   
    * 在规则中，有个 handler 字段，是你自定义处理数据，在 doExecute() 方法中，通过 final String ruleHandle = rule.getHandle(); 获取，然后进行你的操作。


## 自定义单一职责插件

>由于单一职责插件不需要流量控制，所以不需要插件开关，只在请求的执行链路里做一些处理，比如GlobalPlugin插件根据原始请求信息构造soulContext对象，WebClientPlugin插件向真实业务实例发送http请求等。
* 我们设计实现一个接口环绕打印日志的插件

*  新建 soul-plugin-around 模块
在soul-plugin模块下建立子模块 soul-plugin-around
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201074104224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* soul-plugin-around的pom.xml信息
> 关键把soul-plugin-api添加依赖
 ```Java   
    <parent>
        <artifactId>soul-plugin</artifactId>
        <groupId>org.dromara</groupId>
        <version>2.2.1</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>soul-plugin-around-log</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-plugin-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-common</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
 ```

* AroundLogPlugin具体实现
> 执行execute方法，把时间设置到上下文中，然后打印开始日志，然后通过Mono.then方法，打印结束日志，通过exchange上下文统计链路执行耗时。
 ```Java   
   @Override
    public String named() {
        return PluginEnum.AROUNDLOG.getName();
    }

    @Override
    public Mono<Void> execute(ServerWebExchange exchange,SoulPluginChain chain){
        exchange.getAttributes().put(START_TIME, System.currentTimeMillis());

        log.info("AroundPlugin start executing...");

        return chain.execute(exchange)
                .then(Mono.defer(() -> {
                    Long startTime = exchange.getAttribute(START_TIME);
                    log.info("AroundLogPlugin end. Total execution time: {} ms", System.currentTimeMillis() - startTime);
                    return Mono.empty();
                }));
    }

    @Override
        return PluginEnum.AROUNDLOG.getCode();
    public int getOrder() {
    }
 ```


## 总结
*  
