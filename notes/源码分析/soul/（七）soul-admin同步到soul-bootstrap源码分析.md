# soul-admin同步到soul-bootstrap源码分析

##  目标
* SoulClientController的内容



## SoulClientController的内容

*  1：一共六个接口：springmvc-register，springcloud-register，springcloud-register，dubbo-register，sofa-register，tars-register
    
*  2：六个接口全部用于各个插件通过okhttp方式调用接口：
    * 处理被代理服务器的注册信息新增到： meta_data表
    * 通过发布spring事件把元数据刷新到soulbootstrap

``` Java
  RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
``` 


