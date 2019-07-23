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

```java
@Override
        @SuppressWarnings("unchecked")
        public void doWithRequest(ClientHttpRequest httpRequest) throws IOException {
            super.doWithRequest(httpRequest);
            Object requestBody = this.requestEntity.getBody();
            if (requestBody == null) {
                HttpHeaders httpHeaders = httpRequest.getHeaders();
                HttpHeaders requestHeaders = this.requestEntity.getHeaders();
                if (!requestHeaders.isEmpty()) {
                    for (Map.Entry<String, List<String>> entry : requestHeaders.entrySet()) {
                        httpHeaders.put(entry.getKey(), new LinkedList<>(entry.getValue()));
                    }
                }
                if (httpHeaders.getContentLength() < 0) {
                    httpHeaders.setContentLength(0L);
                }
            }
            else {
                Class<?> requestBodyClass = requestBody.getClass();
                Type requestBodyType = (this.requestEntity instanceof RequestEntity ?
                        ((RequestEntity<?>)this.requestEntity).getType() : requestBodyClass);
                HttpHeaders httpHeaders = httpRequest.getHeaders();
                HttpHeaders requestHeaders = this.requestEntity.getHeaders();
                MediaType requestContentType = requestHeaders.getContentType();
                for (HttpMessageConverter<?> messageConverter : getMessageConverters()) {
                    if (messageConverter instanceof GenericHttpMessageConverter) {
                        GenericHttpMessageConverter<Object> genericConverter =
                                (GenericHttpMessageConverter<Object>) messageConverter;
                        if (genericConverter.canWrite(requestBodyType, requestBodyClass, requestContentType)) {
                            if (!requestHeaders.isEmpty()) {
                                for (Map.Entry<String, List<String>> entry : requestHeaders.entrySet()) {
                                    httpHeaders.put(entry.getKey(), new LinkedList<>(entry.getValue()));
                                }
                            }
                            if (logger.isDebugEnabled()) {
                                if (requestContentType != null) {
                                    logger.debug("Writing [" + requestBody + "] as \"" + requestContentType +
                                            "\" using [" + messageConverter + "]");
                                }
                                else {
                                    logger.debug("Writing [" + requestBody + "] using [" + messageConverter + "]");
                                }

                            }
                            genericConverter.write(requestBody, requestBodyType, requestContentType, httpRequest);
                            return;
                        }
                    }
                    else if (messageConverter.canWrite(requestBodyClass, requestContentType)) {
                        if (!requestHeaders.isEmpty()) {
                            for (Map.Entry<String, List<String>> entry : requestHeaders.entrySet()) {
                                httpHeaders.put(entry.getKey(), new LinkedList<>(entry.getValue()));
                            }
                        }
                        if (logger.isDebugEnabled()) {
                            if (requestContentType != null) {
                                logger.debug("Writing [" + requestBody + "] as \"" + requestContentType +
                                        "\" using [" + messageConverter + "]");
                            }
                            else {
                                logger.debug("Writing [" + requestBody + "] using [" + messageConverter + "]");
                            }

                        }
                        ((HttpMessageConverter<Object>) messageConverter).write(
                                requestBody, requestContentType, httpRequest);
                        return;
                    }
                }
                String message = "Could not write request: no suitable HttpMessageConverter found for request type [" +
                        requestBodyClass.getName() + "]";
                if (requestContentType != null) {
                    message += " and content type [" + requestContentType + "]";
                }
                throw new RestClientException(message);
            }
        }

HttpEntityRequestCallback.doWithRequest

```

最简单的解决方案是，可以通过包装http请求头，并将请求对象序列化成字符串的形式传参，参考示例代码如下：
```java
/*
     * Post请求调用
     * */
    public static String postForObject(RestTemplate restTemplate, String url, Object params) {
        HttpHeaders headers = new HttpHeaders();
        MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
        headers.setContentType(type);
        headers.add("Accept", MediaType.APPLICATION_JSON.toString());

        String json = SerializeUtil.Serialize(params);

        HttpEntity<String> formEntity = new HttpEntity<String>(json, headers);

        String result = restTemplate.postForObject(url, formEntity, String.class);

        return result;
    }

postForObject

```

如果我们还想直接返回对象，直接反序列化返回的字符串即可
```java
/*
     * Post请求调用
     * */
    public static <T> T postForObject(RestTemplate restTemplate, String url, Object params, Class<T> clazz) {
        T response = null;

        String respStr = postForObject(restTemplate, url, params);

        response = SerializeUtil.DeSerialize(respStr, clazz);

        return response;
    }

 postForObject

```

其中，序列化和反序列化工具比较多，常用的比如fastjson、jackson和gson。

2、no suitable HttpMessageConverter found for response type异常

和发起请求发生异常一样，处理应答的时候也会有问题。

StackOverflow上有人问过相同的问题，根本原因是HTTP消息转换器HttpMessageConverter缺少MIME Type，也就是说HTTP在把输出结果传送到客户端的时候，客户端必须启动适当的应用程序来处理这个输出文档，这可以通过多种MIME（多功能网际邮件扩充协议）Type来完成。

对于服务端应答，很多HttpMessageConverter默认支持的媒体类型（MIMEType）都不同。StringHttpMessageConverter默认支持的则是MediaType.TEXT_PLAIN，SourceHttpMessageConverter默认支持的则是MediaType.TEXT_XML，FormHttpMessageConverter默认支持的是MediaType.APPLICATION_FORM_URLENCODED和MediaType.MULTIPART_FORM_DATA，在REST服务中，我们用到的最多的还是MappingJackson2HttpMessageConverter，这是一个比较通用的转化器（继承自GenericHttpMessageConverter接口），根据分析，它默认支持的MIMEType为MediaType.APPLICATION_JSON：

```java
/**
     * Construct a new {@link MappingJackson2HttpMessageConverter} with a custom {@link ObjectMapper}.
     * You can use {@link Jackson2ObjectMapperBuilder} to build it easily.
     * @see Jackson2ObjectMapperBuilder#json()
     */
    public MappingJackson2HttpMessageConverter(ObjectMapper objectMapper) {
        super(objectMapper, MediaType.APPLICATION_JSON, new MediaType("application", "*+json"));
    }

MappingJackson2HttpMessageConverter

```

但是有些应用接口默认的应答MIMEType不是application/json，比如我们调用一个外部天气预报接口，如果使用RestTemplate的默认配置，直接返回一个字符串应答是没有问题的
```java
String url = "http://wthrcdn.etouch.cn/weather_mini?city=上海";
String result = restTemplate.getForObject(url, String.class);
ClientWeatherResultVO vo = SerializeUtil.DeSerialize(result, ClientWeatherResultVO.class);

```
但是，如果我们想直接返回一个实体对象：
```java
String url = "http://wthrcdn.etouch.cn/weather_mini?city=上海";

ClientWeatherResultVO weatherResultVO = restTemplate.getForObject(url, ClientWeatherResultVO.class);

```
则直接报异常：
Could not extract response: no suitable HttpMessageConverter found for response type [class ]
and content type [application/octet-stream]

很多人碰到过这个问题，首次碰到估计大多都比较懵吧，很多接口都是json或者xml或者plain text格式返回的，什么是application/octet-stream？

查看RestTemplate源代码，一路跟踪下去会发现HttpMessageConverterExtractor类的extractData方法有个解析应答及反序列化逻辑，如果不成功，抛出的异常信息和上述一致：
```java
@Override
    @SuppressWarnings({"unchecked", "rawtypes", "resource"})
    public T extractData(ClientHttpResponse response) throws IOException {
        MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
        if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
            return null;
        }
        MediaType contentType = getContentType(responseWrapper);

        try {
            for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
                if (messageConverter instanceof GenericHttpMessageConverter) {
                    GenericHttpMessageConverter<?> genericMessageConverter =
                            (GenericHttpMessageConverter<?>) messageConverter;
                    if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
                        if (logger.isDebugEnabled()) {
                            logger.debug("Reading [" + this.responseType + "] as \"" +
                                    contentType + "\" using [" + messageConverter + "]");
                        }
                        return (T) genericMessageConverter.read(this.responseType, null, responseWrapper);
                    }
                }
                if (this.responseClass != null) {
                    if (messageConverter.canRead(this.responseClass, contentType)) {
                        if (logger.isDebugEnabled()) {
                            logger.debug("Reading [" + this.responseClass.getName() + "] as \"" +
                                    contentType + "\" using [" + messageConverter + "]");
                        }
                        return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
                    }
                }
            }
        }
        catch (IOException | HttpMessageNotReadableException ex) {
            throw new RestClientException("Error while extracting response for type [" +
                    this.responseType + "] and content type [" + contentType + "]", ex);
        }

        throw new RestClientException("Could not extract response: no suitable HttpMessageConverter found " +
                "for response type [" + this.responseType + "] and content type [" + contentType + "]");
    }

HttpMessageConverterExtractor.extractData

```

StackOverflow上的解决的示例代码可以接受，但是并不准确，常见的MIMEType都应该加进去，贴一下我认为正确的代码：
```java
package com.power.demo.restclient.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.common.collect.Lists;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.http.MediaType;
import org.springframework.http.converter.*;
import org.springframework.http.converter.cbor.MappingJackson2CborHttpMessageConverter;
import org.springframework.http.converter.feed.AtomFeedHttpMessageConverter;
import org.springframework.http.converter.feed.RssChannelHttpMessageConverter;
import org.springframework.http.converter.json.GsonHttpMessageConverter;
import org.springframework.http.converter.json.JsonbHttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.http.converter.smile.MappingJackson2SmileHttpMessageConverter;
import org.springframework.http.converter.support.AllEncompassingFormHttpMessageConverter;
import org.springframework.http.converter.xml.Jaxb2RootElementHttpMessageConverter;
import org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter;
import org.springframework.http.converter.xml.SourceHttpMessageConverter;
import org.springframework.stereotype.Component;
import org.springframework.util.ClassUtils;
import org.springframework.web.client.RestTemplate;

import java.util.Arrays;
import java.util.List;

@Component
public class RestTemplateConfig {

    private static final boolean romePresent = ClassUtils.isPresent("com.rometools.rome.feed.WireFeed", RestTemplate
            .class.getClassLoader());
    private static final boolean jaxb2Present = ClassUtils.isPresent("javax.xml.bind.Binder", RestTemplate.class.getClassLoader());
    private static final boolean jackson2Present = ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", RestTemplate.class.getClassLoader()) && ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", RestTemplate.class.getClassLoader());
    private static final boolean jackson2XmlPresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.xml.XmlMapper", RestTemplate.class.getClassLoader());
    private static final boolean jackson2SmilePresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.smile.SmileFactory", RestTemplate.class.getClassLoader());
    private static final boolean jackson2CborPresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.cbor.CBORFactory", RestTemplate.class.getClassLoader());
    private static final boolean gsonPresent = ClassUtils.isPresent("com.google.gson.Gson", RestTemplate.class.getClassLoader());
    private static final boolean jsonbPresent = ClassUtils.isPresent("javax.json.bind.Jsonb", RestTemplate.class.getClassLoader());

    // 启动的时候要注意，由于我们在服务中注入了RestTemplate，所以启动的时候需要实例化该类的一个实例
    @Autowired
    private RestTemplateBuilder builder;

    @Autowired
    private ObjectMapper objectMapper;

    // 使用RestTemplateBuilder来实例化RestTemplate对象，spring默认已经注入了RestTemplateBuilder实例
    @Bean
    public RestTemplate restTemplate() {

        RestTemplate restTemplate = builder.build();

        List<HttpMessageConverter<?>> messageConverters = Lists.newArrayList();
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        converter.setObjectMapper(objectMapper);

        //不加会出现异常
        //Could not extract response: no suitable HttpMessageConverter found for response type [class ]

        MediaType[] mediaTypes = new MediaType[]{
                MediaType.APPLICATION_JSON,
                MediaType.APPLICATION_OCTET_STREAM,

                MediaType.APPLICATION_JSON_UTF8,
                MediaType.TEXT_HTML,
                MediaType.TEXT_PLAIN,
                MediaType.TEXT_XML,
                MediaType.APPLICATION_STREAM_JSON,
                MediaType.APPLICATION_ATOM_XML,
                MediaType.APPLICATION_FORM_URLENCODED,
                MediaType.APPLICATION_PDF,
        };

        converter.setSupportedMediaTypes(Arrays.asList(mediaTypes));

        //messageConverters.add(converter);
        if (jackson2Present) {
            messageConverters.add(converter);
        } else if (gsonPresent) {
            messageConverters.add(new GsonHttpMessageConverter());
        } else if (jsonbPresent) {
            messageConverters.add(new JsonbHttpMessageConverter());
        }

        messageConverters.add(new FormHttpMessageConverter());

        messageConverters.add(new ByteArrayHttpMessageConverter());
        messageConverters.add(new StringHttpMessageConverter());
        messageConverters.add(new ResourceHttpMessageConverter(false));
        messageConverters.add(new SourceHttpMessageConverter());
        messageConverters.add(new AllEncompassingFormHttpMessageConverter());
        if (romePresent) {
            messageConverters.add(new AtomFeedHttpMessageConverter());
            messageConverters.add(new RssChannelHttpMessageConverter());
        }

        if (jackson2XmlPresent) {
            messageConverters.add(new MappingJackson2XmlHttpMessageConverter());
        } else if (jaxb2Present) {
            messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
        }


        if (jackson2SmilePresent) {
            messageConverters.add(new MappingJackson2SmileHttpMessageConverter());
        }

        if (jackson2CborPresent) {
            messageConverters.add(new MappingJackson2CborHttpMessageConverter());
        }

        restTemplate.setMessageConverters(messageConverters);

        return restTemplate;
    }

}

RestTemplateConfig

```







