## Spring RestTemplate
> Spring's central class for synchronous client-side HTTP access. It simplifies communication with HTTP servers, and enforces RESTful principles. It handles HTTP connections, leaving application code to provide URLs (with possible template variables) and extract results.

Spring 调用http的Client核心类．它简化与http server的通信，封装通信的细节，并强制遵循restfult规则．
它内部处理http connection，抽取结果，只需应用程序提供url即可．
在http通信的底层，它默认采用的是Jdk的标准设施，当然你也可以切换成其它的http类库
例如`Apache` `HttpComponents`, `Netty`, and `OkHttp`等.

最简单的上手方式:

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
HTTP method	RestTemplate methods
DELETE	delete(java.lang.String, java.lang.Object...)
GET	getForObject(java.lang.String, java.lang.Class<T>, java.lang.Object...)
getForEntity(java.lang.String, java.lang.Class<T>, java.lang.Object...)
HEAD	headForHeaders(java.lang.String, java.lang.Object...)
OPTIONS	optionsForAllow(java.lang.String, java.lang.Object...)
POST	postForLocation(java.lang.String, java.lang.Object, java.lang.Object...)
postForObject(java.lang.String, java.lang.Object, java.lang.Class<T>, java.lang.Object...)
PUT	put(java.lang.String, java.lang.Object, java.lang.Object...)
any	exchange(java.lang.String, org.springframework.http.HttpMethod, org.springframework.http.HttpEntity<?>, java.lang.Class<T>, java.lang.Object...)
execute(java.lang.String, org.springframework.http.HttpMethod, org.springframework.web.client.RequestCallback, org.springframework.web.client.ResponseExtractor<T>, java.lang.Object...)
```

*注意：传入的url会被encode，所以不要传入encode的url，以防重复encode,可以考虑使用 `UriComponentsBuilder`*

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


