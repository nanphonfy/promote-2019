- 自定义ApplicationContext
```java 
public class NDispatcherServlet extends HttpServlet{
    private static final String LOCATION = "npContextConfigLocation";

    private List<Handler> handlerMapping = new ArrayList<>();

    private Map<Handler,HandlerAdapter> adapterMapping = new HashMap<>();

    /**存储遍历模板目录的(模板名称、模板文件)**/
    private List<ViewResolver> viewResolvers = new ArrayList<>();

    /** eg.djfdk#name#dfkjsdfk **/
    private static final Pattern PATTERN = Pattern.compile("#(.+?)#",Pattern.CASE_INSENSITIVE);
    
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

        // 只需取出具体的某个方法
        for (Handler handler : handlerMapping) {
            // 获取该方法所有参数
            Class<?>[] paramsTypes = handler.method.getParameterTypes();

            /* 有顺序，但通过反射，没法拿到参数名字。匹配自定参数列表*/
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
        // eg./mvc/web/query/h.htm
        String url = req.getRequestURI();
        // eg./mvc
        String contextPath = req.getContextPath();
        // 解析成相对路径：/web/query/h.htm
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
            // 先取出来一个Handler，从HandlerMapping取（控制层&方法&正则表达式）
            Handler handler = getHandler(req);
            if(handler == null){
                resp.getWriter().write("404 Not Found");
                return ;
            }

            // 再取出一个适配器，再由适配调用具体方法（参数&对应索引）
            HandlerAdapter ha = getHandlerAdapter(handler);
            // 获取前端传输值，根据索引号对参数赋值（模板页面&model值）
            NModelAndView mv = ha.handle(req, resp, handler);

            /*自定义模板框架
            Veloctiy #
            Freemark  #
            JSP   ${name}
            自定义 #name#*/
            // 加载模板页面，动态设置值，渲染页面
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
                // eg.Hello, I am jack, I come from hongkong.Nice to meet you!
                resp.getWriter().write(r);
                break;
            }
        }
    }

    /**
     * 方法适配器
     */
    private class HandlerAdapter{
        private Map<String, Integer> paramMapping;

        public HandlerAdapter(Map<String, Integer> paramMapping) {
            this.paramMapping = paramMapping;
        }

        /**主要目的：用反射调用url对应的method**/
        public NModelAndView handle(HttpServletRequest req, HttpServletResponse resp, Handler handler)
                throws Exception {
            // 为什么要传req、resp、为什么传handler
            Class<?>[] paramTypes = handler.method.getParameterTypes();
            // 想给参数赋值，只能通过索引号来找到具体的某个参数
            Object[] paramValues = new Object[paramTypes.length];
            Map<String, String[]> params = req.getParameterMap();
            for (Map.Entry<String, String[]> param : params.entrySet()) {
                String value = Arrays.toString(param.getValue()).replaceAll("\\[|\\]", "").replaceAll(",\\s", ",");
                if (!this.paramMapping.containsKey(param.getKey())) {
                    continue;
                }
                int index = this.paramMapping.get(param.getKey());
                // 单个赋值不行
                paramValues[index] = castStringValue(value, paramTypes[index]);
            }
            // request和response要赋值
            String reqName = HttpServletRequest.class.getName();
            if (this.paramMapping.containsKey(reqName)) {
                int reqIndex = this.paramMapping.get(reqName);
                paramValues[reqIndex] = req;
            }
            String resqName = HttpServletResponse.class.getName();
            if (this.paramMapping.containsKey(resqName)) {
                int respIndex = this.paramMapping.get(resqName);
                paramValues[respIndex] = resp;
            }
            boolean isModelAndView = handler.method.getReturnType() == NModelAndView.class;
            Object r = handler.method.invoke(handler.controller, paramValues);
            if (isModelAndView) {
                return (NModelAndView) r;
            } else {
                return null;
            }
        }

        /**
         * 只考虑简单的几种类型，同理可推理
         * @param value
         * @param clazz
         * @return
         */
        private Object castStringValue(String value, Class<?> clazz) {
            if (clazz == String.class) {
                return value;
            } else if (clazz == Integer.class) {
                return Integer.valueOf(value);
            } else if (clazz == int.class) {
                return Integer.valueOf(value).intValue();
            } else {
                return null;
            }
        }
    }

    /**
     * HandlerMapping 定义
     */
    private class Handler{
        protected Object controller;
        protected Method method;
        protected Pattern pattern;

        protected Handler(Pattern pattern,Object controller,Method method){
            this.pattern = pattern;
            this.controller = controller;
            this.method = method;
        }
    }

    private class ViewResolver{
        /**模板名称**/
        private String viewName;
        /**模板文件**/
        private File file;

        protected ViewResolver(String viewName,File file){
            this.viewName = viewName;
            this.file = file;
        }

        protected String parse(NModelAndView mv) throws Exception{
            StringBuffer sb = new StringBuffer();
            // 只读文件
            RandomAccessFile raf = new RandomAccessFile(this.file, "r");
            try{
                /*模板框架语法很复杂，但原理相通，都是用正则表达式处理字符串，模板框架语法没有多高大上
                现在的前后端分离，后端只需暴露json即可，无需渲染数据
                自创模板语法*/
                String line = null;
                while((line = raf.readLine()) != null){
                    Matcher m = matcher(line);
                    while (m.find()) {
                        for (int i = 1; i <= m.groupCount(); i ++) {
                            String paramName = m.group(i);
                            Object paramValue = mv.getModel().get(paramName);
                            if (null == paramValue) {
                                continue;
                            }
                            line = line.replaceAll("\\#" + paramName + "\\#", paramValue.toString());
                        }
                    }
                    sb.append(line);
                }
            }finally{
                raf.close();
            }
            return sb.toString();
        }

        private Matcher matcher(String str){
            Matcher m = PATTERN.matcher(str);
            return m;
        }

        public String getViewName() {
            return viewName;
        }
    }
}
```

