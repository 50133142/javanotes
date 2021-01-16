# （三）soul-bootstrap引入springcloud插件使用.md

##  说明
* 文章主题是如何将springCloud接口，快速接入到soul-bootstrap
* 请在 soul-admin 后台将 springCloud 插件设置为开启

* 在soul-bootstrap的pom.xml添加依赖
```
 <dependency>
      <groupId>org.dromara</groupId>
      <artifactId>soul-spring-boot-starter-plugin-springcloud</artifactId>
      <version>${project.version}</version>
  </dependency>

  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-commons</artifactId>
      <version>2.2.0.RELEASE</version>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
      <version>2.2.0.RELEASE</version>
  </dependency>

```
* 这里我使用eureka作为注册和服务发现
```
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
     <version>2.2.0.RELEASE</version>
 </dependency>
```
*  在网关的yml文件中 新增如下配置
```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```
* 在自己的order-server服务添加依赖，
 order-server自己的一个springcloud项目一个微服务项目
```
 <dependency>
     <groupId>org.dromara</groupId>
     <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
     <version>2.2.1</version>
 </dependency>

 <dependency>
     <groupId>org.dromara</groupId>
     <artifactId>soul-spring-boot-starter-client-springcloud</artifactId>
     <version>2.2.1</version>
 </dependency>
```