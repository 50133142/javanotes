# springcloud-gateway-（九）ribbon的7种负载均衡策略

## ribbon有7种负载均衡策略可供选择：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210225071724382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


// 其中Rule是所有负载均衡算法的父接口
```Java
public interface IRule {
    Server choose(Object var1);

    void setLoadBalancer(ILoadBalancer var1);

    ILoadBalancer getLoadBalancer();
}
```
自定义负载均策略：自己实现Rule这个接口或者继承AbstractLoadBalancerRule，再去自定义实现接口
```Java
    @Bean
    public IRule myRule()
    {
        return new MySelfRule(); //自定义负载均衡规则
    }
   ```
### （1）轮询策略上面详细解析了，这里不重复

### （2） 随机策略解析：com.netflix.loadbalancer.RandomRule 
yml配置：
```Java

order-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```
debug模式走：di623行，看到this.rule对象是RandomRule
![图片: https://uploader.shimo.im/f/cMWBUtvUy5lOnYg4.png](https://img-blog.csdnimg.cn/20210225071751994.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

this.rule.choose()继续往下走：下图
![图片: https://uploader.shimo.im/f/ehFnVJutlHFiKB1T.png](https://img-blog.csdnimg.cn/20210225071759220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

看下看到随机获取个数组返回，再去获取实例list的其中一个值
```Java
protected int chooseRandomInt(int serverCount) {
    return ThreadLocalRandom.current().nextInt(serverCount);
}
```
走nexInt进到随机策略的核心算法
![图片: https://uploader.shimo.im/f/rKhSVn37it7IRbYI.png](https://img-blog.csdnimg.cn/20210225071806878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


### （3）RetryRule轮询重试（重试采用的默认也是轮询）

//RetryRule的成员属性和构造函数，
//默认0.5s重试
long maxRetryMillis = 500L; 
//重试采用的默认也是轮询, 
```Java
public RetryRule(IRule subRule) {
    this.subRule = (IRule)(subRule != null ? subRule : new RoundRobinRule());
}
```
核心代码：
重试策略在获取不到实例时，再次重试获取，
```Java
public Server choose(ILoadBalancer lb, Object key) {
    long requestTime = System.currentTimeMillis();
    long deadline = requestTime + this.maxRetryMillis;
    Server answer = null;
    answer = this.subRule.choose(key);
    //获取不到实例时，在maxRetryMillis后再一次调this.subRule.choose获取
    if ((answer == null || !answer.isAlive()) && System.currentTimeMillis() < deadline) {
        InterruptTask task = new InterruptTask(deadline - System.currentTimeMillis())
        while(!Thread.interrupted()) {
            answer = this.subRule.choose(key);
            if (answer != null && answer.isAlive() || System.currentTimeMillis() >= deadline) {
                break;
            }
            Thread.yield();
        }
        task.cancel();
    }

    return answer != null && answer.isAlive() ? answer : null;
}
```
### （4）WeightedResponseTimeRule响应速度决定权重：
继承RoundRobbin，加入了权重和计算响应时间的概念，。每隔30秒计算一次服务器响应时间，以响应时间作为权重，响应时间越短的服务器被选中的概率越大

### （5）BestAvailableRule最低并发（底层也有RoundRobinRule）：
最优可用，判断最优其实用的是并发连接数。选择并发连接数较小的server发送请求。

### （6）AvailabilityFilteringRule可用性过滤规则（底层也有RoundRobinRule）：
我直接翻译就是可用过滤规则，其实它功能是先过滤掉不可用的Server实例，再选择并发连接最小的实例。

### （7）ZoneAvoidanceRule区域内可用性能最优：
基于AvailabilityFilteringRule基础上做的，首先判断一个zone的运行性能是否可用，剔除不可用的区域zone的所有server，然后再利用AvailabilityPredicate过滤并发连接过多的server。

## 负载均衡7种策略性能测试对比
环境：win10 16g   惠普
压测工具：superbenchmarker
启动两个服务：localhost:8090,localhost:8091 ，一个Eureka

命令：
> sb -u http://localhost:8080/order/gateway -c 20 -N 60  -m post
> 
接口：order/gateway 很简单，没有什么逻辑处理，只返回一个字符串
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021022507202843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)

在网上找的图片：
![图片: https://uploader.shimo.im/f/y1rcrAGS03RxxB7o.png](https://img-blog.csdnimg.cn/20210225072035914.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


superbenchmarker工具使用学习链接：https://www.icode9.com/content-4-114080.html
