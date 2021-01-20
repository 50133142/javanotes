# （（五）springcloud数据注册到soul-admin

##  目标
* springcloud的接口信息获取调用soul-admin源码分析
* soul-admin的soul-client/springcloud-register具体处理
* SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的


## springcloud数据注册到soul-admin源码分析

```
 <dependency>
     <groupId>org.dromara</groupId>
     <artifactId>soul-spring-boot-starter-client-springcloud</artifactId>
     <version>2.2.1</version>
 </dependency>
```
> 1 ：soul-spring-boot-starter-client-springcloud 里面就一个类：soul-spring-boot-starter-client-springcloud 下面依次分析：
>SoulSpringCloudConfig 获取yml文件配置的
``` Java
    @Bean
    @ConfigurationProperties(prefix = "soul.springcloud")
    public SoulSpringCloudConfig soulSpringCloudConfig() {
        return new SoulSpringCloudConfig();
    }
```
>2：ContextRegisterListener ，监听器使用上下文，从类的命名我们大概就猜出这个类具体干什么的，
``` Java
    @Bean
    public ContextRegisterListener contextRegisterListener(final SoulSpringCloudConfig soulSpringCloudConfig, final Environment env) {
        return new ContextRegisterListener(soulSpringCloudConfig, env);
    }
// 初始化之前先要初始化SoulSpringCloudConfig类，下面是ContextRegisterListener的核心代码：

    @Override
    public void onApplicationEvent(final ContextRefreshedEvent contextRefreshedEvent) {
        //防并发同时初始化
        if (!registered.compareAndSet(false, true)) {
            return;
        }
      //soul.springclou.full的配置，如果配置的true，直接把 contextPath/** ,
        if (config.isFull()) {
             //通过OkHttpTools工具请求接口：http://localhost:9095/soul-client/springcloud-register调用soul-admin， 
            RegisterUtils.doRegister(buildJsonParams(), url, RpcTypeEnum.SPRING_CLOUD);
        }
    }

    private String buildJsonParams() {
        String contextPath = config.getContextPath();
        String appName = env.getProperty("spring.application.name");
        String path = contextPath + "/**";
        SpringCloudRegisterDTO registerDTO = SpringCloudRegisterDTO.builder()
                .context(contextPath)
                .appName(appName)
                .path(path)
                .rpcType(RpcTypeEnum.SPRING_CLOUD.getName())
                .enabled(true)
                .ruleName(path)
                .build();
        return OkHttpTools.getInstance().getGson().toJson(registerDTO);
    }
```
``` 
 soul:
     springcloud:
       admin-url: http://localhost:9095
       context-path: /order-service
       full: false
   # adminUrl: 为你启动的soul-admin 项目的ip + 端口，注意要加http://
   # contextPath: 为你的这个mvc项目在soul网关的路由前缀， 比如/order ，/product 等等，网关会根据你的这个前缀来进行路由.
   # full: 设置true 代表代理你的整个服务，false表示代理你其中某几个controller
```

> 3: 初始化SpringCloudClientBeanPostProcessor，实现BeanPostProcessor,
>	spring容器初始化单例对象是【InstantiationAwareBeanPostProcessor】：提前执行；先触发：postProcessBeforeInstantiation()；如果有返回值：触发postProcessAfterInitialization()，具体怎么调用，我在文章最后分析

``` Java
  @Bean
     public SpringCloudClientBeanPostProcessor springCloudClientBeanPostProcessor(final SoulSpringCloudConfig soulSpringCloudConfig, final Environment env) {
         return new SpringCloudClientBeanPostProcessor(soulSpringCloudConfig, env);
     }


//SpringCloudClientBeanPostProcessor的核心方法postProcessAfterInitialization

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (config.isFull()) {
            return bean;
        }
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            String prePath = "";
           //获取类上得注解值
            SoulSpringCloudClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringCloudClient.class);
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.SPRING_CLOUD));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                //获取方法上得注解值
                SoulSpringCloudClient soulSpringCloudClient = AnnotationUtils.findAnnotation(method, SoulSpringCloudClient.class);
                if (Objects.nonNull(soulSpringCloudClient)) {
                   //会把类的值拼接下
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringCloudClient, finalPrePath), url,
                            RpcTypeEnum.SPRING_CLOUD));
                }
            }
        }
        return bean;
    }
```




## soul-admin的soul-client/springcloud-register具体处理

> http://localhost:9095/soul-client/springcloud-register调用soul-admin,会走到如下方法
``` Java

    @Override
    @Transactional
    public synchronized String registerSpringCloud(final SpringCloudRegisterDTO dto) {
        MetaDataDO metaDataDO = metaDataMapper.findByPath(dto.getContext() + "/**");
        if (Objects.isNull(metaDataDO)) {
            saveSpringCloudMetaData(dto);
        }
        String selectorId = handlerSpringCloudSelector(dto);
        handlerSpringCloudRule(selectorId, dto);
        return SoulResultMessage.SUCCESS;
    }
``` 
> 进到方法：saveSpringCloudMetaData
> 1： 保存数据到meta_data表
> 2： spring发布事件eventPublisher.publishEvent  ，这里明天分析

        // publish AppAuthData's event
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.META_DATA, DataEventTypeEnum.CREATE,
                Collections.singletonList(MetaDataTransfer.INSTANCE.mapToData(metaDataDO))));
                

## SpringCloudClientBeanPostProcessor的postProcessAfterInitialization方法怎么被调用的
> 看下图，我们打个断点在postProcessAfterInitialization方法上，再一次看它的调用栈，我们需要从spring的初始化对象入手，深入底层
![postProcessAfterInitialization.png.png](../soul/png/postProcessAfterInitialization.png.png "postProcessAfterInitialization.png")

### spring核心类AbstractApplicationContext的方法refresh,
>refresh里面有13个带this方法  

``` Java
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                 //重点方法，下面分析的
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }

``` 
###  this.finishBeanFactoryInitialization(beanFactory);  字面意思：完成初始化所有剩下的单实例bean
> finishBeanFactoryInitialization：初始化SpringCloudClientBeanPostProcessor对象，
>关键逻辑 
*  1：获取Bean的定义信息 
*  2：【获取当前Bean依赖的其他Bean;如果有按照getBean()把依赖的Bean先创建出来；】
*  3：创建Bean实例  createBeanInstance(beanName, mbd, args)
*  4: Bean属性赋值  populateBean(beanName, mbd, instanceWrapper); 
*  5: Bean初始化   initializeBean(beanName, exposedObject, mbd);
    *  1： wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    *  2： this.invokeInitMethods(beanName, wrappedBean, mbd);
    *  3： wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
*  6：【执行Aware接口方法】invokeAwareMethods(beanName, bean);执行xxxAware接口的方法
*  6：注册Bean的销毁方法
*  7：将创建的Bean添加到缓存中singletonObjects，ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息


