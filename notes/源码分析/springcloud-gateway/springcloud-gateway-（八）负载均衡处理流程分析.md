# springcloud-gateway-（八）负载均衡处理流程分析

上一篇文章案例我们知道有eureka自动和手动配置微服务ip和端口来获取服务进行转发， 这篇我们深度解读原理各自原理
## 1 负载均衡分析
 *  基于前面的学习我们知道RoutePredicateHandlerMapping，作用相当于webmvc的handlermapping：将请求映射到对应的handler来处理。RoutePredicateHandlerMapping会遍历所有路由Route，并将获取到的route放入当前请求上下文的属性中
![图片: https://uploader.shimo.im/f/vLYUGVtZAlD8pL9r.png](https://img-blog.csdnimg.cn/20210225070724243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* this.routeLocator.getRoutes()，在配置的是基于服务发现的路由：spring.cloud.gateway.discovery.locator.enabled: true情况下，RouteDefinitionLocator实现类默认是从DiscoveryClientRouteDefinitionLocator获取路由列表，走到DiscoveryClientRouteDefinitionLocatord的getRouteDefinitions()方法，如下图
*  第127行能获取到路由配置：lb://order-service
![图片: https://uploader.shimo.im/f/V9MOBjhHm4G19oAO.png](https://img-blog.csdnimg.cn/2021022507073349.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* 接下来进入到FilteringWebHandler，FilteringWebHandler获取route的过滤器列表并转为过滤链，开始执行过滤器链 
* 进入到RouteToRequestUrlFilter，构造完整的负载均衡地址，例如route配置的中转服务是lb://order-service，请求的路径是/order/gateway，则构建后的地址是lb://order-service/order/gateway，将构建后的地址放入当前请求上下文中，继续下一个filter 
![图片: https://uploader.shimo.im/f/70FMdRt9ewD2tM1H.png](https://img-blog.csdnimg.cn/20210225070743536.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


* LoadBalancerClientFilter了，进入LoadBalancerClientFilter可以看到，首先获取scheme，如果不是lb，则直接往下一个filter传递；如果是lb，则选择服务节点构建成最终的中转地址 
![图片: https://uploader.shimo.im/f/qylWFq1ght1tcKsU.png](https://img-blog.csdnimg.cn/20210225070754887.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* 接下来重头戏：选择服务节点，先要先获取负载均衡策略器 
在上图的第83行代码
   loadBalancer.choose，因为我们配置开始Ribbon了，LoadBalancerClient有个实现类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225070829745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225070818296.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


## 2.dubug继续走
第54行serviceID值是：order-service
![图片: https://uploader.shimo.im/f/GK9O3pNJNeFCQPvv.png](https://img-blog.csdnimg.cn/20210225070838929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

## 3 继续到这
![图片: https://uploader.shimo.im/f/C9HVWCBYX9OBrWxs.png](https://img-blog.csdnimg.cn/20210225070844933.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

## 4  继续：先根据serviceId去spring-context里获取负载均衡策略器如果没有获取到，则自己初始化一个，继承类是：ZoneAwareLoadBalancer，默认负载均衡策略器
![图片: https://uploader.shimo.im/f/8wW7lev6fFytJQCA.png](https://img-blog.csdnimg.cn/20210225070853192.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

下面是：初始化一个，默认负载均衡策略器是DynamicServerListLoadBalancer源码
```Java
static <C> C instantiateWithConfig(AnnotationConfigApplicationContext context, Class<C> clazz, IClientConfig config) {
    Object result = null;

    try {
        Constructor<C> constructor = clazz.getConstructor(IClientConfig.class);
        result = constructor.newInstance(config);
    } catch (Throwable var5) {
    }

    if (result == null) {
        result = BeanUtils.instantiateClass(clazz);
        if (result instanceof IClientConfigAware) {
            ((IClientConfigAware)result).initWithNiwsConfig(config);
        }

        if (context != null) {
            context.getAutowireCapableBeanFactory().autowireBean(result);
        }
    }

    return result;
}
```
## 5  我们进到RibbonLoadBalancerClient类第54行this.getServer方法里面
![图片: https://uploader.shimo.im/f/HWqlOgI6q4JwKVDr.png](https://img-blog.csdnimg.cn/20210225070909281.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

//走进来this.getServer（）
```Java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

## 6   ZoneAwareLoadBalancer的chooseServer()
    下图第79行，由于我们只有一个区，然后执行else的代码块
![图片: https://uploader.shimo.im/f/J5Mj3nPJM7AfKPyx.png](https://img-blog.csdnimg.cn/2021022507092250.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

下图是this.getLoadBalancerStats()输出对象：
我们能看到服务名称：order-service和两个实例ip/端口
![图片: https://uploader.shimo.im/f/707VitOCFzitAPNK.png](https://img-blog.csdnimg.cn/20210225070932202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


![图片: https://uploader.shimo.im/f/dTF65zqp1SiTYD4r.png](https://img-blog.csdnimg.cn/20210225070939261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

## 7  继续跟踪AbstractServerPredicate的incrementAndGetModulo
![图片: https://uploader.shimo.im/f/L5BhPfWiAnJij2rL.png](https://img-blog.csdnimg.cn/20210225070950955.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


```Java
private int incrementAndGetModulo(int modulo) {
    int current;
    int next;
    do {
        current = this.nextIndex.get(); 
        next = (current + 1) % modulo;
    } while(!this.nextIndex.compareAndSet(current, next) || current >= modulo);

    return current;
   ```
**上段代码解析**：
 第5行： this.nextIndex对象是原子类：AtomicInteger，
 第6行：modulo是服务器集群的总数，轮训算法：第N次请求 % 服务器集群的总数 = 实际调用服务器位置的下标
 第7行： this.nextIndex.compareAndSet(current, next)使用线程安全原子性CAS操作，每次执行 成功加1，同时是使用了自旋锁（do   while），一开始我们没配置任何负载均衡策略，走到这段代码，自是验证默认使用轮训策略

## 8  拿到实例后初始化RibbonLoadBalancerClient对象
```Java
public ServiceInstance choose(String serviceId, Object hint) {
    Server server = this.getServer(this.getLoadBalancer(serviceId), hint);
    return server == null ? null : new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
}
```
到这里我们理清了在没有配置负载均衡策略时，是怎么获取具体哪个实例，拿到RibbonLoadBalancerClient，然后继续执行下一个filter。

最后对实际地址的转发在NettyRoutingFilter中 ，然后调用具体服务。
