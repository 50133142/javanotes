# （十）soul网关集群演示

##  目标
*  
* admin集群下，各实例配置信息是怎同步的
*
* 




## 启动三个soul-admin和两个网关，两个order-service，一个eureka
* 启动三个soul-admin和两个网关（网关需要配置eureka信息） ，如下图
![集群.png](../soul/png/集群.png "集群")
> 我们在9095服务修改选择器匹配规则，去9096,9097都能查看到9095时修改的配置信息
> 这次引起思考问题：admin集群下，各实例配置信息是怎同步的？ ，后续分析
>
* 两个order-service，一个eureka
![8098-8099.png](../soul/png/8098-8099.png "8098-8099")



 ```Java   
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
  sync:
      http:
        enabled: true
 ```




## 总结
*  
*  
*  