[toc]

### 2.IOC容器初始化
#### 2.2 FileSystemXmlApplicationContext的IOC容器流程   
##### 2.2.11 DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法
>DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析（实现BeanDefinitionDocumentReader接口的registerBeanDefinitions）

```java 
//org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
//根据Spring DTD对Bean的定义规则解析Bean定义Document对象
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	//获得XML描述符
	this.readerContext = readerContext;
	logger.debug("Loading bean definitions");
	//获得Document的根元素
	Element root = doc.getDocumentElement();
	doRegisterBeanDefinitions(root);
}

protected void doRegisterBeanDefinitions(Element root) {
	String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
	if (StringUtils.hasText(profileSpec)) {
		Assert.state(this.environment != null, "Environment must be set for evaluating profiles");
		String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		if (!this.environment.acceptsProfiles(specifiedProfiles)) {
			return;
		}
	}

	//具体的解析过程由BeanDefinitionParserDelegate实现，  
    //BeanDefinitionParserDelegate中定义了Spring Bean定义XML文件的各种元素
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(this.readerContext, root, parent);
	
	//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
	preProcessXml(root);
	//从Document的根元素开始进行Bean定义的Document对象
	parseBeanDefinitions(root, this.delegate);
	//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
	postProcessXml(root);
	
	this.delegate = parent;
}

protected BeanDefinitionParserDelegate createDelegate(XmlReaderContext readerContext, Element root, BeanDefinitionParserDelegate parentDelegate) {
	BeanDefinitionParserDelegate delegate = createHelper(readerContext, root, parentDelegate);
	if (delegate == null) {
		delegate = new BeanDefinitionParserDelegate(readerContext, this.environment);
		delegate.initDefaults(root, parentDelegate);
	}
	return delegate;
}

//使用Spring的Bean规则从Document的根元素开始进行Bean定义的Document对象
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	//Bean定义的Document对象使用了Spring默认的XML命名空间
	if (delegate.isDefaultNamespace(root)) {
		//获取Bean定义的Document对象根元素的所有子节点
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			//获得Document节点是XML元素节点
			if (node instanceof Element) {
				Element ele = (Element) node;
				//Bean定义的Document的元素节点使用的是Spring默认的XML命名空间
				if (delegate.isDefaultNamespace(ele)) {
					//使用Spring的Bean规则解析元素节点
					parseDefaultElement(ele, delegate);
				}
				else {
					//没有使用Spring默认的XML命名空间，则使用用户自定义的
					//析规则解析元素节点
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		//Document的根节点没有使用Spring默认的命名空间，则使用用户自定义的
	    //解析规则解析Document根节点
		delegate.parseCustomElement(root);
	}
}
```


```java 
//使用Spring的Bean规则解析Document元素节点
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	//如果元素节点是<Import>导入元素，进行导入解析
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
	//如果元素节点是<Alias>别名元素，进行别名解析
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
	//元素节点既不是导入元素，也不是别名元素，即普通的<Bean>元素，  
	//按照Spring的Bean规则解析元素
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}

//解析<Import>导入元素，从给定的导入路径加载Bean定义资源到Spring IoC容器中
protected void importBeanDefinitionResource(Element ele) {
	//获取给定的导入元素的location属性
	String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	//如果导入元素的location属性值为空，则没有导入任何资源，直接返回
	if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}

	// Resolve system properties: e.g. "${user.dir}"
	//使用系统变量值解析location属性值
	location = environment.resolveRequiredPlaceholders(location);

	Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

	// Discover whether the location is an absolute or relative URI
	//标识给定的导入元素的location是否是绝对路径
	boolean absoluteLocation = false;
	try {
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	}
	catch (URISyntaxException ex) {
		// cannot convert to an URI, considering the location relative
		// unless it is the well-known Spring prefix "classpath*:"
		//给定的导入元素的location不是绝对路径
	}

	// Absolute or relative?
	//给定的导入元素的location是绝对路径
	if (absoluteLocation) {
		try {
			//使用资源读入器加载给定路径的Bean定义资源
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
			}
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
					"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
		}
	}
	else {
		// No URL -> considering resource location as relative to the current file.
		//给定的导入元素的location是相对路径
		try {
			int importCount;
			//将给定导入元素的location封装为相对路径资源
			Resource relativeResource = getReaderContext().getResource().createRelative(location);
			//封装的相对路径资源存在
			if (relativeResource.exists()) {
				//使用资源读入器加载Bean定义资源
				importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
				actualResources.add(relativeResource);
			}
			//封装的相对路径资源不存在
			else {
				//获取Spring IOC容器资源读入器的基本路径 
				String baseLocation = getReaderContext().getResource().getURL().toString();
				//根据Spring IoC容器资源读入器的基本路径加载给定导入路径的资源
				importCount = getReaderContext().getReader().loadBeanDefinitions(
						StringUtils.applyRelativePath(baseLocation, location), actualResources);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
			}
		}
		catch (IOException ex) {
			getReaderContext().error("Failed to resolve current resource location", ele, ex);
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
					ele, ex);
		}
	}
	Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
	//在解析完<Import>元素之后，发送容器导入其他资源处理完成事件
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}

//解析<Alias>别名元素，为Bean向Spring IoC容器注册别名
protected void processAliasRegistration(Element ele) {
	//获取<Alias>别名元素中name的属性值
	String name = ele.getAttribute(NAME_ATTRIBUTE);
	//获取<Alias>别名元素中alias的属性值
	String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
	boolean valid = true;
	//<alias>别名元素的name属性值为空
	if (!StringUtils.hasText(name)) {
		getReaderContext().error("Name must not be empty", ele);
		valid = false;
	}
	//<alias>别名元素的alias属性值为空
	if (!StringUtils.hasText(alias)) {
		getReaderContext().error("Alias must not be empty", ele);
		valid = false;
	}
	if (valid) {
		try {
			//向容器的资源读入器注册别名
			getReaderContext().getRegistry().registerAlias(name, alias);
		}
		catch (Exception ex) {
			getReaderContext().error("Failed to register alias '" + alias +
					"' for bean with name '" + name + "'", ele, ex);
		}
		//在解析完<Alias>元素之后，发送容器别名处理完成事件
		getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
	}
}

//解析Bean定义资源Document对象的普通元素
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	// BeanDefinitionHolder是对BeanDefinition的封装，即Bean定义的封装类  
	// 对Document对象中<Bean>元素的解析由BeanDefinitionParserDelegate实现
	// BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			//向Spring IOC容器注册解析得到的Bean定义，这是Bean定义向IOC容器注册的入口
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		//在完成向Spring IOC容器注册解析得到的Bean定义之后，发送注册事件
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```
>在Spring配置文件中可使用<import>元素来导入IOC容器所需的其他资源，IOC容器在解析时首先会将指定导入的资源加载进容器中。使用<ailas>别名时，IOC容器首先将别名元素所定义的别名注册到容器中。既不是<import>元素，又不是<alias>元素，即Spring配置文件中普通的<bean>元素的解析由BeanDefinitionParserDelegate类的parseBeanDefinitionElement方法来实现。

##### 2.2.12 DefaultBeanDefinitionDocumentReader的BeanDefinitionParserDelegate方法  
>