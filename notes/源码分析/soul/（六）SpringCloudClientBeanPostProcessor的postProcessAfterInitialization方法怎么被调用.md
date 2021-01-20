# SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的

##  目标
* 接着分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的



## 接着分析SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的
> 调用栈
![postProcessAfterInitialization.png.png](../soul/png/postProcessAfterInitialization.png.png "postProcessAfterInitialization.png")

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
