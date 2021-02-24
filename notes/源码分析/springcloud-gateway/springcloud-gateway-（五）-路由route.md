# 路由route 
## 1.1  路由定义定位器RouteDefinitionLocator
在前面的分析GatewayAutoConfiguration类会初始化RouteDefinitionLocator，
```Java
@Bean
@Primary
public RouteDefinitionLocator routeDefinitionLocator(
      List<RouteDefinitionLocator> routeDefinitionLocators) {
   return new CompositeRouteDefinitionLocator(
         Flux.fromIterable(routeDefinitionLocators));
}
```
用下图重新梳理下 ，接口有四种图片: 图片: ![https://uploader.shimo.im/f/AfrBLnv7auI9xTYf.png](https://img-blog.csdnimg.cn/20210225064531652.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

PropertiesRouteDefinitionLocator ，从配置文件( 例如，YML / Properties 等 ) 读取
GatewayProperties的getRoutes（第73行）获取yml配置信息
![图片: https://uploader.shimo.im/f/GEgLPKSIQAZ7njaP.png](https://img-blog.csdnimg.cn/20210225064543341.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

RouteDefinitionLocator对象内容是yml配置gateway的信息
![图片: https://uploader.shimo.im/f/J90yqElmwvAbmN1C.png
图片: https://uploader.shimo.im/f/FwFSVVbqCP196D2x.png](https://img-blog.csdnimg.cn/20210225064552986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225064601117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


RouteDefinitionRepository ，从存储器( 例如，内存 / Redis / MySQL 等 )读取
RouteDefinitionRepository 具体实现类InMemoryRouteDefinitionRepository（实现了基于内存为存储器）
源码下图解读：
GatewayWebfluxEndpoint是个controller，从外部调用接口/routes/{id}，在第138行代码，在save方法最后这个类InMemoryRouteDefinitionRepository保存到线程安全的map中 
>Map<String, RouteDefinition> routes = synchronizedMap(new LinkedHashMap<String, RouteDefinition>());
>
![图片: https://uploader.shimo.im/f/yS0dCVPp1QEz4FtP.png](https://img-blog.csdnimg.cn/20210225064622549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

下图是具体的save方法源码
![图片: https://uploader.shimo.im/f/vTCmpuxrSCppXnP9.png](https://img-blog.csdnimg.cn/2021022506462983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

思考：使用 InMemoryRouteDefinitionRepository 来维护 RouteDefinition 信息，在网关实例重启或者崩溃后，数据就没有了，此时我们可以实现 RouteDefinitionRepository 接口，以实现例如 MySQLRouteDefinitionRepository 。通过类似 MySQL 等持久化、可共享的存储器，也可以带来 Spring Cloud Gateway 实例集群获得一致的、相同的 RouteDefinition 信息

DiscoveryClientRouteDefinitionLocator ，从注册中心( 例如，Eureka / Consul / Zookeeper / Etcd 等 )读取
下图解读：
serviceInstances 注册发现客户端，用于向注册中心发起请求
第124行开始，设置RouteDefinition的属性值处理
![图片: https://uploader.shimo.im/f/04VmO7yud2ypZJ9e.png](https://img-blog.csdnimg.cn/20210225064648818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


CompositeRouteDefinitionLocator ，组合多种 RouteDefinitionLocator 的实现，为 RouteDefinitionRouteLocator 提供统一入口。在 本文 详细解析。
      另外，CachingRouteDefinitionLocator 也是 RouteDefinitionLocator 的实现类，已经被       CachingRouteLocator 取代。
##  1.2 路由定位器RouteLocator 
 GatewayAutoConfiguration中初始化RouteLocator对象的代码块
 ```Java
@Bean
public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
      List<GatewayFilterFactory> gatewayFilters,
      List<RoutePredicateFactory> predicates,
      RouteDefinitionLocator routeDefinitionLocator,
      ConfigurationService configurationService) {
   //routeDefinitionLocator 前面初始化的
   return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates,
         gatewayFilters, properties, configurationService);
}
```

下图是：RouteDefinitionRouteLocator初始化代码
可以断点在第34行看对象信息
![图片: https://uploader.shimo.im/f/wQoXFff69TcYFywW.png](https://img-blog.csdnimg.cn/20210225064705882.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


RouteLocator 的多个实现类
![图片: https://uploader.shimo.im/f/pJLqbFcVuFT8C1E3.png](https://img-blog.csdnimg.cn/20210225064724318.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

CompositeRouteLocator，组合多种 RouteLocator 的实现类

CachingRouteLocator
缓存路由的 RouteLocator 实现类 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225064735612.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
![图片: https://uploader.shimo.im/f/voId8YihNXQs5sTe.png](https://img-blog.csdnimg.cn/20210225064742354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)



RouteDefinitionRouteLocator
##  1.3 RouteDefinition
五个属性，还有一个通过 text解析 创建 RouteDefinition
下图是：四个属性的处理顺序
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021022506480135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

```Java
/**
 * 属性，ID 编号，唯一。
 */
private String id;
/**
 * predicates 属性，谓语定义数组。请求通过 predicates 判断是否匹配。在 Route 里，PredicateDefinition 转换成 Predicate
 */
@NotEmpty
@Valid
private List<PredicateDefinition> predicates = new ArrayList<>();
/**
 * filters 属性，过滤器定义数组。在 Route 里，FilterDefinition 转换成 GatewayFilter
 */
@Valid
private List<FilterDefinition> filters = new ArrayList<>();
/**
 * uri 属性，路由向的 URI 。
 */
@NotNull
private URI uri;
private Map<String, Object> metadata = new HashMap<>();
/**
 * order 属性，顺序。当请求匹配到多个路由时，使用顺序小的。
 */
private int order = 0;
public RouteDefinition() {
  
/**
 * 根据 text 创建 RouteDefinition
 * @param text 格式 ${id}=${uri},${predicates[0]},${predicates[1]}...${predicates[n]}
 *             例如 route001=http://127.0.0.1,Host=**.addrequestparameter.org,Path=/get
 */
public RouteDefinition(String text) {
   int eqIdx = text.indexOf('=');
   if (eqIdx <= 0) {
      throw new ValidationException("Unable to parse RouteDefinition text '" + text
            + "'" + ", must be of the form name=value");
   }

   setId(text.substring(0, eqIdx));

   String[] args = tokenizeToStringArray(text.substring(eqIdx + 1), ",");

   setUri(URI.create(args[0]));

   for (int i = 1; i < args.length; i++) {
      this.predicates.add(new PredicateDefinition(args[i]));
   }
}

```
![图片: https://uploader.shimo.im/f/HWVPpJI3IKmCvc0E.png](https://img-blog.csdnimg.cn/20210225064808950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

