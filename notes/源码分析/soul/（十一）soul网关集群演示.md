# （十一）soul网关集群演示

##  目标
* 启动三个soul-admin和两个网关，两个order-service，一个eureka
*
* 



## 启动三个soul-admin和两个网关，两个order-service，一个eureka

* 启动三个soul-admin和两个网关（网关需要配置eureka信息） ，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210127003735183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)



> 注意网关的websocket的urls配置：ws://localhost:9095/websocket,ws://localhost:9096/websocket,ws://localhost:9097/websocket因为我们启动了三个admin实例，其中一个admin的配置信息更新操作，数据都需要及时同步到soul-bootstrap
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
>多个soul-admin实例时，初始化多个WebSocketClient，onOpen和onMessage接收websocket发了来的数据，这就也就保证了，不同admin修改配置时，都能及时刷新soul-bootstrap的jvm缓存信息
>相应的代码，在类WebsocketSyncDataService，
 ```Java   
    public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        //多个soul-admin实例，初始化多个WebSocketClient，onOpen和onMessage接收websocket发了来的数据

        for (String url : urls) {
            try {
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
            //每个soul-admin实例，存在一个调度线程，去进行断线重连
            for (WebSocketClient client : clients) {
                //进行连接
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
                //使用调度线程池进行断线重连，30秒进行一次
                executor.scheduleAtFixedRate(() -> {
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }

 ```

> 我们在9095服务修改选择器匹配规则，去9096,9097都能查看到9095时修改的配置信息
> 这次引起思考问题：admin集群下，各实例配置信息是怎同步的？ 这个不是问题，三个admin都是公用给，mysql数据库的，在任何一个客户端修改，其他客户端随时能看到
>
* 两个order-service，一个eureka
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210127003810999.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> http://localhost:9196/order-service/order/gateway  和http://localhost:9195/order-service/order/gateway ，请求网关服务，我们都能拿到order-service响应的数据
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-sigMRtP1-1611679030192)(../soul/png/9196.png "9196")]

## 本地安装Nginx
> 添加两处配置就行：新增集群配置和新增代理路径，如下是nginx的配置信息

 ```Java   
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
	
	#---------------1.新增集群配置-------------------------------
	upstream soul {
	 # 可以通过修改weight=10的值来设置权重
	  server  127.0.0.1:9195 weight=10;
	  server  127.0.0.1:9196 weight=10;
	}

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
		
	

        location / {
			  #---------------2.新增代理路径-------------------------------
			#代理路径和集群名称(upstream soul{})需要保持一致
			proxy_pass http://soul;
			proxy_redirect default;s
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}


 ```
#  压测时，结果：order-service的两个实例都有收到集群网关的转发，说明我们的集群生效了
>  8099
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210127003831323.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

>  8098
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210127003831286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)



>http://localhost:80/order-service/order/gateway: 这是通过nginx代理两个网关的最后请求地址
 ```Java   
D:\worksapce-vfresh> sb -u http://localhost:80/order-service/order/gateway -c 20  -N 60 -m post
Starting at 2021-01-27 0:14:46
[Press C to stop the test]
21090   (RPS: 307.8)
---------------Finished!----------------
Finished at 2021-01-27 0:15:55 (took 00:01:08.8615283)
Status 200:    21090

RPS: 340 (requests/second)
Max: 3047ms
Min: 5ms
Avg: 31.9ms

  50%   below 24ms
  60%   below 27ms
  70%   below 30ms
  80%   below 33ms
  90%   below 40ms
  95%   below 48ms
  98%   below 66ms
  99%   below 86ms
99.9%   below 2407ms
 ```


> 我们使用websocket同步数据，今天忘记添加它的依赖，导致网关请求不成功，一直报错：Can not find url, please check your configuration!
 ```Java   
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>soul-spring-boot-starter-sync-data-websocket</artifactId>
        <version>${project.version}</version>
    </dependency>
 ```

## 总结
*  注意网关的websocket的urls配置，其他单节点网关配置都差不多
