# soul-admin同步到soul-bootstrap源码分析

##  目标
* 接着分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的



## 接着分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的

>关键逻辑 
*  1：获取Bean的定义信息 
*  2：获取当前Bean依赖的其他Bean，如果有按照getBean()把依赖的Bean先创建出来；
*  3：创建Bean实例  createBeanInstance(beanName, mbd, args)
*  4: Bean属性赋值  populateBean(beanName, mbd, instanceWrapper); 
*  5: Bean初始化   initializeBean(beanName, exposedObject, mbd);
    *  1： wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    *  2： this.invokeInitMethods(beanName, wrappedBean, mbd);
    *  3： wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
*  6：注册Bean的销毁方法
*  7：将创建的Bean添加到缓存中singletonObjects，ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息

###  1：获取Bean的定义信息 

``` Java
  RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
``` 


