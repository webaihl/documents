#  SpringMvc 请求流程

[SpringMVC执行流程及源码分析](https://segmentfault.com/a/1190000039350496)

## 一、请求经过的主要方法

|           主键           |       说明       |          默认初始化的类(org.springframework.web.*)           |                           作用                           |                默认                |
| :----------------------: | :--------------: | :----------------------------------------------------------: | :------------------------------------------------------: | :--------------------------------: |
|   `multipartResolver`    |  文件上传解析器  |                  默认为null,需自己提供实现                   |                   用于文件上传相关配置                   |                                    |
|      localeResolver      |   国际化解析器   |         servlet.i18n.**AcceptHeaderLocaleResolver**          |               Accept-Language获取语言配置                |                                    |
|      themeResolver       |    主题解析器    |             servlet.theme.**FixedThemeResolver**             |                                                          |                                    |
|     `handlerMapping`     |   处理器映射器   | servlet.handler.**BeanNameUrlHandlerMapping**<br /> servlet.mvc.method.annotation.**RequestMappingHandlerMapping**<br />   servlet.function.support.**RouterFunctionMapping** |             通过请求地址匹配可以处理的Handle             |     BeanNameUrlHandlerMapping      |
|     `handlerAdapter`     |   处理器适配器   | servlet.mvc.**HttpRequestHandlerAdapter**<br /> servlet.mvc.**SimpleControllerHandlerAdapter**<br /> servlet.mvc.method.annotation.**RequestMappingHandlerAdapter** <br />servlet.function.support.**HandlerFunctionAdapter** | 对上一步获取到的Handle进行处理，并执行Controller中的方法 |   SimpleControllerHandlerAdapter   |
| handlerExceptionResolver | 处理器异常解析器 | mvc.method.annotation.**ExceptionHandlerExceptionResolver**<br />   servlet.mvc.annotation.**ResponseStatusExceptionResolver**<br />    servlet.mvc.support.**DefaultHandlerExceptionResolver** |                         异常处理                         |     HandlerExceptionResolvers      |
|    viewNameTranslator    |  视图名称翻译器  |        servlet.view.**InternalResourceViewResolver**         |                                                          | DefaultRequestToViewNameTranslator |
|       viewResolver       |    试图解析器    |        servlet.view.**InternalResourceViewResolver**         |               解析Modle中的数据填充到View                |                                    |
|     flashMapManager      |                  |          servlet.support.**SessionFlashMapManager**          |                                                          |                                    |

### 1. 简单流程

```java
	// 1、doService委托给doDispatch
1. doDispatch(request, response)
   // 2、检查请求是否包含文件上传，若是进行dispose中filename相关处理
2. processedRequest = checkMultipart(request)
   // 3、获取能处理该请求的Handle
3. mappedHandler = getHandler(processedRequest)
   // 4、根据Handle获取适配器
4. HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
   // 5、执行拦截器preHandle()
5. mappedHandler.applyPreHandle(processedRequest, response)
   // 6、通过适配器执行真正的controller方法，并将结果包装成mv对象
6. mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
   // 7、默认视图名称设置
7. applyDefaultViewName(processedRequest, mv);
	// 8、controlle方法执行成功后，执行拦截器的postHandle 逆序执行
8. mappedHandler.applyPostHandle(processedRequest, response, mv);
	// 9、处理返回的mv对象
9. processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	//10、mv对象处理之后，逆序执行拦截器的afterCompletion   【不论是否发生异常】
10. mappedHandler.triggerAfterCompletion(request, response, null);
	//11、发布请求处理成功的事件
11.publishRequestHandledEvent(request, response, startTime, failureCause);
```



## 二、组件初始化

> 在上图的八大组件中，mvc内置了默认的实现，如果未自定义组件的实现,容器启动时则会加载默认的bean，初始化上下文环境

```java
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```



## 三、HandleMapping

### 3.1 如何获取合适的HandleMapping

```java
	/**
	 * Return the HandlerExecutionChain for this request.
	 * <p>Tries all handler mappings in order.
	 * @param request current HTTP request
	 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
	 */
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

### 3.2 如何通过@RequestMapping关联Controller方法

> 启动加载预加载所有的单例Bean, DefaultListableBeanFactory#`preInstantiateSingletons`

1. RequestMappingHandlerMapping#afterPropertiesSet初始化回调方法

```java
public void afterPropertiesSet() {
   super.afterPropertiesSet();
}
```

2. 加载容器中所有的bean名称，根据detectHandlerMethodsInAncestorContexts bool变量的值判断是否获取父容器中的bean，默认为false。因此这里只获取Spring MVC容器中的bean，不去查找父容器，同时将过滤的bean进行handle判断

```java
protected void initHandlerMethods() {
   for (String beanName : getCandidateBeanNames()) {// Determine the names of candidate beans in the application context.
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         processCandidateBean(beanName);
      }
   }
   handlerMethodsInitialized(getHandlerMethods());
}
```

3. 处理带有`Controller` `RequestMapping`注解的Bean

```java
protected void processCandidateBean(String beanName) {
   Class<?> beanType = null;
   
      beanType = obtainApplicationContext().getType(beanName);
    
   if (beanType != null && isHandler(beanType)) {
      detectHandlerMethods(beanName);
   }
}

@Override
protected boolean isHandler(Class<?> beanType) {
   return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
         AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

4. 对符合条件的Handler中的方法进行映射注册

```java
protected void detectHandlerMethods(Object handler) {
   Class<?> handlerType = (handler instanceof String ?
         obtainApplicationContext().getType((String) handler) : handler.getClass());

   if (handlerType != null) {
      Class<?> userType = ClassUtils.getUserClass(handlerType);
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
                  return getMappingForMethod(method, userType);
            });
      
      methods.forEach((method, mapping) -> {
         Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
         registerHandlerMethod(handler, invocableMethod, mapping);
      });
   }
}

protected void registerHandlerMethod(Object handler, Method method, T mapping) {
		this.mappingRegistry.register(mapping, handler, method);
}
```

5. 注册到mappingRegistry的内容

![注册到mappingRegistry的内容](https://vip2.loli.io/2022/08/14/IaYizxtC6rVKbZD.png)

## 四、HandleAdapter

### 4.1 怎样找到合适的HandleAdapter

1. 遍历容器启动时加载的`所有HandlerAdapter`组件

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) {
         if (adapter.supports(handler)) {
            return adapter;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

2. 通过`supports`方法，确定那个Adapter能处理该Handler，每个@Controller中@RequestMapping方法都是`HandlerMethod`类型

```java
// AbstractHandlerMethodAdapter#supports
@Override
public final boolean supports(Object handler) {
   return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}

// RequestMappingHandlerAdapter#supportsInternal
@Override
protected boolean supportsInternal(HandlerMethod handlerMethod) {
	return true;
}
```

> 参数解析、返回值处理等使用的是HandleAdapter中提供的逻辑，但是在`Handle.invoke`时才会对请求内容处理

```java
// RequestMappingHandlerAdapter#getDefaultArgumentResolvers
@Override
public void afterPropertiesSet() {
   // Do this first, it may add ResponseBody advice beans
   // 缓存@ControllerAdvice @RestControllerAdvice的Bean
   initControllerAdviceCache();

   if (this.argumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
      this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.initBinderArgumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
      this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.returnValueHandlers == null) {
      List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
      this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   }
}
```

### 4.2 ArgumentSolver

1. 默认`ArgumentSolver`的加载

```java
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
   List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

   // Annotation-based argument resolution
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
   resolvers.add(new RequestParamMapMethodArgumentResolver());
   resolvers.add(new PathVariableMethodArgumentResolver());
   resolvers.add(new PathVariableMapMethodArgumentResolver());
   resolvers.add(new MatrixVariableMethodArgumentResolver());
   resolvers.add(new MatrixVariableMapMethodArgumentResolver());
   resolvers.add(new ServletModelAttributeMethodProcessor(false));
   resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new RequestHeaderMapMethodArgumentResolver());
   resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new SessionAttributeMethodArgumentResolver());
   resolvers.add(new RequestAttributeMethodArgumentResolver());

   // Type-based argument resolution
   resolvers.add(new ServletRequestMethodArgumentResolver());
   resolvers.add(new ServletResponseMethodArgumentResolver());
   resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RedirectAttributesMethodArgumentResolver());
   resolvers.add(new ModelMethodProcessor());
   resolvers.add(new MapMethodProcessor());
   resolvers.add(new ErrorsMethodArgumentResolver());
   resolvers.add(new SessionStatusMethodArgumentResolver());
   resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
   if (KotlinDetector.isKotlinPresent()) {
      resolvers.add(new ContinuationHandlerMethodArgumentResolver());
   }

   // Custom arguments
   if (getCustomArgumentResolvers() != null) {
      resolvers.addAll(getCustomArgumentResolvers());
   }

   // Catch-all
   resolvers.add(new PrincipalMethodArgumentResolver());
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
   resolvers.add(new ServletModelAttributeMethodProcessor(true));

   return resolvers;
}
```

### 4.3 ReturnValuerHadler

1. 默认的`ReturnValuerHadler`

```java
private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
   List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>(20);

   // Single-purpose return value types
   handlers.add(new ModelAndViewMethodReturnValueHandler());
   handlers.add(new ModelMethodProcessor());
   handlers.add(new ViewMethodReturnValueHandler());
   handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
         this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
   handlers.add(new StreamingResponseBodyReturnValueHandler());
   handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
         this.contentNegotiationManager, this.requestResponseBodyAdvice));
   handlers.add(new HttpHeadersReturnValueHandler());
   handlers.add(new CallableMethodReturnValueHandler());
   handlers.add(new DeferredResultMethodReturnValueHandler());
   handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

   // Annotation-based return value types
   handlers.add(new ServletModelAttributeMethodProcessor(false));
   handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
         this.contentNegotiationManager, this.requestResponseBodyAdvice));

   // Multi-purpose return value types
   handlers.add(new ViewNameMethodReturnValueHandler());
   handlers.add(new MapMethodProcessor());

   // Custom return value types
   if (getCustomReturnValueHandlers() != null) {
      handlers.addAll(getCustomReturnValueHandlers());
   }

   // Catch-all
   if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
      handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
   }
   else {
      handlers.add(new ServletModelAttributeMethodProcessor(true));
   }

   return handlers;
}
```

### 4.4 HttpMessageConverter

> 该组件并不是通过生命周期`InitializingBean`的回调方法`afterPropertiesSet`进行加载的，而是在`WebMvcConfigurationSupport#requestMappingHandlerAdapter`方法中进行收集的。该mvc支持类WebMvcConfigurationSupport通过`@EnableWebMvc`注入

1. 注入到容器中

```java
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
      @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
      @Qualifier("mvcConversionService") FormattingConversionService conversionService,
      @Qualifier("mvcValidator") Validator validator) {
	//1. 构造器中注入一部分返回值处理器
   RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
   adapter.setContentNegotiationManager(contentNegotiationManager);
   adapter.setMessageConverters(getMessageConverters());
   adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer(conversionService, validator));
   adapter.setCustomArgumentResolvers(getArgumentResolvers());
   adapter.setCustomReturnValueHandlers(getReturnValueHandlers());
// ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", classLoader) &&
//	ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", classLoader);
   if (jackson2Present) {
      adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
      adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
   }

   AsyncSupportConfigurer configurer = getAsyncSupportConfigurer();
   if (configurer.getTaskExecutor() != null) {
      adapter.setTaskExecutor(configurer.getTaskExecutor());
   }
   if (configurer.getTimeout() != null) {
      adapter.setAsyncRequestTimeout(configurer.getTimeout());
   }
   adapter.setCallableInterceptors(configurer.getCallableInterceptors());
   adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

   return adapter;
}
```

2. createRequestMappingHandlerAdapter(),构造器中注入一部分

```java
public RequestMappingHandlerAdapter() {
   this.messageConverters = new ArrayList<>(4);
   this.messageConverters.add(new ByteArrayHttpMessageConverter());
   this.messageConverters.add(new StringHttpMessageConverter());
   if (!shouldIgnoreXml) {
      try {
         this.messageConverters.add(new SourceHttpMessageConverter<>());
      }
      catch (Error err) {
         // Ignore when no TransformerFactory implementation is available
      }
   }
   this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
}
```

## 五、Interceptor.preHandle

> 责任链模式

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
   for (int i = 0; i < this.interceptorList.size(); i++) {
      HandlerInterceptor interceptor = this.interceptorList.get(i);
      if (!interceptor.preHandle(request, response, this.handler)) {
         // preHandle返回false时
         triggerAfterCompletion(request, response, null);
         return false;
      }
      this.interceptorIndex = i;
   }
   return true;
}
```

## 六、Handle.invoke

1. HandlerAdapter处理

```java
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   // 1、构造ModelAndView对象
   ModelAndView mav;
   // 2、检查是否是支持的Request Method
   checkRequest(request);
 
   // No synchronization on session demanded at all...
   mav = invokeHandlerMethod(request, response, handlerMethod);
   
   if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
      if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
         applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
      }
      else {
         prepareResponse(response);
      }
   }

   return mav;
}
```

```java
// RequestMappingHandlerAdapter#invokeHandlerMethod
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
   
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
 		// 将数据封装在一个对象中，便于多处使用
      ModelAndViewContainer mavContainer = new ModelAndViewContainer();
 	   // 调用处理handle方法
      invocableMethod.invokeAndHandle(webRequest, mavContainer);
      
      return getModelAndView(mavContainer, modelFactory, webRequest);
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

2. 实际发出请求，并查找返回值处理器，处理结果

```java
// ServletInvocableHandlerMethod#invokeAndHandle
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   //实际发出请求
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);
   ....
   // 查找返回值处理器，处理结果
   this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
}
```

3. 选择可以处理结果的返回值处理器

```java
// HandlerMethodReturnValueHandlerComposite#handleReturnValue
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
//HandlerMethodReturnValueHandlerComposite#selectHandler
@Nullable
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
   boolean isAsyncValue = isAsyncReturnValue(value, returnType);
   for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
      if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
         continue;
      }
      if (handler.supportsReturnType(returnType)) {
         return handler;
      }
   }
   return null;
}
```

4. 返回值处理器结果处理(@ResponseBody)

```java
// RequestResponseBodyMethodProcessor#handleReturnValue
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   mavContainer.setRequestHandled(true);
   ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
   ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

   // Try even with null return value. ResponseBodyAdvice could get involved.
   writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

## 七、Interceptor.postHandler

> **逆序**处理

```java
mappedHandler.applyPostHandle(processedRequest, response, mv);

void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
      throws Exception {
	
   for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
      HandlerInterceptor interceptor = this.interceptorList.get(i);
      interceptor.postHandle(request, response, this.handler, mv);
   }
}
```

## 八、Interceptor.afterComplete

>  **逆序**处理

```java
mappedHandler.triggerAfterCompletion(request, response, null);

void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) {
		for (int i = this.interceptorIndex; i >= 0; i--) {
			HandlerInterceptor interceptor = this.interceptorList.get(i);
			try {
				interceptor.afterCompletion(request, response, this.handler, ex);
			}
			catch (Throwable ex2) {
				logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
			}
		}
	}
```

## 九、总结

1. 将行为按照目的采用**接口**设计，暴露出自定义的方式给开发者，获取时采用**循环查找**。拓展时继承相关接口，无需现有逻辑
2. 匹配跟处理**分离**，增加复用减低耦合
3. **命名**处理，resolver,handler,adapter