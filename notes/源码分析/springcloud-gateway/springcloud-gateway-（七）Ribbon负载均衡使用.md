
# springcloud-gateway-（七）Ribbon负载均衡使用
# 1 如何使用
添加依赖
```Java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

提示： spring-cloud-starter-netflix-eureka-client包含spring-cloud-starter-netflix-ribbon包，如果下面的方式二配置，添加依赖spring-cloud-starter-netflix-ribbon即可；
```Java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
```
![图片: https://uploader.shimo.im/f/yhKkg1fshlNb9QFC.png](https://img-blog.csdnimg.cn/20210225065815943.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)





* 方式一：通过eureka动态获取服务注册与发现
yml配置：
 ```Java
spring:
  application:
    name: springcloud-gateway
  cloud:
    gateway:
      routes:
      - id: order
        uri:  lb://order-service
        predicates:
         - Path=/order/**
logging:
  level:
    io.github.kimmking: ERROR
    org: ERROR

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```
解析：第8行的order-service是服务注册在eureka的服务名称，多个order-service实例时默认使用轮训策略转发到其他一个实例；
案例：启动两个order-service实例，post发起六次请求：http://localhost:9527/order/gateway ，
![图片: https://uploader.shimo.im/f/ubwgZ8FFgkC0QQgE.png](https://img-blog.csdnimg.cn/20210225065829196.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


![图片: https://uploader.shimo.im/f/TKmg8Yi1SUCgaput.png](https://img-blog.csdnimg.cn/20210225065834356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


* 方式二：手动配置的微服务的服务器ip端口
yml配置：
 ```Java
spring:
  application:
    name: springcloud-gateway
  cloud:
    gateway:
      routes:
      - id: order
        uri:  lb://order-service
        predicates:
         - Path=/order/**
ribbon:
  eureka:
    enabled: false //必须要关闭eureka的服务注册与发现
order-service:
  ribbon:
    listOfServers: localhost:8090, localhost:8091,localhost:8092
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
 ```

>解析：
>第8行的lb表示匹配规则为order-service负载均衡，规则名为order-service。
> 第14行表示该负载均衡规则采用Ribbon，后面就是Ribbon的配置。
>第16行的listOfServer：配置的微服务的服务器ip端口，由于我不需要去动态获取服务                        注册与发现，因此可以在此写死。需要注意的就是必须要把ribbon.eureka.enabled置为false，                                                         
> 第17行NFLoadBalancerRuleClassName：使用的负载均衡策略，RoundRobinRule是轮询的负载均衡策略，有提供7种策略，也可自定义负载均衡策略，具体分享各自策略请查看后续文章。
> 
![图片: https://uploader.shimo.im/f/hQXbnamr4XAyLMt6.png](https://img-blog.csdnimg.cn/20210225065843797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

