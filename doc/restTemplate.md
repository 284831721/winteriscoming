## Spring RestTemplate
> Spring's central class for synchronous client-side HTTP access. It simplifies communication with HTTP servers, and enforces RESTful principles. It handles HTTP connections, leaving application code to provide URLs (with possible template variables) and extract results.

Spring 调用http的Client核心类．它简化与http server的通信，并强制遵循restfult规则．
它内部处理http connection,抽取结果，只需应用程序提供url即可．
在http通信的底层，它默认采用的是Jdk的标准设施，当然你也可以切换成其它的http类库
例如`Apache` `HttpComponents`, `Netty`, and `OkHttp`等.

``` Java
   RestTemplate template = new RestTemplate();
   MyResult result = template.getForObject("http://springautowired.test.com", MyResult.class);
```

```
    HTTP method	RestTemplate methods
    DELETE	delete(java.lang.String, java.lang.Object...)
    GET	getForObject(java.lang.String, java.lang.Class<T>, java.lang.Object...)
    getForEntity(java.lang.String, java.lang.Class<T>, java.lang.Object...)
    HEAD	headForHeaders(java.lang.String, java.lang.Object...)
    OPTIONS	optionsForAllow(java.lang.String, java.lang.Object...)
    POST	postForLocation(java.lang.String, java.lang.Object, java.lang.Object...)
```

