# （十二）sould的divide插件底层原理，负载均衡及ip端口探活

##  目标
* divide插件底层原理
*




## divide插件底层原理
>  divide的具体使用使用在小节一里有分析了，这里不解释
>
 ```Java   
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
      websocket :
        urls: ws://localhost:9095/websocket,ws://localhost:9096/websocket,ws://localhost:9097/websocket

 ```

## 总结
*  注意网关的websocket的urls配置，其他单节点网关配置都差不多
