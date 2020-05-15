前几天Analytics服务里job manager微服务接口报错，于是开始了捉虫之旅。感觉过程挺有意思，也许可以启发下大家，就写下来了。

登入k8s查看日志，发现是空指针异常。
```
c.s.m.a.c.u.RestTemplateInterceptor - In interceptor. Add authorization header for tenant altcdev
09:19:43.602 [cc5cf09e-2849-45ea-8871-cdd4f8c66774] [jmPollingTaskExecutor-3] [] ERROR c.s.m.a.s.a.CopyInputDataIntoS3Service - null
java.lang.NullPointerException: null
        at com.siemens.mindsphere.analytics.service.activiti.CopyInputDataIntoS3Service.createFile(CopyInputDataIntoS3Service.java:209)
        at com.siemens.mindsphere.analytics.service.activiti.CopyInputDataIntoS3Service.moveFiles(CopyInputDataIntoS3Service.java:90)
```
这段代码业务逻辑是通过调用Data Exchange服务接口下载文件存储到阿里云oss bucket里面。经过初步排查发现DE并没有异常，我们的代码也看不出问题，并且开发环境并没有这个问题。而客户那边重现的几率非常大，只好又回到日志，对比了生产环境大量相同问题的日志，最后发现是抛异常的都是下载同一个文件的时候发生，那问题就出现在这个文件上了。
为了找到了问题源头，调用DE的文件properties接口查看文件描述，发现和其他文件的不同之处是这是一个0字节的空文件。是不是空文件会导致空指针异常？为了排除这个可能，开始从代码里寻找蛛丝马迹，直接定位到报错的类，定位到createFile方法用了restTemplate来发送http请求  

`CopyInputDataIntoS3Service.java` 源码
``` java
ResponseEntity<Resource> reponse = restTemplate.exchange(endpoint, HttpMethod.GET, entity, Resource.class);
```
exchange方法将http response封装成`ResponseEntity`，接着调用reponse.getbody()就遇到了空指针异常。经过分析发现restTemplate使用ResoureHttpMessageConverter将http response封装为`ByteArrayResource`中的字节数组，便可通过调用Resource类的getInputStream()方法返回输入流，但如果http response为空，也就是空文件的情况，直接将body设置为`null`并返回。
至此问题原因明了，必须避免使用Resource.class作为返回类型，开始尝试自定义返回类型。
看了restTemplate中的方法，发现其中execute方法可以自定义responseExtractor，于是直接`InputStream inputStream = copyFileRestTemplate.execute(endpoint, HttpMethod.GET, null, HttpInputMessage::getBody)`返回输出流。然后调用阿里云oss sdk `PutObjectRequest putObjectRequest = new PutObjectRequest(bucketName, keyPath, inputStream, new ObjectMetadata());`将流传过去便成功了。  

但是事情并没有这么简单。  

改完以后SDK中 writeTo 方法 `instream.read(buffer)` 报错说流已关闭。    
`com.aliyun.oss.common.comm.io.ChunkedInputStreamEntity.java` 源码
``` java
// consume until EOF
while ((l = instream.read(buffer)) != -1) {
    outstream.write(buffer, 0, l);
}
```

这就很奇怪了，之前为什么没有出现这个报错？试了下非空文件现在也会出现 stream closed 错误！完全翻车。  
于是继续看源码，如上文所说，ResourceHttpMessageConverter将http response封装为`ByteArrayResource`中的字节数组，便可通过调用Resource类的getInputStream()方法返回输入流，所以重点是调用Resource类的子类ByteArrayResource中的getInputStream()方法返回输入流，这个流是ByteArrayResource内部的字节数组流，而http reponse中的流确实在方法执行完毕后就被关闭了。  
`ResourceHttpMessageConverter.java` 源码
``` java
if (Resource.class == clazz || ByteArrayResource.class.isAssignableFrom(clazz)) {
	byte[] body = StreamUtils.copyToByteArray(inputMessage.getBody());
```
更坑爹的是其源码使用了`this.pushbackInputStream = new PushbackInputStream(body)`回退流来判断返回是否emptyMessage, 因此不会导致流被close, 真正close掉流的是RestTemplate类doExecute方法中的finally代码块。
```
finally {
    if (response != null) {
        response.close();
    }
}
```
于是只好重写doExecute方法，去掉finally代码块，这样便不会把stream close掉了。
``` java
class CopyFileRestTemplate extends RestTemplate {
        @Override
        protected <T> T doExecute(URI url, HttpMethod method, RequestCallback requestCallback, ResponseExtractor<T> responseExtractor) throws RestClientException {
            Assert.notNull(url, "URI is required");
            Assert.notNull(method, "HttpMethod is required");
            ClientHttpResponse response = null;
            try {
                ClientHttpRequest request = createRequest(url, method);
                if (requestCallback != null) {
                    requestCallback.doWithRequest(request);
                }
                response = request.execute();
                handleResponse(url, method, response);
                return (responseExtractor != null ? responseExtractor.extractData(response) : null);
            }
            catch (IOException ex) {
                String resource = url.toString();
                String query = url.getRawQuery();
                resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
                throw new ResourceAccessException("I/O error on " + method.name() +
                        " request for \"" + resource + "\": " + ex.getMessage(), ex);
            }
        }
    }
```
试了下修改以后空文件和非空文件都没有问题，这个bug终于完全的fix了。并且因为直接把流传给了oss的接口，使得我们的服务不再使用ByteArrayResource来通过字节数组缓存文件，因此可以减少部分内存占用。  
为了验证这个想法，登陆到k8s里的pod查看堆内存情况：  

用`jstat`命令查看堆内存(相同参数调用)  

修改前 eden space使用率大概 2.08G
```
S0C    S1C    S0U    S1U      EC       EU        OC         OU      
419392.0 419392.0 64433.2  0.0   3355520.0 2084941.3 1048576.0     0.0    

```

修改后 eden space使用率大概 1.75G
```
S0C    S1C    S0U    S1U      EC       EU        OC         OU          
419392.0 419392.0 64426.8  0.0   3355520.0 1755979.1 1048576.0     0.0       
```

可以看到年轻代堆内存占用确实比之前少了三百兆左右。
