# （一）soul的入门使用

## 目标
 * 下载soul源码和编译
 * 启动soul-admin
 * 接入把springboot应用，接入soul-admin
 * 响应式编程


## 下载soul源码和编译

* github地址：https://github.com/dromara/soul.git，
```
git cloue  https://github.com/dromara/soul.git
```
* 最好fork一份到自己的github账户下，在IDEA上，右键>> git >> Repository >> Remotes>> 把自己的分支添加上去，
  以后最新的master代码，自己修改的就提交到自己的github账户去，源码处加注释修改，便于后期的复盘，学习
* 编译
  在本地的工程目录下调出dos窗口，出现如下的SUCCESS表示成功
```
D:\github\soul>mvn clean package install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true

[INFO] Installing D:\github\soul\soul-register-center\soul-register-server\soul-register-server-zookeeper\target\soul-register-server-zookeeper-2.2.1-sources.jar to D:\repository\org\dromara\soul-register-server-zookeeper\2.2.1\soul-register-server-zookeeper-2.2.1-sources.jar
[INFO] Installing D:\github\soul\soul-register-center\soul-register-server\soul-register-server-zookeeper\target\soul-register-server-zookeeper-2.2.1-sources.jar to D:\repository\org\dromara\soul-register-server-zookeeper\2.2.1\soul-register-server-zookeeper-2.2.1-sources.jar
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for soul 2.2.1:
[INFO]
[INFO] soul ............................................... SUCCESS [ 10.511 s]
[INFO] soul-common ........................................ SUCCESS [ 12.273 s]
[INFO] soul-admin ......................................... SUCCESS [ 27.272 s]
[INFO] soul-plugin ........................................ SUCCESS [  0.223 s]
[INFO] soul-plugin-api .................................... SUCCESS [  2.814 s]
[INFO] soul-sync-data-center .............................. SUCCESS [  0.286 s]
[INFO] soul-sync-data-api ................................. SUCCESS [  3.181 s]
[INFO] soul-plugin-global ................................. SUCCESS [  3.505 s]
[INFO] soul-spring-boot-starter ........................... SUCCESS [  0.306 s]
[INFO] soul-spring-boot-starter-plugin .................... SUCCESS [  0.446 s]
[INFO] soul-spring-boot-starter-plugin-global ............. SUCCESS [  6.835 s]
[INFO] soul-spi ........................................... SUCCESS [  2.628 s]
[INFO] soul-plugin-base ................................... SUCCESS [  5.202 s]
[INFO] soul-metrics ....................................... SUCCESS [  0.283 s]
[INFO] soul-metrics-spi ................................... SUCCESS [  3.326 s]
[INFO] soul-metrics-prometheus ............................ SUCCESS [  4.468 s]
[INFO] soul-metrics-facade ................................ SUCCESS [  3.197 s]
[INFO] soul-web ........................................... SUCCESS [  7.285 s]
[INFO] soul-spring-boot-starter-gateway ................... SUCCESS [  0.389 s]
[INFO] soul-plugin-divide ................................. SUCCESS [  6.045 s]
[INFO] soul-spring-boot-starter-plugin-divide ............. SUCCESS [  3.488 s]
[INFO] soul-plugin-alibaba-dubbo .......................... SUCCESS [  5.049 s]
[INFO] soul-spring-boot-starter-plugin-alibaba-dubbo ...... SUCCESS [  3.558 s]
[INFO] soul-plugin-apache-dubbo ........................... SUCCESS [  5.107 s]
[INFO] soul-spring-boot-starter-plugin-apache-dubbo ....... SUCCESS [  4.003 s]
[INFO] soul-plugin-httpclient ............................. SUCCESS [  4.695 s]
[INFO] soul-spring-boot-starter-plugin-httpclient ......... SUCCESS [  5.502 s]
[INFO] soul-plugin-springcloud ............................ SUCCESS [  4.308 s]
[INFO] soul-spring-boot-starter-plugin-springcloud ........ SUCCESS [  3.508 s]
[INFO] soul-plugin-hystrix ................................ SUCCESS [  3.619 s]
[INFO] soul-spring-boot-starter-plugin-hystrix ............ SUCCESS [  2.559 s]
[INFO] soul-plugin-monitor ................................ SUCCESS [  2.663 s]
[INFO] soul-spring-boot-starter-plugin-monitor ............ SUCCESS [  2.607 s]
[INFO] soul-plugin-ratelimiter ............................ SUCCESS [  4.467 s]
[INFO] soul-spring-boot-starter-plugin-ratelimiter ........ SUCCESS [  2.901 s]
[INFO] soul-plugin-sign ................................... SUCCESS [  3.692 s]
[INFO] soul-spring-boot-starter-plugin-sign ............... SUCCESS [  3.158 s]
[INFO] soul-plugin-waf .................................... SUCCESS [  3.521 s]
[INFO] soul-spring-boot-starter-plugin-waf ................ SUCCESS [  3.304 s]
[INFO] soul-plugin-rewrite ................................ SUCCESS [  2.925 s]
[INFO] soul-spring-boot-starter-plugin-rewrite ............ SUCCESS [  2.818 s]
[INFO] soul-plugin-sentinel ............................... SUCCESS [  3.279 s]
[INFO] soul-spring-boot-starter-plugin-sentinel ........... SUCCESS [  3.251 s]
[INFO] soul-plugin-sofa ................................... SUCCESS [ 10.609 s]
[INFO] soul-spring-boot-starter-plugin-sofa ............... SUCCESS [  2.688 s]
[INFO] soul-plugin-resilience4j ........................... SUCCESS [  4.407 s]
[INFO] soul-spring-boot-starter-plugin-resilience4j ....... SUCCESS [  2.722 s]
[INFO] soul-plugin-tars ................................... SUCCESS [  5.019 s]
[INFO] soul-spring-boot-starter-plugin-tars ............... SUCCESS [  2.677 s]
[INFO] soul-plugin-context-path ........................... SUCCESS [  2.929 s]
[INFO] soul-spring-boot-starter-plugin-context-path ....... SUCCESS [  2.589 s]
[INFO] soul-sync-data-zookeeper ........................... SUCCESS [  3.548 s]
[INFO] soul-spring-boot-starter-sync-data-center .......... SUCCESS [  0.286 s]
[INFO] soul-spring-boot-starter-sync-data-zookeeper ....... SUCCESS [  3.120 s]
[INFO] soul-sync-data-websocket ........................... SUCCESS [  3.393 s]
[INFO] soul-spring-boot-starter-sync-data-websocket ....... SUCCESS [  3.773 s]
[INFO] soul-sync-data-http ................................ SUCCESS [  3.701 s]
[INFO] soul-spring-boot-starter-sync-data-http ............ SUCCESS [  3.028 s]
[INFO] soul-sync-data-nacos ............................... SUCCESS [  3.196 s]
[INFO] soul-spring-boot-starter-sync-data-nacos ........... SUCCESS [  3.263 s]
[INFO] soul-client ........................................ SUCCESS [  0.182 s]
[INFO] soul-client-common ................................. SUCCESS [  2.531 s]
[INFO] soul-client-http ................................... SUCCESS [  0.192 s]
[INFO] soul-client-springmvc .............................. SUCCESS [  2.802 s]
[INFO] soul-spring-boot-starter-client .................... SUCCESS [  0.277 s]
[INFO] soul-spring-boot-starter-client-springmvc .......... SUCCESS [  2.597 s]
[INFO] soul-client-springcloud ............................ SUCCESS [  3.098 s]
[INFO] soul-spring-boot-starter-client-springcloud ........ SUCCESS [  2.520 s]
[INFO] soul-client-dubbo .................................. SUCCESS [  0.200 s]
[INFO] soul-client-dubbo-common ........................... SUCCESS [  2.563 s]
[INFO] soul-client-alibaba-dubbo .......................... SUCCESS [  3.065 s]
[INFO] soul-spring-boot-starter-client-alibaba-dubbo ...... SUCCESS [  2.738 s]
[INFO] soul-client-apache-dubbo ........................... SUCCESS [  3.107 s]
[INFO] soul-spring-boot-starter-client-apache-dubbo ....... SUCCESS [  3.139 s]
[INFO] soul-client-sofa ................................... SUCCESS [  3.784 s]
[INFO] soul-spring-boot-starter-client-sofa ............... SUCCESS [  3.177 s]
[INFO] soul-client-tars ................................... SUCCESS [  3.394 s]
[INFO] soul-spring-boot-starter-client-tars ............... SUCCESS [  3.070 s]
[INFO] soul-bootstrap ..................................... SUCCESS [  4.603 s]
[INFO] soul-dist .......................................... SUCCESS [ 15.974 s]
[INFO] soul-register-center ............................... SUCCESS [  0.185 s]
[INFO] soul-register-common ............................... SUCCESS [  2.186 s]
[INFO] soul-register-client ............................... SUCCESS [  0.217 s]
[INFO] soul-register-client-api ........................... SUCCESS [  2.306 s]
[INFO] soul-register-client-http .......................... SUCCESS [  2.607 s]
[INFO] soul-register-client-zookeeper ..................... SUCCESS [  6.012 s]
[INFO] soul-register-server ............................... SUCCESS [  0.504 s]
[INFO] soul-register-server-api ........................... SUCCESS [  3.027 s]
[INFO] soul-register-server-http .......................... SUCCESS [  3.370 s]
[INFO] soul-register-server-zookeeper ..................... SUCCESS [  3.890 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  05:36 min
[INFO] Finished at: 2021-01-13T23:32:40+08:00
[INFO] ------------------------------------------------------------------------

D:\github\soul>

```

## 启动soul-admin
* 项目导入IDEA，使用jdk1.8
* 修改soul-admin的application.yml
  url，root，password三个即可，然后启动项目SoulAdminBootstrap，Soul-Admin 会自动创建数据库，以及表结构，并初始化默认数据

```
spring:
  #profiles:
  #  active: h2
  thymeleaf:
    cache: true
    encoding: utf-8
    enabled: true
    prefix: classpath:/static/
    suffix: .html
  datasource:
    url: jdbc:mysql://114.67.171.251:3341/soul?useUnicode=true&characterEncoding=utf-8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
```
* 项目导入IDEA，使用jdk1.8


## 进入soul-admin页面

* 访问界面：http://localhost:9095/index.html#/home，点击看看里面有什么组件
* sawgger界面：http://localhost:9095/swagger-ui.html
*



##  接入把springboot应用，接入soul-admin
* 添加依赖
```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
            <version>2.2.1<</version>
        </dependency>
```
* 添加配置
> 注意：context-path和port要修改，不要soul-admin选择器列表找不到
```
soul:
  # Soul 针对 SpringMVC 的配置项，对应 SoulHttpConfig 配置类
  http:
    admin-url: http://127.0.0.1:9095 # Soul Admin 地址
    context-path: /order-service # 设置在 Soul 网关的路由前缀，例如说 /order、/product 等等。
    # 后续，网关会根据该 context-path 来进行路由
    app-name: order-service # 应用名。未配置情况下，默认使用 `spring.application.name` 配置项
    port: 8098 #你本项目的启动端口
    full: false   # 设置true 代表代理你的整个服务，false表示代理你其中某几个controller
```
* 代码添加注解：    
@SoulSpringMvcClient 注解一共有三个属性：
path：映射的 HTTP 接口的请求路径。
desc：接口的描述，便于知道其用途。
enable：是否开启，默认为 true 开启。
```
    @PostMapping(value = "/gateway")
    @ApiOperation(value = "网管测试")
    @SoulSpringMvcClient(path = "/order/gateway", desc = "获得用户详细")
    public String testGateway(HttpServletRequest request){
        final long start = System.currentTimeMillis();
        //orderService.batchInsertOrser();
        String url = request.getRequestURL().toString();//获得客户端发送请求的完整url
        System.out.println("获得客户端发送请求的完整url"+url);
        return "网关测试";
    }
```
*  在sooul-admin查看
在http://localhost:9095/index.html#/plug/divide，界面可以看到如下信息

```
规则名称                       开启           更新时间                 操作
/sb-demo-api/order/gateway	  开启           2021-01-14 00:32:49	修改删除
/sb-demo-api/user/get	      开启           2021-01-14 00:32:49	修改删除
```


到这里我们算springboot接入成功了


## 响应式编程
* 举例了一个例子：我们乘坐同一条线的地铁类比初始化响应式编程（Flux和Mono），每趟车类比数据元素，
  我们乘坐地铁的人不需要关心上下趟车次会发生撞车问题，如果前面一趟车发生事故，或者开慢了，前面的那趟车次会发通知告知后面的车次，从而减慢速度，
  因为每次车之间依次存在相互感应/通知,上一趟车发生事故会及时发出信息告知下一趟车,再传递到后面的地铁车次。响应式编程就是这个逻辑 
* 相关学习文章：
>>https://juejin.cn/post/6844903858599428110
>>https://segmentfault.com/a/1190000024499748                                  
