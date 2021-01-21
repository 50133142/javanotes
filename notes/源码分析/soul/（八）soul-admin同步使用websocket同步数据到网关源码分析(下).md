# soul-admin同步使用websocket同步数据到网关源码分析(下).md

##  目标
> 第七小节主要分析了soul-admin通过websocket发送数据
> 本小节主要分析soul-bootstrap怎么处理websocket发来的数据



## SoulClientController的内容

*  1：一共六个接口：springmvc-register，springcloud-register，springcloud-register，dubbo-register，sofa-register，tars-register
    
*  2：六个接口全部用于各个插件通过okhttp方式调用接口：
    * 处理被代理服务器的注册信息新增到： meta_data表
    * 通过发布spring事件把元数据刷新到soulbootstrap

``` Java
  RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
``` 


