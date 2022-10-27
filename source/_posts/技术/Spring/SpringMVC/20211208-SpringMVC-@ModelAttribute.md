---
title: SpringMVC 参数处理 - @ModelAttribute
date: 2021-12-08 21:58:58
author: Zenfery
categories:
  - [Spring, SpringMVC]
tags:
  - Web开发
  - JavaWeb开发
---



## 一些背景

- **JDK 版本**：1.8.0_181
- **Spring**: 5.3.2
- **SpringMVC**: 5.3.2
- **SpringBoot**: 2.4.1

## 概述

Spring MVC 提供的 `@ModelAttribute` 用于更方便地操作 Model。用于接收封装请求参数，抽取 Controller 通用逻辑，Model 绑定方法返回值等。主要有以下三种用法：

## 注解于非@RequestMapping方法上

在`@Controller` 或 `@ControllerAdvice` 类中，将 `@ModelAttribute` 注解于普通的方法（非`@RequestMapping`注解）上；在`@Controller`中，作用是在当前类的所有`@RequestMapping`方法执行前执行，相当于 Controller 级别的前置过滤器；在`@ControllerAdvice`中，作用是在所有的 Controller 中的所有`@RequestMapping`方法执行前执行前执行，相当于全局级别的前置过滤器。

### 在 Controller 中使用

```java
@Controller
public class ModelArgController {

    @ModelAttribute
    public void preModelHandle(Model model){
        model.addAttribute("preModelAttr", "Pre Model Atrr Value");
    }
}
```

上例中的 `preModelHandle()` 方法，使用 `@ModelAttribute` 标注，方法的参数类型可以是 Controller 方法可使用的参数类型；其中 model 设置的 preModelAttr 属性，可以在该 Controller 中所有 @RequestMapping 标注方法中使用。

### 在 ControllerAdvice 中使用

```java
@ControllerAdvice
public class ModelControllerAdvice {

    @ModelAttribute
    public void preAdviceModelHandle(Model model){
        model.addAttribute("preAdviceModelAttr", "Pre Advice Model Atrr Value");
    }
}
```

上例中 model 设置的 preAdviceModelAttr 属性，可以在所有 Controller 的所有  @RequestMapping 标注方法中使用。

## 注解于@RequestMapping方法的参数上

注解于 `@RequestMapping` 方法的参数上，如果方法执行之前的 Model 中没有对应的参数，则会自动初始化，如果存在，则会取出之前的值，使用新值将其覆盖。对于数据类型不同，注解于否，表现则不同。下面这个示例 [ModelArgController](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/ctrl/modelattr/ModelArgController.java) ：

```java
@Controller
public class ModelArgController {

    @RequestMapping(value = "/model/modelattr/{pathVar}")
    public String modelAttr(Model model, @RequestParam String requestParam, ModelAttr ma,
        @ModelAttribute ModelAttr1 ma1, @ModelAttribute(value = "preModelAttr") String preModelAttr,
        @ModelAttribute(value = "preModelAttrList") List<String> preModelAttrList,
        @SessionAttribute("sessionAttr") String sessionAttr) {
        model.addAttribute("normalAttr", "I am normalAttr");
        return "model/model-attr";
    }
}
```



###  注解基本数据类型

注解于基本数据类型（包含普通基本类型和对应的包装类型，如 int 、Integer），要保证在方法之前，Model 中必定存在该属性的值，否则就会类似如下错误：

```bash
java.lang.IllegalStateException: No primary or single public constructor found for ...
```

> 注：接口类型的集合结构（如 Map、List）也会报上述错误，因为它们都没有对应的默认构造方法。

### 注解 String 类型

注解于 String 类型的参数上，将会将方法执行之前 Model 中对应的属性值绑定到该参数上，如上例`@ModelAttribute(value = "preModelAttr") String preModelAttr` 。如果之前的 Model 中没有，则该参数绑定空值。

### 注解自定义对象类型

采用`@ModelAttribute` 注解的对象或未注解的对象（采用此方法判断是否不是简单类型[BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.2.18.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)，并且未注解的对象参数，对应的处理器`ServletModelAttributeMethodProcessor`执行的优先级比较低），会先从 Model 、[`@SessionAttributes`](https://docs.spring.io/spring-framework/docs/5.2.18.RELEASE/spring-framework-reference/web.html#mvc-ann-sessionattributes) 中查看是否有对应的对象，如果没有则会自动初始化对象；初始化之后，会进行属性值的绑定，`WebDataBinder`类将 URL 中的请求参数(`@RequestParam`)、Model 中属性、Path属性（`@PathVariable`）等绑定至对象。

数据绑定的结果，包括异常会保存在 `BindingResult`中，如果想获取到结果内容，可以紧接着`@ModelAttribute`注解的参数后，增加一个 `BindingResult`参数来获取绑定结果。

上例中`modelAttr()`方法的参数`@ModelAttribute ModelAttr1 ma1` 对应的类 [ModelAttr1](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/model/attr/ModelAttr1.java) 如下：

```java
import lombok.Data;

@Data
public class ModelAttr1 {

    /**
     * url 中的请求参数 或 表单参数
     */
    private String requestParam;

    /**
     * url 中的请求参数（带下划线）
     */
    private String underlineRequestParam;

    /**
     * Path 中存储的属性
     */
    private String pathVar;

    /**
     * Model 中之前存储的属性
     */
    private String preAtrr;

    /**
     * Session 中存储的属性 注：这个属性是不能自动被 @ModelAttribute 注解的对象绑定
     */
    private String sessionAttr;
}
```

HTTP 请求示例：

```bash
curl http://localhost:8080/model/modelattr/pathVarValue?requestParam=123
```

参数 `ModelAttr1 ma1` 绑定的结果如下：

```bash
{
  "requestParam": "123",
  "pathVar": "pathVarValue",
  "preAtrr": "pre attr 123",
  "sessionAttr": null
}
```

> 注意：sessionAttr 属性并没有绑定，说明 session 中的属性不会被绑定。

对象绑定完属性后，若想对属性值进行校验，可以采用注解`javax.validation.Valid` 或 `@Validated` 标注，自动进行检验。



### 下划线风格的请求参数如何自动绑定至对象

在采用 SpringMVC 进行开发时，提供的接口，有的时候会采用下划线风格的参数名，但 JAVA 代码的属性命名为驼峰风格，如果将下划线风格的请求参数与驼峰风格的属性进行绑定呢？

先来看下 SpringMVC 提供的处理机制，简单分析下其源码，SpringMVC 提供的类 `ServletModelAttributeMethodProcessor` 用于绑定请求参数到对象上，它实现了接口 `HandlerMethodArgumentResolver` ，所有的参数解析器都实现了此接口。

`ServletModelAttributeMethodProcessor` 真正进行绑定的操作是由 `WebDataBinder` 的实现类来完成。

`WebDataBinder` 实现类是由 `WebDataBinderFactory` 来生产。

`WebDataBinderFactory` 是在 `RequestMappingHandlerAdapter` 来指定。看源码：

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
  // ...
  	protected InitBinderDataBinderFactory createDataBinderFactory(List<InvocableHandlerMethod> binderMethods)
			throws Exception {

		return new ServletRequestDataBinderFactory(binderMethods, getWebBindingInitializer());
	}
  // ...
  
}
```

看 `ServletRequestDataBinderFactory` 的源码，它指定使用 `ExtendedServletRequestDataBinder` 来完成绑定操作。

```java
public class ServletRequestDataBinderFactory extends InitBinderDataBinderFactory {
// ...
	@Override
	protected ServletRequestDataBinder createBinderInstance(
			@Nullable Object target, String objectName, NativeWebRequest request) throws Exception  {

		return new ExtendedServletRequestDataBinder(target, objectName);
	}

}
```

`ExtendedServletRequestDataBinder.addBindValues()` 方法扩展了 @PathVariable 类型参数的绑定操作，所以我们只需要自定义一个 `WebDataBinder` ，扩展此方法，再把此实现注册到 SpringMVC 的体系里面即可。

下面来看一下实现，因为当前大部分开发均基于 SpringBoot ，该实现也是基于 SpringBoot 环境。

首先看一下 `WebDataBinder` 的实现类 [BetterServletRequestDataBinder](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/config/springboot/BetterServletRequestDataBinder.java) ，它引用了 `ExtendedServletRequestDataBinder.addBindValues()` 方法扩展了 @PathVariable 类型参数的绑定操作，还实现了 **下划线（蛇形）参数转成驼峰参数的绑定操作**。

```java
public class BetterServletRequestDataBinder extends ExtendedServletRequestDataBinder {

    public BetterServletRequestDataBinder(Object target) {
        super(target);
    }

    public BetterServletRequestDataBinder(Object target, String objectName) {
        super(target, objectName);
    }

    @Override
    protected void addBindValues(MutablePropertyValues mpvs, ServletRequest request) {
        // 采用 ExtendedServletRequestDataBinder 的绑定 @PathVariable 参数
        super.addBindValues(mpvs, request);

        // 添加 下划线参数（蛇形参数） 对应的 驼峰 参数
        PropertyValue[] pvs = mpvs.getPropertyValues();
        Arrays.stream(pvs).forEach(pv -> {
            // System.out.println(pv.getName() + " : " + pv.getValue());
            String propertyName = pv.getName();
            StringBuilder sb = new StringBuilder();
            boolean isContainUnderline = false; // 判断是否有下划线
            boolean preIsUnderline = false; // 前一个字符是否为下划线
            for (int i = 0; i < propertyName.length(); i++) {
                char c = propertyName.charAt(i);
                if (c == '_') {
                    isContainUnderline = true;
                    preIsUnderline = true;
                } else {
                    if (preIsUnderline) {
                        sb.append(Character.toUpperCase(c));
                    } else {
                        sb.append(c);
                    }
                    preIsUnderline = false;
                }
            }

            // 如果包含下划线，添加一个新值
            if (isContainUnderline) {
                mpvs.addPropertyValue(sb.toString(), pv.getValue());
            }
        });
    }
}
```

`WebDataBinderFactory` 的实现类 [BetterServletRequestDataBinderFactory](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/config/springboot/BetterServletRequestDataBinderFactory.java) 用于生产 `BetterServletRequestDataBinder`。

```java
public class BetterServletRequestDataBinderFactory extends InitBinderDataBinderFactory {
    // ...

    @Override
    protected WebDataBinder createBinderInstance(Object target, String objectName, NativeWebRequest webRequest) throws Exception {
        return new BetterServletRequestDataBinder(target, objectName);
    }
}
```

`RequestMappingHandlerAdapter` 的实现类 [BetterRequestMappingHandlerAdapter](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/config/springboot/BetterRequestMappingHandlerAdapter.java) 用于指定采用哪个 `DataBinderFactory` 工厂。

```java
public class BetterRequestMappingHandlerAdapter extends RequestMappingHandlerAdapter {

    @Override
    protected InitBinderDataBinderFactory createDataBinderFactory(List<InvocableHandlerMethod> binderMethods) throws Exception {
        return new BetterServletRequestDataBinderFactory(binderMethods, getWebBindingInitializer());
    }
}
```

最后将 `BetterRequestMappingHandlerAdapter` 注册进 Spring。该注册方法是 SpringBoot 2.0 提供的 `WebMvcRegistrations` 注册机制。实现类 [WebMvcRegistrationsConfig](https://github.com/zenfery/spring-guide/blob/main/spring-boot-samples/springboot-test/src/main/java/com/zenfery/demo/springboottest/config/springboot/WebMvcRegistrationsConfig.java)。

```java
@Configuration
public class WebMvcRegistrationsConfig implements WebMvcRegistrations {

    @Override
    public RequestMappingHandlerAdapter getRequestMappingHandlerAdapter() {
        return new BetterRequestMappingHandlerAdapter();
    }
}
```

至此，实现已经完成了。下面顺便看一下 SpringBoot 是如何加载 `BetterRequestMappingHandlerAdapter` 。从源码可以看出，`EnableWebMvcConfiguration.createRequestMappingHandlerAdapter()` 方法先从 `mvcRegistrations` 里查找是否有自定义的 `RequestMappingHandlerAdapter` ，这样就加载了我们上面自定义的 `BetterRequestMappingHandlerAdapter`。

```java
public class WebMvcAutoConfiguration {
  // ...
  @Configuration(proxyBeanMethods = false)
	@Import(EnableWebMvcConfiguration.class)
	@EnableConfigurationProperties({ WebMvcProperties.class,
			org.springframework.boot.autoconfigure.web.ResourceProperties.class, WebProperties.class })
	@Order(0)
	public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
    // ...
  }
  
  @Configuration(proxyBeanMethods = false)
	@EnableConfigurationProperties(WebProperties.class)
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
    // ...
    
    @Bean
		@Override
		public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
				@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
				@Qualifier("mvcConversionService") FormattingConversionService conversionService,
				@Qualifier("mvcValidator") Validator validator) {
			RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter(contentNegotiationManager,
					conversionService, validator);
			adapter.setIgnoreDefaultModelOnRedirect(
					this.mvcProperties == null || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
			return adapter;
		}
    
    @Override
		protected RequestMappingHandlerAdapter createRequestMappingHandlerAdapter() {
			if (this.mvcRegistrations != null) {
				RequestMappingHandlerAdapter adapter = this.mvcRegistrations.getRequestMappingHandlerAdapter();
				if (adapter != null) {
					return adapter;
				}
			}
			return super.createRequestMappingHandlerAdapter();
		}
    
    // ...
  }
}
```

> 非SpringBoot 环境下的 SpringMVC 注册方式及加载方式，在 SpringMVC 源码解析的文章里来分析。

## 注解于@RequestMapping方法上

注解于 `@RequestMapping` 方法上，会将方法的返回值设置进 Model 中，供 View 渲染使用，若 `@ModelAttribute` 未指定名称，则会采用返回值对象类名 对应 的驼峰形式名 做为 Model 中的属性名。

`@ModelAttribute` 注解返回值示例如下：

```java
@Controller
public class ModelArgController {

    @ModelAttribute("attrReturn")
    @RequestMapping(value = "/model/model-attr-returnvalue")
    public String modelAttrReturnValue(Model model) {
        model.addAttribute("normalAttr", "I am normalAttr");
        return "Model Attr Returnvalue";
    }
}
```

> 注：在注解返回值的情况下，接口对应的要解析的模板文件即 @RequestMapping 指定的 URL ；上例中则会去解析 /model/model-attr-returnvalue.html 模板。

`View` 渲染示例（thymeleaf 模板）：

```html
<span>attrReturn: </span><span th:text="${attrReturn}">我是一个默认的 attrReturn</span>
```

页面渲染结果：

```html
<span>attrReturn: </span><span>Model Attr Returnvalue</span>
```





