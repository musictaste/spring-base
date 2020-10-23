# Spring源码讲解
[TOC]


![spring架构](6E91C5068DF14911B1131F5E27781A12)

![Spring IOC的初始化过程](A416523E76DF44AC8C7B8BA52D879C7D)

## new ClassPathXmlApplicationContext


```
super(parent);  父类的构造方法，获取了资源的处理器getResourcePatternResolver
    
setConfigLocations(configLocations); 加载配置文件
    
refresh();  处理配置文件
```


## AbstractApplicationContext.refresh()

    synchronized (this.startupShutdownMonitor) { 开启了同步，
        // Prepare this context for refreshing.
		prepareRefresh();
		obtainFreshBeanFactory();
    }  

### AbstractApplicationContext.prepareRefresh()
    
    //设置IOC容器的状态
    this.closed.set(false);
	this.active.set(true);
	
	initPropertySources(); //初始化了一些为子类扩展的属性

    getEnvironment().validateRequiredProperties();
        environment对象，两个属性值：systemInvoment   systemProterties
    
    LinkedHashSet earlyApplicationListeners 存储监听事件

### AbstractApplicationContext.obtainFreshBeanFactory
    
    自己查看BeanFactory类
    
    安装翻译软件：translate，右键有translate选项，选中一段进行翻译
    

```
refreshBeanFactory();
return getBeanFactory();
```



#### AbstractRefreshableApplicationContext.refreshBeanFactory()

    
```
@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {  //如果有BeanFactory，销毁BeanFactory
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();  //DefaultListableBeanFactory上场喽 
			//Set<Class<?>> ignoredDependencyInterfaces -> 忽略的接口：BeanNameAware/BeanFactoryAware/BeanClassLoaderAware.class
			
			//aware的作用，将当前对象注册到aware中，方便以后调用；BeanFactory的注解中也有很多Aware的初始化
			
			
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);  //定制化的BeanFactory
			loadBeanDefinitions(beanFactory);  //关键方法
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

##### AbstractXmlApplicationContext.loadBeanDefinitions()

对照spring架构图，发现xml文件通过BeanDefinitionReader将信息读取进来

```
@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);  //设置了一些属性值，不重要

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		
		//this指当前类本身，当前类为：AbstractXmlApplicationContext
		beanDefinitionReader.setResourceLoader(this);
		
		//ResourceEntityResolver:用于解析xml文件，先创建对象
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		
		//初始化BeanDefinitionReader
		//XML Schema 语言也称作 XML Schema 定义（XML Schema Definition，XSD）;基于 XML 的 DTD 替代者;描述 XML 文档的结构
		initBeanDefinitionReader(beanDefinitionReader);
		
		//这个方法比较重要，加一个断电
		loadBeanDefinitions(beanDefinitionReader);
	}
```

###### loadBeanDefinitions(AbstractXmlApplicationContext.class)-很重要


```
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		
		//spring初始化时，指定了new ClassPathXmlApplicationContext("applicationContext.xml"); ->setConfigLocations()

		String[] configLocations = getConfigLocations(); //applicationContext.xml
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);//代码在下面
		}
	}
```

###### loadBeanDefinitions


```
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);   //具体为：PathMatchingResourcePatternResolver
				//值为：applicationContext.xml
				//**仅仅是将applicationContext.xml路径映射为Resource对象
				
				//参数是：Resoucrce对象 -> EncodedResource  
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}
```

###### loadBeanDefinitions(EncodedResource)(XmlBeanDefinitionReader.class)


```
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
		    //通过流将文件读入
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				
				//加断点，doXXX：是实际干活的
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

###### doLoadBeanDefinitions


```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
		    //按照文档格式进行读取  -> laodDocument()  -> getValidationModeForResource()  -> detectValidationMode() 验证这个配置文件是什么样的规范要求
		    
		    //laodDocument()  -> createDocumentBuilderFactory()--DocumentBuilder:可以拿到xml文件中的详细信息了  -> bulider.parse()  得到Document对象
		    
		    //可以debug查看Document的对象的内容
		    //将xml文件的内容读取到Document对象中，但是现在还没有做具体的属性解析
			Document doc = doLoadDocument(inputSource, resource);
			
			//这个方法很重要
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

###### -loadDocument(DefaultDocumentLoader.class)


```
//按照文档格式进行读取  -> laodDocument()  -> getValidationModeForResource()  -> detectValidationMode() 验证这个配置文件是什么样的规范要求
		    
//laodDocument()  -> createDocumentBuilderFactory()--DocumentBuilder:可以拿到xml文件中的详细信息了  -> bulider.parse()  得到Document对象

public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

        //DocumentBuilderFactory
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isTraceEnabled()) {
			logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		
		//DocumentBuilder
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		
		//  parese()  ->domParser.parse(is);  可以看到SAXException  -> XML11Configuration.parse()
		//SAX:simple API for XML,是一种XML解析的替代方法。相比于DOM，SAX是一种速度更快，更有效的方法
		return builder.parse(inputSource);  //返回Document对象
	}

```

###### 	-registerBeanDefinitions


```
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	    //BeanDefinitionDocumentReader对象，之前是BeanDefinitionReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		
		//countBefore=0
		int countBefore = getRegistry().getBeanDefinitionCount();
		
		//createReaderContext(resource)得到XmlReaderContext对象
		//XmlReaderContext类的注解  Handler  
		//联想到：spring.handlers  (context jar包的META-INF)，存放了要加载的handler类
		
		//createReaderContext() -> DefaultNamespaceHandlerResolver类，指定了"META-INF/spring.handlers"
		
		//DefaultNamespaceHandlerResolver类得到对应jar包的命名空间的handlers
		
		
		//registerBeanDefinitions  -> doRegisterBeanDefinitions() 根据给定的<beans/>，注册beanDefinition ->createDelegate() -> BeanDefinitionParserDelegate 跟document对象匹配，得到具体的属性值
		documentReader.registerBeanDefinitions(doc, 
		createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

###### --doRegisterBeanDefinitions
    
    //registerBeanDefinitions  -> doRegisterBeanDefinitions() 根据给定的<beans/>，注册beanDefinition
    ->createDelegate() -> BeanDefinitionParserDelegate 跟document对象匹配，得到具体的属性值
    
    ->parseBeanDefinitions

```
/**
 * Register each bean definition within the given root {@code <beans/>} element.
 */
protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		
		//BeanDefinitionParserDelegate跟document对象匹配，得到具体的属性值
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		
		//
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
```

###### ---parseBeanDefinitions


```
/**
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
					    
					    //默认的，例如对应xml文件的<beans>
						parseDefaultElement(ele, delegate);
					}
					else {
					
					    //定制化的 例如：对应xml文件的bean、context:component-scan
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```


###### ----parseCustomElement


```
/**
	 * Parse a custom element (outside of the default namespace).
	 * @param ele the element to parse
	 * @param containingBd the containing bean definition (if any)
	 * @return the resulting bean definition
	 */
	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		
		//听到这了。。。TODO
		
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		
		
		//
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```



----


BeanDefinitionReaderUtils

    registerBeanDefinition(definitionHolder,this.registry);

DefaultListableBeanFactory  对当前类做一些基本的注册

    Map<>  beanDefinitionMap  这时bean没有实例化
    List<String> beanDefinitionNames

registerComponents 注册组件

AnnotationConfigUtils
    类似：org.springframework.context.annotation.internalConfigurationAnnotationProcessor
    里面的类，在jar包中找不到，需要执行的时候才能找到
    
    定义RootBeanDefinition  ->  ConfigurationClassPostProcessor增强类
    
    作用：定义RootBeanDefinition，只有通过AnnotationConfigUtils中的属性，才能找到对应的增强类
    
DefaultBeanDefinitionDocumentReader
    
    doRegisterBeanDefinitions()
        preProcessXml
        parseBeanDefinitions()
        postProcessXml

    
### AbstractApplicationContext.refresh()

    理解bean初始化过程，bean的整个生命周期、
    
#### 1.obtainFreshBeanFactory()    

##### AbstractRefreshableApplicationContext.refreshBeanFactory()

    refresh() ->  obtainFreshBeanFactory() -> AbstractRefreshableApplicationContext.refreshBeanFactory() 加断点

```
DefaultListableBeanFactory beanFactory = createBeanFactory();
beanFactory.setSerializationId(getId());
customizeBeanFactory(beanFactory);
loadBeanDefinitions(beanFactory);
synchronized (this.beanFactoryMonitor) {
	this.beanFactory = beanFactory;
}

```

    此方法有以下几个操作：
        1、createBeanFactory方法会创建一个DefaultListableBeanFactory对象
        2、customizeBeanFactory此方法是留给子类实现的，推荐通过overwrite的方式对现有的BeanFactory做个性化设置：
            allowBeanDefinitionOverriding表示是否允许注册一个同名的类来覆盖原有类，allowCircularReferences表示是否运行多个类之间的循环引用
        3、loadBeanDefinitions把所有bean的定义后保存在context注意是bean的定义，而不是实例化
        
     这一步就是《spring架构》将“bean信息”加载到“容器”中
     
##### ==描述下Spring IOC容器的初始化过程==

​		Spring IOC容器的初始化简单的可以分为三个过程：

​		第一个过程是Resource资源定位。这个Resouce指的是BeanDefinition的资源定位。这个过程就是容器找数据的过程，就像水桶装水需要先找到水一样。

    xml文件资源的定位

​		第二个过程是BeanDefinition的载入过程。这个载入过程是把用户定义好的Bean表示成Ioc容器内部的数据结构，而这个容器内部的数据结构就是BeanDefition。

    读取文件变成一个document对象；把document对象解析成我们的一些标签(可以识别的东西)，例如component-scan、bean,再封装到BeanDefition中

​		第三个过程是向IOC容器注册这些BeanDefinition的过程，这个过程就是将前面的BeanDefition保存到HashMap中的过程。

    将BeanDefition保存到Map中
            
不需要死记硬背，debug看源码
    
    
#### 2.prepareBeanFactory

    往对象中注册了某些属性值或对象值，方便以后调用

    Configure the factory's standard context characteristics, such as the context's ClassLoader and post-processors.
    ClassLoader:类加载器
    post-processors：类增强器
    
    
    setBeanClassLoader
    
    setBeanExpressionResolver  -> StandardBeanExpressionResolver  -> spring EL(SPEL) 表达式->  应用在AOP
    
    addBeanPostProcessor  -> ApplicationContextAwareProcessor  implements BeanPostProcessor(对对象进行增强)
        对应《spring架构》的BeanPostProcessor(对对象进行增强)
        
        BeanPostProcessor就是容器对象
        我们加载进来的bean信息，生成的普通对象
        bean信息通过各种各种的BeanPostProcessor，变成普通对象
        
    ignoreDependencyInterface  忽略的依赖接口
    
    registerResolvableDependency
    
    addBeanPostProcessor -> ApplicationListenerDetector
    
    
    
#### 3.invokeBeanFactoryPostProcessors

    执行BeanFactoryPostProcessors相关的方法