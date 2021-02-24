# 响应式编程中的Flux和Mono 的理解
## 1.1 Publisher
由于响应流的特点，我们不能再返回一个简单的POJO对象来表示结果了。必须返回一个类似Java中的Future的概念，在有结果可用时通知消费者进行消费响应。  
## 2.1  Flux 
  是一个发出(emit)0-N个元素组成的异步序列的Publisher<T>,可以被onComplete信号或者onError信号所终止 ，在有结果可用时通知消费者进行消费响应
传统数据处理
```Java
public List<ClientUser> allUsers() {
    return Arrays.asList(new ClientUser("felord.cn", "reactive"),
            new ClientUser("Felordcn", "Reactor"));
}
```
流式数据处理
```Java
public Stream<ClientUser> allUsers() {
    return  Stream.of(new ClientUser("felord.cn", "reactive"),
            new ClientUser("Felordcn", "Reactor"));
}
```
反应式数据处理
```Java
public Flux<ClientUser> allUsers(){
    return Flux.just(new ClientUser("felord.cn", "reactive"),
            new ClientUser("Felordcn", "Reactor"));
}
```
## 3 .1 Mono<T>
Mono 是一个发出(emit)0-1个元素的Publisher<T>,可以被onComplete信号或者onError信号所终止。
反应式数据处理
```Java
 public Mono<ClientUser> currentUser () {
    return isAuthenticated ? Mono.just(new ClientUser("felord.cn", "reactive"))
            : Mono.empty();
}
```
和Optional有点类似的机制，当然Mono不是为了解决NPE问题的，它是为了处理响应流中单个值（也可能是Void）而存在的 
总结：Fulx和Mono不太好理解，需要结合代码及调试深入去实践
>参考链接：
https://developer.ibm.com/zh/languages/java/articles/j-cn-with-reactor-response-encode/  （详细使用案例）
https://juejin.cn/post/6844903858599428110
https://segmentfault.com/a/1190000024499748
