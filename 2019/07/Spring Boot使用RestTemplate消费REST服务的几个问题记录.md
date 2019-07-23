我们可以通过Spring Boot快速开发REST接口，同时也可能需要在实现接口的过程中，通过Spring Boot调用内外部REST接口完成业务逻辑。

在Spring Boot中，调用REST Api常见的一般主要有两种方式，通过自带的RestTemplate或者自己开发http客户端工具实现服务调用。

RestTemplate基本功能非常强大，不过某些特殊场景，我们可能还是更习惯用自己封装的工具类，比如上传文件至分布式文件系统、处理带证书的https请求等。

本文以RestTemplate来举例，记录几个使用RestTemplate调用接口过程中发现的问题和解决方案。
一、RestTemplate简介
1、什么是RestTemplate

我们自己封装的HttpClient，通常都会有一些模板代码，比如建立连接，构造请求头和请求体，然后根据响应，解析响应信息，最后关闭连接。

RestTemplate是Spring中对HttpClient的再次封装，简化了发起HTTP请求以及处理响应的过程，抽象层级更高，减少消费者的模板代码，使冗余代码更少。

其实仔细想想Spring Boot下的很多XXXTemplate类，它们也提供各种模板方法，只不过抽象的层次更高，隐藏了更多细节而已。

顺便提一下，Spring Cloud有一个声明式服务调用Feign,是基于Netflix Feign实现的，整合了Spring Cloud Ribbon与 Spring Cloud Hystrix，并且实现了声明式的Web服务客户端定义方式。

本质上Feign是在RestTemplate的基础上对其再次封装，由它来帮助我们定义和实现依赖服务接口的定义。

2、RestTemplate常见方法

常见的REST服务有很多种请求方式，如GET,POST,PUT,DELETE,HEAD,OPTIONS等。RestTemplate实现了最常见的方式，用的最多的就是Get和Post了，调用API可参考源码，这里列举几个方法定义（GET、POST、DELETE）：
```java

```
