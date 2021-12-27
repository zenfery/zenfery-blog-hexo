---
title: 使用Spring的RestTemplate提供的过滤器来定制Rest请求
#permalink: spring-resttemplate-custom
categories:
  - JAVA
  - Spring
date: 2017-12-02 18:22:22
updated: 2017-12-02 20:46:00
author: Zenfery
tags:
  - 分布式
thumbnail: asset/img/thumbnail/rest.jpg
excerpt: 分布式应用环境下，免不了系统之间的相互调用；在使用 Spring 开发的应用中，在调用其它提供 HTTP 接口（Restful风格、非 RPC ）的应用时， 经常会使用到一个工具类 RestTemplate , RestTemplate 提供了调用接口的模板抽象及封装；它不仅能让我们随意选择 Http 调用客户端，还提供了定制扩展机制。下面就重点说一下它的扩展机制。
---

分布式应用环境下，免不了系统之间的相互调用；在使用 Spring 开发的应用中，在调用其它系统提供 HTTP 接口（Restful风格、非 RPC ）时， 经常会使用到一个Spring提供的工具类 RestTemplate , RestTemplate 提供了调用接口的模板抽象及封装；它不仅能让我们随意选择 Http 调用客户端，还提供了定制扩展机制。下面就重点说一下它的扩展机制。

## 扩展机制 - 过滤器

可以为 RestTemplate 设置过滤器，过滤器可以在请求前 修改或设置 请求信息，在请求结束后，预处理请求结果。RestTemplate 继承了类 InterceptingHttpAccessor ，类 InterceptingHttpAccessor 提供了获取和设置过滤器( ClientHttpRequestInterceptor) 的方法：
```java
public abstract class InterceptingHttpAccessor extends HttpAccessor {

	private List<ClientHttpRequestInterceptor> interceptors = new ArrayList<ClientHttpRequestInterceptor>();

  // Sets the request interceptors that this accessor should use.
	public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
		this.interceptors = interceptors;
	}

  // Return the request interceptor that this accessor uses.
	public List<ClientHttpRequestInterceptor> getInterceptors() {
		return interceptors;
	}

// ...
}
```

ClientHttpRequestInterceptor 接口仅包含一个方法 ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution)。HttpRequest 是请求信息的一个封装；body 是请求 body 体；ClientHttpRequestExecution 是请求执行器的一个抽象。可以实现此接口，处理请求信息及响应结果。

## 实现思路
在项目中，有个场景是公司里某个系统A采用的是 Oauth2 认证，每次调用业务，需要调用两次；第一次获取认证结果信息，第二次根据认证结果信息请求业务数据。由于多个项目都要调用系统A，因为认证复杂，并且系统A的响应结果中带有转义字符需要处理；为此每个系统、每个新来的开发人员编写这一块调用代码都会花比较长的时间。所以就在考虑能不能用一种封装的办法来解决此问题达到一劳永逸。主要就是实现过滤器，及相应的请求和响应结果的处理。

#### 请求信息封装
- 封装认证信息。
- 认证信息缓存。认证信息有一定的失效时间，缓存起来，可以避免每次请求都要进行认证请求。
- 业务数据请求认证失败自动重新认证。
- 定期刷新认证结果信息。

#### 响应结果封装
- 自动处理认证结果中的转义字符。避免开发人员每次都得特殊处理。

主要实现代码如下：
```java
// 过滤器
public class CustomClientHttpRequestInterceptor implements
        ClientHttpRequestInterceptor {

    // 存储认证信息
    private AuthInfo authInfo;
    
    public CustomClientHttpRequestInterceptor(AuthInfo authInfo) {
        this.authInfo = authInfo;
    }
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
    
        // 封装请求信息
        CustomHttpRequest httpRequest = new CustomHttpRequest(request, this.authInfo);
    
        ClientHttpResponse originalResponse = execution.execute(httpRequest, body);
    
        // 封装响应结果
        return new CustomClientHttpResponse(originalResponse);
    }

}

// 请求封装
public class CustomHttpRequest extends HttpRequestWrapper {

    private AuthInfo authInfo;
    
    public CustomHttpRequest(HttpRequest request, AuthInfo authInfo) {
        super(request);
        this.authInfo = authInfo;
    }
    
    @Override
    public URI getURI() {
    
        URI originalURI = super.getURI();
        URI newURI = null;
        // 根据规则生成新的URI ...
        return uri;
    }
    
    @Override
    public HttpHeaders getHeaders() {
        HttpHeaders headers = super.getHeaders();
        // 修改 header 信息...
        return headers;
    }

}

// 响应封装
@CommonsLog
public class CustomClientHttpResponse implements ClientHttpResponse {

    private ClientHttpResponse originalResponse;
    private InputStream handledInputStream; // 经过处理后的输入流
    
    public CustomClientHttpResponse(ClientHttpResponse originalResponse) {
        this.originalResponse = originalResponse;
    }
    
    @Override
    public InputStream getBody() throws IOException {
        if (handledInputStream != null) {
            return handledInputStream;
        }
        if (originalResponse != null) {
            InputStream originalInputStream = originalResponse.getBody();
            String content = IOUtils.toString(originalInputStream, RESPONSE_CHARSET);
    
            // 对响应结果进行处理 content ...
    
            handledInputStream = IOUtils.toInputStream(content, RESPONSE_CHARSET);
        }
        return handledInputStream;
    
    }
    
    @Override
    public void close() {
        originalResponse.close();
        if (handledInputStream != null) {
            try {
                handledInputStream.close();
            } catch (IOException e) {
                log.error("handledInputStream() close error: " + e.getMessage());
                e.printStackTrace();
            }
        }
    }

}
```
