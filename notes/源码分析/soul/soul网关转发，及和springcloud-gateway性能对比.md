# soul网关转发，及和springcloud-gateway性能对比.md

## soul网关转发
接着上篇内容，我们使用springboot接入soul
* 启动soul-bootstrap，soul-admin及order-server都在本地起的话，不需要修改配置，直接启动
* 启动两个order-server实例： 8098,8099
> 选择器列表:order-serve 点击修改进去，能看到两个实例信息
* post请求地址：http://localhost:9195/order-service/order/gateway，能拿到order-servcer返回的数据，说明soul网关转发成功



> 发现点：之前order-server配置的app-name：sb-demo-api启动，后面我又kill掉order-server，修改app-name：order-server，启动服务后，
过来三四分钟刷新soul-admin页面，选择器列表：sb-demo-api的信息还在，没有剔除掉，先把问题记录，后续跟源码看为什么？


## soul和springcloud-gateway性能对比
环境：win10 16g   惠普
压测工具：superbenchmarker
启动两个服务：localhost:8098,localhost:8099 ，


### soul-bootStrap 
执行命令
```
sb -u http://localhost:9195/order-service/order/gateway -c 20  -N 60

```

###springcloud-gateway
执行命令
```
 sb -u http://localhost:8080/order/gateway -c 20 -N 60  -m post

```


