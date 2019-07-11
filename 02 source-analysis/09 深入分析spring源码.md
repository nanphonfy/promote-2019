
>web.xml配置一个dispatcherservlet（启动入口），实现Aware接口，能够得到ApplicationContext->默认加载IOC容器（ApplicationContext）->开始扫描spring mvc配置（扫描注解controller/requestmapping/view的配置），插件（拦截器、转换器、视图解析器）->解析成一个handlermapping的list，保存url和具体的执行方对应关系。

>等待用户请求的过程：浏览器输入URL->统一拦截（扫描handlermapping）->dispatcherservlet接收到请求，从上面初始化已保存的数据中找到url对应方法调用->输出结果。

#### 1.1 建立Map<urls,Controller>关系
```java 
//org.springframework.context.support.ApplicationObjectSupport
// public abstract class ApplicationObjectSupport implements ApplicationContextAware
public final void setApplicationContext(ApplicationContext context) throws BeansException {
    if (context == null && !isContextRequired()) {
    	// Reset internal context state.
    	this.applicationContext = null;
    	this.messageSourceAccessor = null;
    }
    else if (this.applicationContext == null) {
    	// Initialize with passed-in context.
    	if (!requiredContextClass().isInstance(context)) {
    		throw new ApplicationContextException(
    				"Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
    	}
    	this.applicationContext = context;
    	this.messageSourceAccessor = new MessageSourceAccessor(context);
    	// 放到子类实现
    	initApplicationContext(context);
    }
    else {
    	// Ignore reinitialization if same context passed in.
    	if (this.applicationContext != context) {
    		throw new ApplicationContextException(
    				"Cannot reinitialize with different application context: current one is [" +
    				this.applicationContext + "], passed-in one is [" + context + "]");
    	}
    }
}
```


```java 
// org.springframework.web.servlet.handler.AbstractDetectingUrlHandlerMapping
// public abstract class AbstractDetectingUrlHandlerMapping extends AbstractUrlHandlerMapping
@Override
public void initApplicationContext() throws ApplicationContextException {
	super.initApplicationContext();
	detectHandlers();
}

/**
 * 建立当前ApplicationContext的所有controller和url的对应关系
 * @throws BeansException
 */
protected void detectHandlers() throws BeansException {
	if (logger.isDebugEnabled()) {
		logger.debug("Looking for URL mappings in application context: " + getApplicationContext());
	}
	// 获取ApplicationContext中的所有bean&name
	String[] beanNames = (this.detectHandlersInAncestorContexts ?
			BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
			getApplicationContext().getBeanNamesForType(Object.class));

	// Take any bean name that we can determine URLs for.
	// 遍历beanNames，并找到这些bean对应的url
	for (String beanName : beanNames) {
		// 找到bean上所有url（controller的url&方法的url）——由子类实现
		String[] urls = determineUrlsForHandler(beanName);
		if (!ObjectUtils.isEmpty(urls)) {
			// URL paths found: Let's consider it a handler.
			// 保存urls和beanName的对应关系，put it to Map<urls,beanName>，该方法在父类AbstractUrlHandlerMapping中实现
			registerHandler(urls, beanName);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Rejected bean name '" + beanName + "': no URL paths identified");
			}
		}
	}
}
```


```java 
// org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
// public class BeanNameUrlHandlerMapping extends AbstractDetectingUrlHandlerMapping
/**
 * 获取controller所有的url
 * @param beanName the name of the candidate bean
 * @return
 */
@Override
protected String[] determineUrlsForHandler(String beanName) {
	List<String> urls = new ArrayList<String>();
	if (beanName.startsWith("/")) {
		urls.add(beanName);
	}
	String[] aliases = getApplicationContext().getAliases(beanName);
	for (String alias : aliases) {
		if (alias.startsWith("/")) {
			urls.add(alias);
		}
	}
	return StringUtils.toStringArray(urls);
}
```

#### 1.2 根据url找到对应Controller方法
- DispatcherServlet的核心方法:doService()

```java 
// org.springframework.web.servlet.DispatcherServlet
// public class DispatcherServlet extends FrameworkServlet
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (logger.isDebugEnabled()) {
		String requestUri = urlPathHelper.getRequestUri(request);
		String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
		logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
				" processing " + request.getMethod() + " request for [" + requestUri + "]");
	}

	// Keep a snapshot of the request attributes in case of an include,
	// to be able to restore the original attributes after the include.
	Map<String, Object> attributesSnapshot = null;
	if (WebUtils.isIncludeRequest(request)) {
		logger.debug("Taking snapshot of request attributes before include");
		attributesSnapshot = new HashMap<String, Object>();
		Enumeration<?> attrNames = request.getAttributeNames();
		while (attrNames.hasMoreElements()) {
			String attrName = (String) attrNames.nextElement();
			if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
				attributesSnapshot.put(attrName, request.getAttribute(attrName));
			}
		}
	}

	// Make framework objects available to handlers and view objects.
	request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
	request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
	request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
	request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

	FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
	if (inputFlashMap != null) {
		request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
	}
	request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
	request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

	try {
		doDispatch(request, response);
	}
	finally {
		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			return;
		}
		// Restore the original attribute snapshot, in case of an include.
		if (attributesSnapshot != null) {
			restoreAttributesAfterInclude(request, attributesSnapshot);
		}
	}
}

/** 中央控制器,控制请求的转发 **/
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			//1.检查是否是文件上传的请求
			processedRequest = checkMultipart(request);
			multipartRequestParsed = processedRequest != request;

			// Determine handler for the current request.
			// 2.取得处理当前请求的controller,这里也称为hanlder,处理器,第一个步骤的意义就在这里体现了.
			//这里并不是直接返回controller,而是返回的HandlerExecutionChain请求处理器链对象,
			//该对象封装了handler和interceptors.
			mappedHandler = getHandler(processedRequest, false);
			// 如果handler为空,则返回404
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			//3. 获取处理request的处理器适配器handler adapter 
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			// 处理 last-modified 请求头
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					String requestUri = urlPathHelper.getRequestUri(request);
					logger.debug("Last-Modified value for [" + requestUri + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			// 4.拦截器的预处理方法
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			try {
				// Actually invoke the handler.
				// 5.实际的处理器处理请求,返回结果视图对象
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
			}
			finally {
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
			}

			// 结果视图对象的处理
			applyDefaultViewName(request, mv);
			// 6.拦截器的后处理方法
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		// 请求成功响应之后的方法
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Error err) {
		triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			return;
		}
		// Clean up any resources used by a multipart request.
		if (multipartRequestParsed) {
			cleanupMultipart(processedRequest);
		}
	}
}
```
#### 1.3 反射调用处理请求方法，返回结果视图
>根据url确定Controller中处理请求的方法,然后通过反射获取该方法上的注解和参数,解析方法和参数上的注解,最后反射调用方法获取ModelAndView结果视图。

```java 
// org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
// public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware,
		InitializingBean
protected final ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
		// Always prevent caching in case of session attribute management.
		checkAndPrepare(request, response, this.cacheSecondsForSessionAttributeHandlers, true);
	}
	else {
		// Uses configured default cacheSeconds setting.
		checkAndPrepare(request, response, true);
	}

	// Execute invokeHandlerMethod in synchronized block if required.
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				return invokeHandleMethod(request, response, handlerMethod);
			}
		}
	}

	return invokeHandleMethod(request, response, handlerMethod);
}

// 根据url获取处理请求的方法
// org.springframework.web.servlet.handler.AbstractHandlerMethodMapping
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    // 若请求url为http://localhost:8080/web/hello.json，则lookupPath=web/hello.json
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    if (logger.isDebugEnabled()) {
    	logger.debug("Looking up handler method for path " + lookupPath);
    }
    // 遍历controller上的所有方法，获取url匹配方法
    HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
    
    if (logger.isDebugEnabled()) {
    	if (handlerMethod != null) {
    		logger.debug("Returning handler method [" + handlerMethod + "]");
    	}
    	else {
    		logger.debug("Did not find handler method for [" + lookupPath + "]");
    	}
    }
    
    return (handlerMethod != null) ? handlerMethod.createWithResolvedBean() : null;
    }
  
// 获取处理请求的方法，执行并返回结果视图
// org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
private ModelAndView invokeHandleMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ServletWebRequest webRequest = new ServletWebRequest(request, response);

	WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
	ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
	ServletInvocableHandlerMethod requestMappingMethod = createRequestMappingMethod(handlerMethod, binderFactory);

	ModelAndViewContainer mavContainer = new ModelAndViewContainer();
	mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
	modelFactory.initModel(webRequest, mavContainer, requestMappingMethod);
	mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

	AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
	asyncWebRequest.setTimeout(this.asyncRequestTimeout);

	final WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.setTaskExecutor(this.taskExecutor);
	asyncManager.setAsyncWebRequest(asyncWebRequest);
	asyncManager.registerCallableInterceptors(this.callableInterceptors);
	asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

	if (asyncManager.hasConcurrentResult()) {
		Object result = asyncManager.getConcurrentResult();
		mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
		asyncManager.clearConcurrentResult();

		if (logger.isDebugEnabled()) {
			logger.debug("Found concurrent result value [" + result + "]");
		}
		requestMappingMethod = requestMappingMethod.wrapConcurrentResult(result);
	}

	requestMappingMethod.invokeAndHandle(webRequest, mavContainer);

	if (asyncManager.isConcurrentHandlingStarted()) {
		return null;
	}

	return getModelAndView(mavContainer, modelFactory, webRequest);
}
```
>requestMappingMethod.invokeAndHandle最终目的:完成Request参数和方法参数上数据的绑定。
SpringMVC提供两种Request参数到方法中参数的绑定方式:  
①通过注解进行绑定,@RequestParam;  
②通过参数名称进行绑定.  
>使用注解绑定,只要方法参数前声明@RequestParam("a"),就可将request中参数a的值绑定到方法参数上。使用参数名称绑定，前提：必须获取方法参数名称,Java反射只提供获取方法的参数类型,并没提供获取参数名称的方法.解决该问题：用asm框架读取字节码,获取方法参数名称。asm框架是一个字节码操作框架（建议使用注解）。

```java 
// org.springframework.web.method.support.InvocableHandlerMethod
public final Object invokeForRequest(NativeWebRequest request,
									 ModelAndViewContainer mavContainer,
									 Object... providedArgs) throws Exception {
	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);

	if (logger.isTraceEnabled()) {
		StringBuilder builder = new StringBuilder("Invoking [");
		builder.append(this.getMethod().getName()).append("] method with arguments ");
		builder.append(Arrays.asList(args));
		logger.trace(builder.toString());
	}

	Object returnValue = invoke(args);

	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + this.getMethod().getName() + "] returned [" + returnValue + "]");
	}

	return returnValue;
}

// org.springframework.web.method.support.InvocableHandlerMethod
private Object[] getMethodArgumentValues(
		NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(parameterNameDiscoverer);
		GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());

		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}

		if (argumentResolvers.supportsParameter(parameter)) {
			try {
				args[i] = argumentResolvers.resolveArgument(parameter, mavContainer, request, dataBinderFactory);
				continue;
			} catch (Exception ex) {
				if (logger.isTraceEnabled()) {
					logger.trace(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
				}
				throw ex;
			}
		}

		if (args[i] == null) {
			String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
			throw new IllegalStateException(msg);
		}
	}
	return args;
}
```
