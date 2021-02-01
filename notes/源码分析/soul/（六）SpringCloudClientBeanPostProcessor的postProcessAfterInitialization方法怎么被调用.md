# SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的

##  目标
* spring容器的核心类中方法初探
* 接着分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的
* 整理的思维导图

## 先熟悉下 spring容器的核心类AbstractApplicationContext，类中方法refresh()有如下12个调用逻辑方法
* 1 this.prepareRefresh();
>刷新前的预处理:检验属性的合法等,保存容器中的一些早期的事件(new LinkedHashSet<ApplicationEvent>())
* 2 ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory(); 
>获取BeanFactory:this.beanFactory = new DefaultListableBeanFactory()
* 3 this.prepareBeanFactory(beanFactory);
>BeanFactory进行一些设置:设置BeanFactory的类加载器、支持表达式解析器
* 4 this.postProcessBeanFactory(beanFactory);
>BeanFactory准备工作完成后进行的后置处理工作
* 5 this.invokeBeanFactoryPostProcessors(beanFactory);
>执行BeanFactoryPostProcessor的方法
* 6 this.registerBeanPostProcessors(beanFactory);
>注册BeanPostProcessor（Bean的后置处理器）【 intercept bean creation】
 		不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
 		BeanPostProcessor、
 		DestructionAwareBeanPostProcessor、
 		InstantiationAwareBeanPostProcessor、
 		SmartInstantiationAwareBeanPostProcessor、
 		MergedBeanDefinitionPostProcessor【internalPostProcessors】、


* 7 this.initMessageSource();
>初始化MessageSource组件（做国际化功能；消息绑定，消息解析）
* 8 this.initApplicationEventMulticaster();
>初始化事件派发器
* 9 this.onRefresh();
>子类重写这个方法，在容器刷新的时候可以自定义逻辑
* 10 this.registerListeners();
>给容器中将所有项目里面的ApplicationListener注册进来；
 		1、从容器中拿到所有的ApplicationListener
 		2、将每个监听器添加到事件派发器中；
 			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
 		3、派发之前步骤产生的事件；

* 11 this.finishBeanFactoryInitialization(beanFactory);
>初始化所有剩下的单实例bean
* 12 this.finishRefresh();
>完成BeanFactory的初始化创建工作；IOC容器就创建完成

## 接着文章（五）分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的
> 调用栈
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121003512130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)


> 从调用堆栈我们来到finishBeanFactoryInitialization(beanFactory)，它的关键逻辑 
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




###  2：获取当前Bean依赖的其他Bean，如果有按照getBean()把依赖的Bean先创建出来
``` Java
    this.registerDependentBean(dep, beanName);


    //上面这个方法registerDependentBean的具体实现
    public void registerDependentBean(String beanName, String dependentBeanName) {
        String canonicalName = this.canonicalName(beanName);
        Set dependenciesForBean;
        synchronized(this.dependentBeanMap) {
            dependenciesForBean = (Set)this.dependentBeanMap.computeIfAbsent(canonicalName, (k) -> {
                return new LinkedHashSet(8);
            });
            if (!dependenciesForBean.add(dependentBeanName)) {
                return;
            }
        }

        synchronized(this.dependenciesForBeanMap) {
            dependenciesForBean = (Set)this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, (k) -> {
                return new LinkedHashSet(8);
            });
            dependenciesForBean.add(canonicalName);
        }
    }

```


###  3：创建Bean实例和初始化  createBeanInstance(beanName, mbd, args)
``` Java
  //让BeanPostProcessor先拦截返回代理对象；
  beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
   //创建Bean实例对象；
   if (instanceWrapper == null) {
       instanceWrapper = this.createBeanInstance(beanName, mbd, args);
   }
```
>继续createBeanInstance 方式继续执行：利用工厂方法或者对象的构造器创建出Bean实例 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121092922239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)



>继续更调用栈走到下图的断点调用，这个时候：SpringCloudClientBeanPostProcessor就在这里被实例的，同时参数SoulSpringCloudConfig和Environment传递进来，


![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121092935900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)

###  4: Bean属性赋值  populateBean(beanName, mbd, instanceWrapper)
``` Java
    this.populateBean(beanName, mbd, instanceWrapper);
```
###  5: Bean初始化   initializeBean(beanName, exposedObject, mbd)
   *  1： wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
   *  2： this.invokeInitMethods(beanName, wrappedBean, mbd);
   *  3： wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
       (springCloudClientBeanPostProcessor的postProcessAfterInitialization方法就是在这里被调用的)
       
``` Java
        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210131123822389.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


> 进到applyBeanPostProcessorsAfterInitialization方法

``` Java

    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;

        Object current;
       // 拿到所有ioc容器的BeanPostProcessor，遍历当前对象比配是否是BeanPostProcessor的实现类，如果是执行：processor.postProcessAfterInitialization(result, beanName)
        for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
            BeanPostProcessor processor = (BeanPostProcessor)var4.next();
            current = processor.postProcessAfterInitialization(result, beanName);
            if (current == null) {
                return result;
            }
        }

        return result;
    }
```


###  6：注册Bean的销毁方法
``` Java
    this.registerDisposableBeanIfNecessary(beanName, bean, mbd);

```

###  7：将创建的Bean添加到缓存中singletonObjects，ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息
>    this.addSingleton(beanName, singletonObject);
>    singletonObjects对象是：Map<String, Object> singletonObjects = new ConcurrentHashMap(256)，即ioc容器就是这些Map
``` Java
    protected void addSingleton(String beanName, Object singletonObject) {
        synchronized(this.singletonObjects) {
            this.singletonObjects.put(beanName, singletonObject);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }

```
;
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }

```
## 整理的思维导图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210131123602225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70#pic_center)

