# springcloud-gateway-（三）如何处理一个请求
 ## gateway是如何处理一个请求
>例：post请求http://localhost:9527/order/gateway 
>       最后调用http://10.201.35.189:8090/order/gateway
 
路由比配：
打断点到RoutePredicateHandlerMapping的lookupRoute，
循环每个路由，看看predicate是否匹配，一直到找到匹配的路由，这里是默认的default_path_to_httpbin
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183106714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
 
 
 
debug的调用栈，目前找到这里熟悉的类DispatcherHandler，依次找到FilteringWebHandler，handle方法的大致逻辑
：从 GATEWAY_ROUTE_ATTR 获得 请求对应的 Route 。
：获得 GatewayFilter 数组，包含 route.filters 和 globalFilters 。
：排序获得的 GatewayFilter 数组。
：使用获得的 GatewayFilter 数组创建 DefaultGatewayFilterChain ，过滤处理请求。
下图：FilteringWebHandler 的handler方法上打上断点，可以看到 从route拿到两个filter，合并到之前的10个filter，一起来处理过滤 exchange
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183117248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
 
再走到下图的代码
 
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183121394.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

跟踪找这：
FilteringWebHandler
核心处理方法 ，使用调用链机制，
自定义的HeadersFilter 的order=0
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183128465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

NettyRoutingFilter（GlobalFilter的实现类）的方法filter，
filtered.forEach(httpHeaders::set);    headers信息设置
 ```Java 
//核心调用方法
Mono<HttpClientResponse> responseMono = this.httpClient.request(method, url, req -> {
   final HttpClientRequest proxyRequest = req.options(NettyPipeline.SendOptions::flushOnEach)
         .headers(httpHeaders)
         .chunkedTransfer(chunkedTransfer)
         .failOnServerError(false)
         .failOnClientError(false)
 ```
从bug的看到url是需要实际调用的服务地址，
 
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183155947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

接着处理后面的filter，NettyRoutingFilter 用httpclient连接池对数据的转发
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183207280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

 
总体逻辑还需要在整理，有很多不明白的
1：具体怎么路由细节还不清楚
2：fiters一共12个，按照order从小到大一次执行，具体每个filter做什么还没去细看，只能通过类名猜想假设 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183218691.png)

3 链式调用机制关联学习AOP的拦截器链调用：
链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成  返回以后再来执行，拦截器链的机制，保证通知方法与目标方法的执行顺序，运行顺序遵循“后进先出”的原则
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183235209.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

例：AspectJAfterThrowingAdvice 对应@AfterThrowing注解，如下图：在目标方法出现异常以后，才会走到invokeAdviceMethod方法，
思考：所以使用@AfterThrowing时，想要注解生效，自己的业务代码不要吃掉异常，异常必须抛出来
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183241780.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

•	AspectJAfterThrowingAdvice 对应后置通知(@After)
在目标方法运行结束之后运行（无论方法正常结束还是异常结束）
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183248127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

•	 mi.proceed() 这个方法源码如下图
解析：当前index值和增强器的数量相同执行真实目标方法，不同则执行增强器的方法，同时index+1，此处猜想index的初始值是：0
 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210220183252694.png)


