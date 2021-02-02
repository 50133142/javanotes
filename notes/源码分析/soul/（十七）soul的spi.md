# （十七）soul的spi使用

##  目标
* spi是什么

## spi是什么
###  说明
>SPI 全称为 Service Provider Interface, 是 JDK 内置的一种服务提供发现功能, 一种动态替换发现的机制.
>举个例子, 要想在运行时动态地给一个接口添加实现, 只需要添加一个实现即可.
>如果现在需要对API提供一种新的实现，我们可以不用修改原来的代码，直接生成新的Jar包，在包里提供API的新实现。
>通过Java的SPI机制，可以实现了框架的动态扩展，让第三方的实现能像插件一样嵌入到系统中
>Java的SPI类似于IOC的功能，将装配的控制权移到了程序之外，实现在模块装配的时候不用在程序中动态指明。所以SPI的核心思想就是解耦
* 官方提供的脑图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202070848947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* 从上图我们知道SPI使用方法并不复杂，只需要简单的3步就可以搞定。

    * 定义一个接口
    * 提供方‘META-INF/services’目录下新建一个名称为接口全限定名的文件，内容为接口实现类的全限定名。
    * 调用方通过ServiceLoader.load方法加载接口的实现类实例
    
## soul中spi的运用    
> soulde的spi机制一样，但是对原生的spi做了增强，
>1：增加了缓存机制，2；使用的时候才去创建  3：减少遍历
### DividePlugin#doExecute核心方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202073212539.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

 ```Java   
 DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip)
 ```
>upstreamList是多个节点组成的集群， 通过负载均衡策略从集群中选择一个实例
>负载均衡策略怎么被加载起来的，是我们今天分享的重点

* LoadBalanceUtils#selector
 ```Java   
    public static DivideUpstream selector(final List<DivideUpstream> upstreamList, final String algorithm, final String ip) {
       //从LoadBalance类中获取一个实例类，然后执行实现类的select负载均衡方法 
        LoadBalance loadBalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getJoin(algorithm);
        return loadBalance.select(upstreamList, ip);
    }
 ```
* LoadBalance有三个子类实现有:
    * HashLoadBalance
    * RandomLoadBalance
    * RoundRobinLoadBalance

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202224959427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* 如下图，基本都是按照官方提供的脑图步骤实现的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202225443561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

* soul-spi有哪些类
>ExtensionLoader及对应ServiceLoader，他是扩展的加载器，自己定义了 SPI 资源文件的路径位置：  private static final String SOUL_DIRECTORY = "META-INF/soul/";
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203064034855.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

 ```Java   

    private static final String SOUL_DIRECTORY = "META-INF/soul/";

//第一层缓存, 它是我们搜索接口的具体实现类时最先接触到的, 如果命中它则直接可以得到实现类的对象 key是random，roundRobin，hash
    private static final Map<Class<?>, ExtensionLoader<?>> LOADERS = new ConcurrentHashMap<>();
//对象LoadBalance
    private final Class<T> clazz;
//cachedClasses 存放的是 标识(random) 与 类对象 的映射
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
//joinInstances 缓存存放的是 类对象与对象实例 的映射
    private final Map<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();

    private final Map<Class<?>, Object> joinInstances = new ConcurrentHashMap<>();
 ```
* 核心代码
>重要是这个方法去加载资源的
>properties
 ```Java   

    private void loadResources(final Map<String, Class<?>> classes, final URL url) throws IOException {
        try (InputStream inputStream = url.openStream()) {
            Properties properties = new Properties();
            properties.load(inputStream);
            properties.forEach((k, v) -> {
                String name = (String) k;
                String classPath = (String) v;
                if (StringUtils.isNotBlank(name) && StringUtils.isNotBlank(classPath)) {
                    try {
                        loadClass(classes, name, classPath);
                    } catch (ClassNotFoundException e) {
                        throw new IllegalStateException("load extension resources error", e);
                    }
                }
            });
        } catch (IOException e) {
            throw new IllegalStateException("load extension resources error", e);
        }
    }

 ```

 ``` java
    private void loadClass(final Map<String, Class<?>> classes,
                           final String name, final String classPath) throws ClassNotFoundException {
        Class<?> subClass = Class.forName(classPath);
        if (!clazz.isAssignableFrom(subClass)) {
            throw new IllegalStateException("load extension resources error," + subClass + " subtype is not of " + clazz);
        }
        Join annotation = subClass.getAnnotation(Join.class);
        if (annotation == null) {
            throw new IllegalStateException("load extension resources error," + subClass + " with Join annotation");
        }
        Class<?> oldClass = classes.get(name);
        if (oldClass == null) {
            classes.put(name, subClass);
        } else if (oldClass != subClass) {
            throw new IllegalStateException("load extension resources error,Duplicate class " + clazz.getName() + " name " + name + " on " + oldClass.getName() + " or " + subClass.getName());
        }
    }

 ```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203072414651.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
### 优缺点
* 优点：使用Java SPI机制的优势是实现了解耦，使第三方模块的装配逻辑与业务代码分离。应用程序可以根据实际业务情况使用新的框架拓展或者替换原有组件。
    
* 缺点：ServiceLoader在加载实现类的时候会全部加载并实例化，假如不想使用某些实现类，它也会被加载示例化的，这就造成了浪费。另外获取某个实现类只能通过迭代器迭代获取，不能根据某个参数来获取，使用方式上不够灵活。
    
* Dubbo框架中大量使用了SPI来进行框架扩展，但soul是重新对SPI进行了实现，完美的解决上面提到的问题。


