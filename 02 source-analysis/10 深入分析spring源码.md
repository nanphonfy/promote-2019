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
