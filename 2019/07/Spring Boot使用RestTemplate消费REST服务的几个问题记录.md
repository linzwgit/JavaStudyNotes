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
public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) 

public <T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables)

public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType,Object... uriVariables)

public <T> ResponseEntity<T> postForEntity(String url, @Nullable Object request,Class<T> responseType, Object... uriVariables)

public void delete(String url, Object... uriVariables)

public void delete(URI url)

methods

```

同时要注意两个较为“灵活”的方法exchange和execute。

RestTemplate暴露的exchange与其它接口的不同：

（1）允许调用者指定HTTP请求的方法（GET,POST,DELETE等）

（2）可以在请求中增加body以及头信息，其内容通过参数‘HttpEntity<?>requestEntity’描述

（3）exchange支持‘含参数的类型’（即泛型类）作为返回类型，该特性通过‘ParameterizedTypeReference<T>responseType’描述。

RestTemplate所有的GET,POST等等方法，最终调用的都是execute方法。excute方法的内部实现是将String格式的URI转成了java.net.URI，之后调用了doExecute方法，doExecute方法的实现如下：

```java
/**
     * Execute the given method on the provided URI.
     * <p>The {@link ClientHttpRequest} is processed using the {@link RequestCallback};
     * the response with the {@link ResponseExtractor}.
     * @param url the fully-expanded URL to connect to
     * @param method the HTTP method to execute (GET, POST, etc.)
     * @param requestCallback object that prepares the request (can be {@code null})
     * @param responseExtractor object that extracts the return value from the response (can be {@code null})
     * @return an arbitrary object, as returned by the {@link ResponseExtractor}
     */
    @Nullable
    protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
            @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

        Assert.notNull(url, "'url' must not be null");
        Assert.notNull(method, "'method' must not be null");
        ClientHttpResponse response = null;
        try {
            ClientHttpRequest request = createRequest(url, method);
            if (requestCallback != null) {
                requestCallback.doWithRequest(request);
            }
            response = request.execute();
            handleResponse(url, method, response);
            if (responseExtractor != null) {
                return responseExtractor.extractData(response);
            }
            else {
                return null;
            }
        }
        catch (IOException ex) {
            String resource = url.toString();
            String query = url.getRawQuery();
            resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
            throw new ResourceAccessException("I/O error on " + method.name() +
                    " request for \"" + resource + "\": " + ex.getMessage(), ex);
        }
        finally {
            if (response != null) {
                response.close();
            }
        }
    }

doExecute

```

doExecute方法封装了模板方法，比如创建连接、处理请求和应答，关闭连接等。

多数人看到这里，估计都会觉得封装一个RestClient不过如此吧？

3、简单调用

以一个POST调用为例：



