- 自定义ApplicationContext
```java 
public class NApplicationContext {
	/** 模拟spring的ioc容器 **/
	private Map<String,Object> instanceMapping = new ConcurrentHashMap<>();

	/** 类似内部配置信息(类外面看不到)只能通过getBean间接调用，看到ioc容器 **/
	private List<String> classCache = new ArrayList<>();

	private Properties config = new Properties();

	public static void main(String[] args) {
		NApplicationContext context = new NApplicationContext("application.properties");
	}

	public NApplicationContext(String location){
		InputStream is = null;
		try{
			// 1、定位
			is = this.getClass().getClassLoader().getResourceAsStream(location);
			// 2、载入
			config.load(is);
			// 3、注册（找出所有class，保存）
			String packageName = config.getProperty("scanPackage");
			doRegister(packageName);
			// 4、实例化需要ioc的对象(@NService，@NController等)，循环class
			doCreateBean();
			// 5、注入
			populate();
			System.out.println("<<===自定义的N-IOC容器已初始化...===>");
		}catch(Exception e){
			e.printStackTrace();
		}
	}

	/**
	 * 找出符合条件的class，注册到缓存
	 * @param packageName
     */
	private void doRegister(String packageName){
		String path = packageName.replaceAll("\\.", "\\/");
		URL url = this.getClass().getClassLoader().getResource(path);
		File dir = new File(url.getFile());
		for (File file : dir.listFiles()) {
			// 若为文件夹，则递归
			if(file.isDirectory()){
				doRegister(packageName + "." + file.getName());
			}else{
				classCache.add(packageName + "." + file.getName().replace(".class", "").trim());
			}
		}
	}
	
	private void doCreateBean(){
		/*检查有否注册信息（保存所有class名字）
		eg.BeanDefinition保存了类的名字，也保存类和类之间的关系(Map/list/Set/ref/parent)
		类之间的复杂依赖关系，暂不考虑，只考虑最简单场景。
		processArray*/
		if (classCache.size() == 0) {
			return;
		}

		try {
			for (String className : classCache) {
				// 为了简单，直接使用反射，暂不考虑JDK或Cglib动态代理
				Class<?> clazz = Class.forName(className);

				// 哪个类需初始化、不需初始化，只要加了@NController、@NService都需初始化
				if (clazz.isAnnotationPresent(NController.class)) {
					// 名字默认为类名首字母小写
					String id = lowerFirstChar(clazz.getSimpleName());
					instanceMapping.put(id, clazz.newInstance());
				} else if (clazz.isAnnotationPresent(NService.class)) {
					NService service = clazz.getAnnotation(NService.class);
					// 若设置自定义名字，则优先用自定义名字
					String id = service.value();
					if (!"".equals(id.trim())) {
						instanceMapping.put(id, clazz.newInstance());
						continue;
					}

					/*若为空，则用默认规则
					 1、类名首字母小写
					 若该类为接口
					 2、可根据类型类匹配*/
					Class<?>[] interfaces = clazz.getInterfaces();
					// 若该类实现了接口，则用接口类型作为id
					for (Class<?> i : interfaces) {
						instanceMapping.put(i.getName(), clazz.newInstance());
					}
				} else {
					continue;
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 依赖注入
	 */
	private void populate(){
		// 判断ioc容器是否为空
		if (instanceMapping.isEmpty()) {
			return;
		}

		for (Entry<String, Object> entry : instanceMapping.entrySet()) {
			// 取出所有属性(包括私有)
			Field[] fields = entry.getValue().getClass().getDeclaredFields();
			for (Field field : fields) {
				if (!field.isAnnotationPresent(NAutowired.class)) {
					continue;
				}
				NAutowired autowired = field.getAnnotation(NAutowired.class);
				String id = autowired.value().trim();
				// 若id为空，即没自定义，则默认根据类型注入
				if ("".equals(id)) {
					id = field.getType().getName();
				}
				// 开放私有变量访问权限
				field.setAccessible(true);
				try {
					field.set(entry.getValue(), instanceMapping.get(id));
				} catch (Exception e) {
					e.printStackTrace();
					continue;
				}
			}
		}
	}
	
	/**
	 * 将首字母小写
	 * @param str
	 * @return
	 */
	private String lowerFirstChar(String str){
		char [] chars = str.toCharArray();
		chars[0] += 32;
		return String.valueOf(chars);
	}

	public Map<String,Object> getAll(){
		return instanceMapping;
	}

	public Properties getConfig() {
		return config;
	}
}
```
- 

```java 
public class NDispatcherServlet extends HttpServlet{
    private static final String LOCATION = "npContextConfigLocation";

    private List<Handler> handlerMapping = new ArrayList<>();

    private Map<Handler,HandlerAdapter> adapterMapping = new HashMap<>();

    /**存储遍历模板目录的(模板名称、模板文件)**/
    private List<ViewResolver> viewResolvers = new ArrayList<>();

    private static final Pattern PATTERN = Pattern.compile("\\#\\(.+?\\)\\#",Pattern.CASE_INSENSITIVE);
    
    /** 初始化IOC容器 **/
    @Override
    public void init(ServletConfig config) throws ServletException {
        // IOC容器须先初始化（假装容器已启动）
        NApplicationContext context = new NApplicationContext(config.getInitParameter(LOCATION));
        // Map<String, Object> ioc = context.getAll();
        // 请求解析
        initMultipartResolver(context);
        // 多语言、国际化
        initLocaleResolver(context);
        // 主题View层的
        initThemeResolver(context);

        //============== 重要 ================
        // 解析url和Method的关联关系【自定义】
        initHandlerMappings(context);
        // 适配器（匹配过程）【自定义】
        initHandlerAdapters(context);
        //============== 重要 ================
        // 异常解析
        initHandlerExceptionResolvers(context);
        // 视图转发（根据视图名字匹配到具体模板）
        initRequestToViewNameTranslator(context);

        // 解析模板中的内容（拿到服务器传过来的数据，生成HTML代码）【自定义】
        initViewResolvers(context);
        initFlashMapManager(context);
        System.out.println("NSpring MVC is init.");
    }

    /**请求解析**/
    private void initMultipartResolver(NApplicationContext context) {
    }

    /**多语言、国际化**/
    private void initLocaleResolver(NApplicationContext context) {
    }

    /**主题View层的**/
    private void initThemeResolver(NApplicationContext context) {
    }
    /**解析url和Method的关联关系**/
    private void initHandlerMappings(NApplicationContext context){
        Map<String, Object> ioc = context.getAll();
        if (ioc.isEmpty()) {
            return;
        }
        /* 找出Cotroller修饰类的所有方法(应该要加RequestMaping注解)，若没加该注解，该方法不能被外界访问*/
        for (Map.Entry<String, Object> entry : ioc.entrySet()) {
            Class<?> clazz = entry.getValue().getClass();
            if (!clazz.isAnnotationPresent(NController.class)) {
                continue;
            }

            String url = "";
            if (clazz.isAnnotationPresent(NRequestMapping.class)) {
                NRequestMapping requestMapping = clazz.getAnnotation(NRequestMapping.class);
                url = requestMapping.value();
            }

            // 扫描Controller下面的所有的方法
            Method[] methods = clazz.getMethods();
            for (Method method : methods) {
                if (!method.isAnnotationPresent(NRequestMapping.class)) {
                    continue;
                }
                NRequestMapping requestMapping = method.getAnnotation(NRequestMapping.class);
                String regex = (url + requestMapping.value()).replaceAll("/+", "/");
                Pattern pattern = Pattern.compile(regex);
                handlerMapping.add(new Handler(pattern, entry.getValue(), method));
                System.out.println("np---mapping: " + regex + " " + method.toString());
            }
        }
        //RequestMapping会配置一个url，那么一个url就对应一个方法，并将这个关系保存到Map中
    }
    /**适配器（匹配的过程），主要用来动态匹配参数**/
    private void initHandlerAdapters(NApplicationContext context){
        if (handlerMapping.isEmpty()) {
            return;
        }
        // 参数类型作为key，参数的索引号作为值
        Map<String, Integer> paramMapping = new HashMap<>();

        //只需取出具体的某个方法
        for (Handler handler : handlerMapping) {
            // 获取该方法所有参数
            Class<?>[] paramsTypes = handler.method.getParameterTypes();

            /*有顺序，但通过反射，没法拿到参数名字。匹配自定参数列表*/
            for (int i = 0; i < paramsTypes.length; i++) {
                Class<?> type = paramsTypes[i];

                if (type == HttpServletRequest.class || type == HttpServletResponse.class) {
                    paramMapping.put(type.getName(), i);
                }
            }

            // 匹配Request和Response
            Annotation[][] pa = handler.method.getParameterAnnotations();
            for (int i = 0; i < pa.length; i++) {
                for (Annotation a : pa[i]) {
                    if (a instanceof NRequestParam) {
                        String paramName = ((NRequestParam) a).value();
                        if (!"".equals(paramName.trim())) {
                            paramMapping.put(paramName, i);
                        }
                    }
                }
            }
            adapterMapping.put(handler, new HandlerAdapter(paramMapping));
        }
    }

    /**
     * 异常解析
     **/
    private void initHandlerExceptionResolvers(NApplicationContext context) {
    }

    /**
     * 视图转发（根据视图名字匹配具体模板）
     **/
    private void initRequestToViewNameTranslator(NApplicationContext context) {
    }

    /**
     * 解析模板内容（拿到服务器数据，生成HTML代码）
     **/
    private void initViewResolvers(NApplicationContext context) {
        /*模板一般不会放到WebRoot下，而是放在WEB-INF或classes下(避免用户直接请求模板)
        加载模板的个数，存储到缓存中，检查模板中的语法错误*/
        String tempateRoot = context.getConfig().getProperty("templateRoot");
        // 模板所处目录
        String rootPath = this.getClass().getClassLoader().getResource(tempateRoot).getFile();
        File rootDir = new File(rootPath);
        for (File template : rootDir.listFiles()) {
            viewResolvers.add(new ViewResolver(template.getName(), template));
        }
    }

    private void initFlashMapManager(NApplicationContext context) {
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doPost(req, resp);
    }

    /**调用自定义Controller的方法**/
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try{
            doDispatch(req,resp);
        }catch(Exception e){
            resp.getWriter().write("500 Exception, Msg :" + Arrays.toString(e.getStackTrace()));
        }
    }

    private Handler getHandler(HttpServletRequest req){
        if (handlerMapping.isEmpty()) {
            return null;
        }
        // eg.
        String url = req.getRequestURI();
        // eg.
        String contextPath = req.getContextPath();
        // 解析成相对路径
        url = url.replace(contextPath, "").replaceAll("/+", "/");

        // 循环handlerMapping
        for (Handler handler : handlerMapping) {
            Matcher matcher = handler.pattern.matcher(url);
            if (!matcher.matches()) {
                continue;
            }
            return handler;
        }

        return null;
    }

    private HandlerAdapter getHandlerAdapter(Handler handler) {
        if (adapterMapping.isEmpty()) {
            return null;
        }
        return adapterMapping.get(handler);
    }

    private void doDispatch(HttpServletRequest req, HttpServletResponse resp) throws Exception{
        try{
            // 先取出来一个Handler，从HandlerMapping取
            Handler handler = getHandler(req);
            if(handler == null){
                resp.getWriter().write("404 Not Found");
                return ;
            }

            // 再取出一个适配器，再由适配调用具体方法
            HandlerAdapter ha = getHandlerAdapter(handler);
            NModelAndView mv = ha.handle(req, resp, handler);

            /*自定义模板框架
            Veloctiy #
            Freemark  #
            JSP   ${name}
            自定义 #name#*/
            applyDefaultViewName(resp, mv);
        }catch(Exception e){
            throw e;
        }
    }

    public void applyDefaultViewName(HttpServletResponse resp,NModelAndView mv) throws Exception{
        if (null == mv) {
            return;
        }
        if (viewResolvers.isEmpty()) {
            return;
        }

        for (ViewResolver resolver : viewResolvers) {
            if (!mv.getView().equals(resolver.getViewName())) {
                continue;
            }
            String r = resolver.parse(mv);
            if (r != null) {
                resp.getWriter().write(r);
                break;
            }
        }
    }
}
```

