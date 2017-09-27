## Spring RestTemplate
> Spring's central class for synchronous client-side HTTP access. It simplifies communication with HTTP servers, and enforces RESTful principles. It handles HTTP connections, leaving application code to provide URLs (with possible template variables) and extract results.


什么是 RestTemplate？
RestTemplate是一个http请求的客户端工具，它不是类似HttpClient的东东，也不是类似Jackson，jaxb等工具，但它封装了这些工具．

RestTemplate能做什么？
RestTemplate是Spring 调用http的client端核心类．顾名思义，与其它template类如`JdbcTemplate`一样，它封装了与http server的通信的细节，使调用端无须关心http接口响应的消息是什么格式，统统使用Java pojo来接收请求结果．它主要做了以下事情：

- 封装并屏蔽了Http通信细节
- 组装Http请求参数
- 抽取结果并转成调用端指定的Java pojo对象
- 屏蔽了不同消息格式的差异化处理


在http通信的底层，它默认采用的是Jdk的标准设施，当然你也可以切换成其它的http类库
例如`Apache` `HttpComponents`, `Netty`, and `OkHttp`等.
> 这个类是用在同步调用上的，异步调用请移步`AsyncRestTemplate`

最简单的上手方式，我们拿到的result直接就是Java Pojo，而不需要我们自己解析转换对方接口返回的Json格式数据:

``` Java
RestTemplate template = new RestTemplate();
MyResult result = template.getForObject("http://springautowired.test.com/getResult.json", MyResult.class);
```
再来看个复杂点的case:
```Java
MyRequest body = ...
RequestEntity request = RequestEntity.post(new URI("http://springautowired.com/foo")).accept(MediaType.APPLICATION_JSON).body(body);
ParameterizedTypeReference<List<MyResponse>> myBean = new ParameterizedTypeReference<List<MyResponse>>() {};
ResponseEntity<List<MyResponse>> response = template.exchange(request, myBean);

```
主要的方法有：

```Java
DELETE	delete(java.lang.String, java.lang.Object...)
GET	 getForObject(java.lang.String, java.lang.Class<T>, java.lang.Object...)
GET      getForEntity(java.lang.String, java.lang.Class<T>, java.lang.Object...)
HEAD	headForHeaders(java.lang.String, java.lang.Object...)
OPTIONS 	optionsForAllow(java.lang.String, java.lang.Object...)
POST	postForLocation(java.lang.String, java.lang.Object, java.lang.Object...)
POSt    postForObject(java.lang.String, java.lang.Object, java.lang.Class<T>, java.lang.Object...)
PUT	put(java.lang.String, java.lang.Object, java.lang.Object...)
any	exchange(java.lang.String, org.springframework.http.HttpMethod, org.springframework.http.HttpEntity<?>, java.lang.Class<T>, java.lang.Object...)
any    execute(java.lang.String, org.springframework.http.HttpMethod, org.springframework.web.client.RequestCallback, org.springframework.web.client.ResponseExtractor<T>, java.lang.Object...)
```

- 注意：传入的url会被encode，所以不要传入encode的url，以防重复encode,可以考虑使用 `UriComponentsBuilder`

在template内部使用是`HttpMessageConverter`来负责http message与POJO之间的互相转换，默认主要类型的Mime type的Converter会被自动注册，此外你也可以通过`setMessageConverters`来添加自定义converter.
来看默认的构造函数，只要对应的第三方类库存在classpath中，就会注册相应的`HttpMessageConverter`

```Java
public RestTemplate() {
this.messageConverters.add(new ByteArrayHttpMessageConverter());
this.messageConverters.add(new StringHttpMessageConverter());
this.messageConverters.add(new ResourceHttpMessageConverter());
this.messageConverters.add(new SourceHttpMessageConverter<Source>());
this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
if (romePresent) {
    this.messageConverters.add(new AtomFeedHttpMessageConverter());
    this.messageConverters.add(new RssChannelHttpMessageConverter());
}
if (jackson2XmlPresent) {
    this.messageConverters.add(new MappingJackson2XmlHttpMessageConverter());
}
else if (jaxb2Present) {
    this.messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
}
if (jackson2Present) {
    this.messageConverters.add(new MappingJackson2HttpMessageConverter());
}
else if (gsonPresent) {
    this.messageConverters.add(new GsonHttpMessageConverter());
}
}
```
默认能转换的类型有：
* json(jackson和gson都有)
* xml
* rss
* atom feed

template是采用`SimpleClientHttpRequestFactory`来处理Http的连接，用` DefaultResponseErrorHandler`处理Http错误，这些默认的策略可以采用`HttpAccessor.setRequestFactory`和`HttpAccessor.setRequestFactory`来替换．

构造方法汇总：
```Java
RestTemplate()
Create a new instance of the RestTemplate using default settings.
RestTemplate(ClientHttpRequestFactory requestFactory)
Create a new instance of the RestTemplate based on the given ClientHttpRequestFactory.
RestTemplate(List<HttpMessageConverter<?>> messageConverters)
Create a new instance of the RestTemplate using the given list of HttpMessageConverter to use
```

看个典型的方法：
```Java
public <T> ResponseEntity<T> postForEntity(String url, Object request, Class<T> responseType, Object... uriVariables)throws RestClientException {
RequestCallback requestCallback = httpEntityCallback(request, responseType);
ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);
return execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables);
	}
```
```Java
public <T> T execute(String url, HttpMethod method, RequestCallback requestCallback,
			ResponseExtractor<T> responseExtractor, Object... urlVariables) throws RestClientException {
URI expanded = getUriTemplateHandler().expand(url, urlVariables);
return doExecute(expanded, method, requestCallback, responseExtractor);
	}
```

最终调用的execute方法里有两个特殊的参数`RequestCallback`和`ResponseExtractor`．

###### RequestCallback接口
可以定制Request Header和往Request body中写入数据，主要处理请求参数等

###### ResponseExtractor接口
负责从Http response中抽取数据，转换成客户端指定的类型，其抽取转换过程是依赖`HttpMessageConverter`

这是抽取的核心代码:
```Java
public T extractData(ClientHttpResponse response) throws IOException {
    MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
    if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
        return null;
    }
    MediaType contentType = getContentType(responseWrapper);

    for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
        if (messageConverter instanceof GenericHttpMessageConverter) {
            GenericHttpMessageConverter<?> genericMessageConverter = (GenericHttpMessageConverter<?>) messageConverter;
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

    throw new RestClientException("Could not extract response: no suitable HttpMessageConverter found " +
            "for response type [" + this.responseType + "] and content type [" + contentType + "]");
}
```
原文地址：<https://github.com/284831721/winteriscoming/blob/master/doc/restTemplate.md>

