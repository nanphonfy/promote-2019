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
>Bean定义资源文件中的<import>和<alias>元素解析在DefaultBeanDefinitionDocumentReader中已完成，对Bean定义资源文件中使用最多的<bean>元素交由BeanDefinitionParserDelegate来解析。

```java 
//org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
//解析<Bean>元素的入口
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
	return parseBeanDefinitionElement(ele, null);
}

//解析Bean定义资源文件中的<Bean>元素，这个方法中主要处理<Bean>元素的id，name和别名属性
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
	//获取<Bean>元素中的id属性值
	String id = ele.getAttribute(ID_ATTRIBUTE);
	//获取<Bean>元素中的name属性值
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	//获取<Bean>元素中的alias属性值
	List<String> aliases = new ArrayList<String>();
	//将<Bean>元素中的所有name属性值存放到别名中
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

	String beanName = id;
	//如果<Bean>元素中没有配置id属性时，将别名中的第一个值赋值给beanName
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		beanName = aliases.remove(0);
		if (logger.isDebugEnabled()) {
			logger.debug("No XML 'id' specified - using '" + beanName +
					"' as bean name and " + aliases + " as aliases");
		}
	}

	//检查<Bean>元素所配置的id或者name的唯一性，containingBean标识<Bean> 元素中是否包含子<Bean>元素
	if (containingBean == null) {
		//检查<Bean>元素所配置的id、name或者别名是否重复
		checkNameUniqueness(beanName, aliases, ele);
	}

	//详细对<Bean>元素中配置的Bean定义进行解析的地方
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
					//如果<Bean>元素中没有配置id、别名或者name，且没有包含子元素
					//<Bean>元素，为解析的Bean生成一个唯一beanName并注册
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
				   //如果<Bean>元素中没有配置id、别名或者name，且包含了子元素
					//<Bean>元素，为解析的Bean使用别名向IOC容器注册
					beanName = this.readerContext.generateBeanName(beanDefinition);
					// Register an alias for the plain bean class name, if still possible,
					// if the generator returned the class name plus a suffix.
					// This is expected for Spring 1.2/2.0 backwards compatibility.
                    //为解析的Bean使用别名注册时，为了向后兼容
					//Spring1.2/2.0，给别名添加类名后缀
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Neither XML 'id' nor 'name' specified - using generated bean name [" + beanName + "]");
				}
			}
			catch (Exception ex) {
				error(ex.getMessage(), ele);
				return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}
	//当解析出错时，返回null
	return null;
}

protected void checkNameUniqueness(String beanName, List<String> aliases, Element beanElement) {
	String foundName = null;

	if (StringUtils.hasText(beanName) && this.usedNames.contains(beanName)) {
		foundName = beanName;
	}
	if (foundName == null) {
		foundName = (String) CollectionUtils.findFirstMatch(this.usedNames, aliases);
	}
	if (foundName != null) {
		error("Bean name '" + foundName + "' is already used in this <beans> element", beanElement);
	}

	this.usedNames.add(beanName);
	this.usedNames.addAll(aliases);
}

//详细对<Bean>元素中配置的Bean定义其他属性进行解析，由于上面的方法中已经对
//Bean的id、name和别名等属性进行了处理，该方法中主要处理除这三个以外的其他属性数据  
public AbstractBeanDefinition parseBeanDefinitionElement(
		Element ele, String beanName, BeanDefinition containingBean) {

	//记录解析的<Bean>
	this.parseState.push(new BeanEntry(beanName));

	//这里只读取<Bean>元素中配置的class名字，然后载入到BeanDefinition中去  
    //只是记录配置的class名字，不做实例化，对象的实例化在依赖注入时完成
	String className = null;
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}

	try {
		String parent = null;
		//如果<Bean>元素中配置了parent属性，则获取parent属性的值
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}
		
		//根据<Bean>元素配置的class名称和parent属性值创建BeanDefinition  
        //为载入Bean定义信息做准备
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);

		//对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		//为<Bean>元素解析的Bean设置description信息
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

		//对<Bean>元素的meta(元信息)属性解析
		parseMetaElements(ele, bd);
		//对<Bean>元素的lookup-method属性解析
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
		//对<Bean>元素的replaced-method属性解析
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

		//解析<Bean>元素的构造方法设置
		parseConstructorArgElements(ele, bd);
		//解析<Bean>元素的<property>设置
		parsePropertyElements(ele, bd);
		//解析<Bean>元素的qualifier属性
		parseQualifierElements(ele, bd);

		//为当前解析的Bean设置所需的资源和依赖对象
		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	}
	catch (ClassNotFoundException ex) {
		error("Bean class [" + className + "] not found", ele, ex);
	}
	catch (NoClassDefFoundError err) {
		error("Class that bean class [" + className + "] depends on not found", ele, err);
	}
	catch (Throwable ex) {
		error("Unexpected failure during bean definition parsing", ele, ex);
	}
	finally {
		this.parseState.pop();
	}
	//解析<Bean>元素出错时，返回null
	return null;
}
```
>在Spring配置文件中<Bean>元素中配置的属性就是通过以上方法解析，设置到Bean中的。
>- 注意：在解析<Bean>元素过程中没有创建和实例化Bean对象，只是创建了Bean对象的定义类BeanDefinition，将<Bean>元素中的配置信息设置到BeanDefinition中作为记录，当依赖注入时才使用这些记录信息创建和实例化具体的Bean对象。  
上面方法中对一些配置如元信息(meta)、qualifier等的解析，使用的也不多，在使用Spring的<Bean>元素时，配置最多的是<property>属性，因此下面继续分析源码，了解Bean的属性在解析时是如何设置的。

##### 2.2.13 BeanDefinitionParserDelegate解析<property>元素    
>BeanDefinitionParserDelegate在解析<Bean>调用时调用parsePropertyElements方法解析<Bean>元素中的<property>属性子元素。

```java 
//解析<Bean>元素中的<property>子元素
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
	//获取<Bean>元素中所有的子元素
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		//如果子元素是<property>子元素，则调用解析<property>子元素方法解析
		if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
			parsePropertyElement((Element) node, bd);
		}
	}
}

//解析<property>元素
public void parsePropertyElement(Element ele, BeanDefinition bd) {
	//获取<property>元素的名字
	String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
	if (!StringUtils.hasLength(propertyName)) {
		error("Tag 'property' must have a 'name' attribute", ele);
		return;
	}
	this.parseState.push(new PropertyEntry(propertyName));
	try {
		//如果一个Bean中已经有同名的property存在，则不进行解析，直接返回。  
        //即如果在同一个Bean中配置同名的property，则只有第一个起作用
		if (bd.getPropertyValues().contains(propertyName)) {
			error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
			return;
		}
		//解析获取property的值
		Object val = parsePropertyValue(ele, bd, propertyName);
		//根据property的名字和值创建property实例
		PropertyValue pv = new PropertyValue(propertyName, val);
		//解析<property>元素中的属性
		parseMetaElements(ele, pv);
		pv.setSource(extractSource(ele));
		bd.getPropertyValues().addPropertyValue(pv);
	}
	finally {
		this.parseState.pop();
	}
}

//解析获取property值
public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
	String elementName = (propertyName != null) ?
					"<property> element for property '" + propertyName + "'" :
					"<constructor-arg> element";

	// Should only have one child element: ref, value, list, etc.
	//获取<property>的所有子元素，只能是其中一种类型:ref,value,list等
	NodeList nl = ele.getChildNodes();
	Element subElement = null;
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		//子元素不是description和meta属性
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
				!nodeNameEquals(node, META_ELEMENT)) {
			// Child element is what we're looking for.
			if (subElement != null) {
				error(elementName + " must not contain more than one sub-element", ele);
			}
			else {//当前<property>元素包含有子元素
				subElement = (Element) node;
			}
		}
	}

	//判断property的属性值是ref还是value，不允许既是ref又是value 
	boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
	boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
	if ((hasRefAttribute && hasValueAttribute) ||
			((hasRefAttribute || hasValueAttribute) && subElement != null)) {
		error(elementName +
				" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
	}

	//如果属性是ref，创建一个ref的数据对象RuntimeBeanReference
    //这个对象封装了ref信息
	if (hasRefAttribute) {
		String refName = ele.getAttribute(REF_ATTRIBUTE);
		if (!StringUtils.hasText(refName)) {
			error(elementName + " contains empty 'ref' attribute", ele);
		}
		//一个指向运行时所依赖对象的引用
		RuntimeBeanReference ref = new RuntimeBeanReference(refName);
		//设置这个ref的数据对象是被当前的property对象所引用
		ref.setSource(extractSource(ele));
		return ref;
	}
	//如果属性是value，创建一个value的数据对象TypedStringValue
    //这个对象封装了value信息
	else if (hasValueAttribute) {
		//一个持有String类型值的对象
		TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
		//设置这个value数据对象是被当前的property对象所引用
		valueHolder.setSource(extractSource(ele));
		return valueHolder;
	}
	//如果当前<property>元素还有子元素
	else if (subElement != null) {
		//解析<property>的子元素
		return parsePropertySubElement(subElement, bd);
	}
	else {
		// Neither child element nor "ref" or "value" attribute found.
		//propery属性中既不是ref，也不是value属性，解析出错返回null
		error(elementName + " must specify a ref or value", ele);
		return null;
	}
}
```
>在Spring配置文件中，<Bean>元素中<property>元素的相关配置的处理：  
a.ref被封装为指向依赖对象一个引用；  
b.value配置都会封装成一个字符串类型的对象；  
c.ref和value都通过“解析的数据类型属性值.setSource(extractSource(ele));”方法将属性值/引用与所引用的属性关联起来；
在方法的最后对于<property>元素的子元素通过parsePropertySubElement方法解析。

##### 2.2.14 解析<property>元素的子元素  
>在BeanDefinitionParserDelegate类中的parsePropertySubElement方法对<property>中的子元素解析:

```JAVA 
//解析<property>元素中ref,value或者集合等子元素
public Object parsePropertySubElement(Element ele, BeanDefinition bd) {
	return parsePropertySubElement(ele, bd, null);
}

public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
	//如果<property>没有使用Spring默认的命名空间，则使用用户自定义的规则解析
	//内嵌元素
	if (!isDefaultNamespace(ele)) {
		return parseNestedCustomElement(ele, bd);
	}
	//如果子元素是bean，则使用解析<Bean>元素的方法解析
	else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
		BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
		if (nestedBd != null) {
			nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
		}
		return nestedBd;
	}
	//如果子元素是ref，ref中只能有以下3个属性：bean、local、parent
	else if (nodeNameEquals(ele, REF_ELEMENT)) {
		// A generic reference to any name of any bean.
		//获取<property>元素中的bean属性值，引用其他解析的Bean的名称  
        //可以不再同一个Spring配置文件中，具体请参考Spring对ref的配置规则
		String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
		boolean toParent = false;
		if (!StringUtils.hasLength(refName)) {
			// A reference to the id of another bean in the same XML file.
			//获取<property>元素中的local属性值，引用同一个Xml文件中配置  
            //的Bean的id，local和ref不同，local只能引用同一个配置文件中的Bean
			refName = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
			if (!StringUtils.hasLength(refName)) {
				// A reference to the id of another bean in a parent context.
				//获取<property>元素中parent属性值，引用父级容器中的Bean
				refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
				toParent = true;
				
				if (!StringUtils.hasLength(refName)) {
					error("'bean', 'local' or 'parent' is required for <ref> element", ele);
					return null;
				}
			}
		}
		
		//没有配置ref的目标属性值 
		if (!StringUtils.hasText(refName)) {
			error("<ref> element contains empty target attribute", ele);
			return null;
		}
		//创建ref类型数据，指向被引用的对象
		RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
		//设置引用类型值是被当前子元素所引用
		ref.setSource(extractSource(ele));
		return ref;
	}
	//如果子元素是<idref>，使用解析ref元素的方法解析
	else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
		return parseIdRefElement(ele);
	}
	//如果子元素是<value>，使用解析value元素的方法解析
	else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
		return parseValueElement(ele, defaultValueType);
	}
	//如果子元素是null，为<property>设置一个封装null值的字符串数据
	else if (nodeNameEquals(ele, NULL_ELEMENT)) {
		// It's a distinguished null value. Let's wrap it in a TypedStringValue
		// object in order to preserve the source location.
		TypedStringValue nullHolder = new TypedStringValue(null);
		nullHolder.setSource(extractSource(ele));
		return nullHolder;
	}
	//如果子元素是<array>，使用解析array集合子元素的方法解析
	else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
		return parseArrayElement(ele, bd);
	}
	//如果子元素是<list>，使用解析list集合子元素的方法解析
	else if (nodeNameEquals(ele, LIST_ELEMENT)) {
		return parseListElement(ele, bd);
	}
	//如果子元素是<set>，使用解析set集合子元素的方法解析
	else if (nodeNameEquals(ele, SET_ELEMENT)) {
		return parseSetElement(ele, bd);
	}
	//如果子元素是<map>，使用解析map集合子元素的方法解析
	else if (nodeNameEquals(ele, MAP_ELEMENT)) {
		return parseMapElement(ele, bd);
	}
	//如果子元素是<props>，使用解析props集合子元素的方法解析
	else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
		return parsePropsElement(ele);
	}
	//既不是ref，又不是value，也不是集合，则子元素配置错误，返回null
	else {
		error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
		return null;
	}
}
```
>在Spring配置文件中，对<property>元素中配置的array、list、set、map、prop等各种集合子元素的都通过上述方法解析，生成对应的数据对象，比如ManagedList、ManagedArray、ManagedSet等，这些Managed类是Spring对象BeanDefiniton的数据封装，对集合数据类型的具体解析有各自的解析方法实现，解析方法的命名非常规范，一目了然，我们对<list>集合元素的解析方法进行源码分析，了解其实现过程。

##### 2.2.15 解析<list>子元素  
>在BeanDefinitionParserDelegate类中的parseListElement方法就是具体实现解析<property>元素中的<list>集合子元素:

```JAVA 
//解析<list>集合子元素
public List parseListElement(Element collectionEle, BeanDefinition bd) {
	//获取<list>元素中的value-type属性，即获取集合元素的数据类型
	String defaultElementType = collectionEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
	//获取<list>集合元素中的所有子节点
	NodeList nl = collectionEle.getChildNodes();
	//Spring中将List封装为ManagedList
	ManagedList<Object> target = new ManagedList<Object>(nl.getLength());
	target.setSource(extractSource(collectionEle));
	//设置集合目标数据类型
	target.setElementTypeName(defaultElementType);
	target.setMergeEnabled(parseMergeAttribute(collectionEle));
	//具体的<list>元素解析
	parseCollectionElements(nl, target, bd, defaultElementType);
	return target;
}

//具体解析<list>集合元素，<array>、<list>和<set>都使用该方法解析
protected void parseCollectionElements(
		NodeList elementNodes, Collection<Object> target, BeanDefinition bd, String defaultElementType) {
	//遍历集合所有节点
	for (int i = 0; i < elementNodes.getLength(); i++) {
		Node node = elementNodes.item(i);
		//节点不是description节点
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
			//将解析的元素加入集合中，递归调用下一个子元素 
			target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
		}
	}
}
```
>经过对SpringBean定义资源文件转换的Document对象中的元素层层解析，SpringIOC现在已将XML形式定义的Bean定义资源文件转换为SpringIOC所识别的数据结构——BeanDefinition，它是Bean定义资源文件中配置的POJO对象在SpringIOC容器中的映射，我们可以通过AbstractBeanDefinition为入口，看到了IOC容器进行索引、查询和操作。  
通过SpringIOC容器对Bean定义资源的解析后，IOC容器大致完成了管理Bean对象的准备工作，即初始化过程，但是最为重要的依赖注入还没发生，现在IOC容器中BeanDefinition存储的只是一些静态信息，接下来需要向容器注册Bean定义信息才能全部完成IOC容器的初始化过程，

##### 2.2.16 解析过后的BeanDefinition在IOC容器中的注册
>接下来分析DefaultBeanDefinitionDocumentReader对Bean定义转换的Document对象解析的流程中，在其parseDefaultElement方法中完成对Document对象的解析后得到封装BeanDefinition的BeanDefinitionHold对象，然后调用BeanDefinitionReaderUtils的registerBeanDefinition方法向IOC容器注册解析的Bean：

```java 
//将解析的BeanDefinitionHold注册到容器中
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	// Register bean definition under primary name.
	//获取解析的BeanDefinition的名称
	String beanName = definitionHolder.getBeanName();
	//向IOC容器注册BeanDefinition
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// Register aliases for bean name, if any.
	//如果解析的BeanDefinition有别名，向容器为其注册别名
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String aliase : aliases) {
			registry.registerAlias(beanName, aliase);
		}
	}
}
```
>当调用BeanDefinitionReaderUtils向IOC容器注册解析的BeanDefinition时，真正完成注册功能的是DefaultListableBeanFactory。

##### 2.2.17 DefaultListableBeanFactory向IOC容器注册解析后的BeanDefinition 

![image](https://github.com/nanphonfy/note-images/blob/master/promote-2019/source-analysis/04/SimpleAliasRegistry.png?raw=true)

>DefaultListableBeanFactory中使用一个HashMap的集合对象存放IOC容器中注册解析的BeanDefinition：

```java 
//向IoC容器注册解析的BeanDefinito
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");

	//校验解析的BeanDefiniton
	if (beanDefinition instanceof AbstractBeanDefinition) {
		try {
			((AbstractBeanDefinition) beanDefinition).validate();
		}catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,"Validation of bean definition failed", ex);
		}
	}

	//注册的过程中需要线程同步，以保证数据的一致性
	synchronized (this.beanDefinitionMap) {
		Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		
		//检查是否有同名的BeanDefinition已经在IOC容器中注册，如果已经注册，  
        //并且不允许覆盖已注册的Bean，则抛出注册失败异常
		if (oldBeanDefinition != null) {
			if (!this.allowBeanDefinitionOverriding) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName + "': There is already [" + oldBeanDefinition + "] bound.");
			}
			else {//如果允许覆盖，则同名的Bean，后注册的覆盖先注册的
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName + "': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
		}
		else {//IOC容器中没有已经注册同名的Bean，按正常注册流程注册
			this.beanDefinitionNames.add(beanName);
			this.frozenBeanDefinitionNames = null;
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	//重置所有已经注册过的BeanDefinition的缓存
	resetBeanDefinition(beanName);
}
```
>Bean定义资源文件中配置的Bean被解析过后，已注册到IOC容器中，被容器管理，真正完成IOC容器初始化所做的全部工作。现IOC容器已建立整个Bean的配置信息，这些BeanDefinition信息已可使用，且可被检索，IOC容器的作用是对这些注册的Bean定义信息进行处理和维护。这些的注册的Bean定义信息是IOC容器控制反转的基础，正是有了这些注册的数据，容器才可进行依赖注入。

