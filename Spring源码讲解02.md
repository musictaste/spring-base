[TOC]

# 回顾


## ClassPathXmlApplicationContext构造方法
```
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {

	super(parent); //创建一系列的对象
	setConfigLocations(configLocations); //设置配置文件的所存在的路径
	if (refresh) {
		refresh();
	}
}
```


## refresh方法（spring的核心）：bean工厂创建的过程，以及一些处理类


```
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//设置了启动时间、状态标志
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//创建BeanFactory,以及把xml文件的信息读取进来，并且注册成一堆BeanDefinition，放到DefaultListableBeanFactory中去
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//设置了一些具体的对象或类，方便以后做一些最基本的处理
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//是空的，留给子类自己实现
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//这一步昨天没讲，但是非常非常麻烦，看多以后 会发现代码不是非常难；现在我们已经把BeanDefinition注册到Map中了，
				//下一步就是调用反射，把这些Bean对象进行一些实例化的基本操作
				//（利用PostProcessor对BeanDefinition或Bean对象进行一些相关的处理，对应SPring架构图，蓝色部分，“对对象进行增强”BeanPostProccesor对对象进行处理；
				//所以在BeanFactoryPostProccesor上也有一些这样的方法，BeanFactoryPostProccesor也是一些规定好的接口，拿过来直接用）
				//*******这个方法一定要自己点一下，包含了Springboot的核心处理逻辑，ConfigurationClassPostProcessor这个类就是springboot注解识别的处理过程
				//总结就是：之前我们已经把BeanDefinition注册到Map中，现在呢调用反射，对这些Bean对象进行实例化，扫描Bean对象设置的注解并进行相应的处理（把设置的配置类识别进来，是对BeanFactory的一些增强，是springboot能够成功运行的核心所在，核心类就是ConfigurationClassPostProcessor），这块必须搞明白
				//如果ConfigurationClassPostProcessor这个类看不明白的话，可以到SPringboot启动源码解析中去学习，必须自己看了这个方法才能听懂
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//完成BeanPostProcessor的注册功能,这时候并没有执行，在调用我们的实例化对象的时候才会执行
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				//完成国际化的消息通知，一般不会问
				initMessageSource();

				// Initialize event multicaster for this context.
				//初始化事件监听器，之前说过sping用到了很多的装饰者模式，这个事件监听器就是
				//完成事件监听器的通知，不是核心
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				//默认什么都不做，留给子类自己实现
				onRefresh();

				// Check for listener beans and register them.
				//注意虽然spring中是空的，但是在springboot用的是比较多的
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//重点来了：实例化单例对象 -》 CGlib/调用反射newInstance()/循环依赖的解决
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				//重置公共缓存
				resetCommonCaches();
			}
		}
	}
```

### invokeBeanFactoryPostProcessors(beanFactory); 

```
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //现在还没有看到BeanFactoryPostProcessors相关的注册信息,所以getBeanFactoryPostProcessors()中size=0,size为空
    //查看invokeBeanFactoryPostProcessors
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```


#### PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors()); -》解析存在的BeanFactoryPostProcessor,分为两种情况，三种方式

解析存在的BeanFactoryPostProcessor,分为两种情况：BeanDefinitionREgisterPostProcessor、BeanFactoryPostProcessor；每种情况对应三种方式：PriorityOrdered 、 Ordered 、 NonOrdered

下来就是每种情况的每种方式具体的处理过程

只是带着梳理处理逻辑，细节需要自己看

如果觉着看的难受，可以到工程spring-framework-5.1.x，查看PostProcessorRegistrationDelegate类，invokeBeanFactoryPostProcessors方法，里面有写好的注释  -》 发现没有，再找找

```
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		//将我们已经执行过的PostProcessed放到set中，避免重复执行；set就是避免PostProcessors重复执行的集合
		Set<String> processedBeans = new HashSet<>();
        
        //***对BeanFactoryPostProcessor类、BeanDefinitionRegistryPostProcessor（子类）的处理---start*******
        
        //当前的BeanFactory是DefaultListableBeanFactory，通过查看show Diagrams类图结构，发现有子类BeanDefinitionRegistry继承DefaultListableBeanFactory接口
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			
			//两个集合，一定要记住；查看这两个类的类继承关系，BeanDefinitionRegistryPostProcessor继承了BeanFactoryPostProcessor这个接口，
			//所以下面这个集合registryProcessors就是上面集合regularPostProcessors的子类实现
			//现在map中已经有BeanDefinition了，
			//记住这两个集合的 名字：regularPostProcessors，registryProcessors里面放的东西是不一样的
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

            //遍历beanFactoryPostProcessors，但是现在没有BeanFactoryPostProcessor
    	    //但是要知道它的处理逻辑：根据是否是BeanDefinitionRegistryPostProcessor类，将BeanFactoryPostProcessor放到不同的集合中
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}


            //*********对BeanDefinitionRegistryPostProcessor(子类)的处理

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			//复杂的地方来了，又定了一个集合，存放BeanDefinitionRegistryPostProcessor，
			//为什么又创建一个集合呢？用来做什么呢？存放当前正在处理的BeanDefinitionRegistryPostProcessor
    		//先处理子类实现中最核心的代码
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			//BeanDefinition的map里面有18个，对这个18个BeanDefinition进行过滤，查看getBeanNamesForType方法-》 doGetBeanNamesForType () -> getMergedLocalBeanDefinition(返回一个RootBeanDefinition，就是返回一个根上来的BeanDefinition，包含对应的父子继承关系) + isFactoryBean(Spring有BeanFactory和FactoryBean, FactoryBean里面有三个方法：getObject（可以在new 对象的时候，在前后中间，加上自己的逻辑代码-》对应Spring架构图中的“普通对象”FactoryBean产生一个对象getObject），getObjectTye、isSingleton)
    		//符合要求的只有internalConfigurationAnnotationProcessor
    		//一句话表述：查看已经有的BeanDefinition是否属于BeanDefinitionRegistryPostProcessor这个类，如果属于就放到数组中，18个BeanDefinition只有一个匹配
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			//下面循环的作用：从我们注册进去的BeanDefinition里面来获取到属于BeanDefinitionRegistryPostProcessor这些类，找到这些类，然后对这些类进行相关处理，把处理类进行执行，但是现在只有找到ConfigurationClassPostProcessor，直接把这个类进行调用处理就可以了 
			for (String ppName : postProcessorNames) {
			    //PriorityOrdered优先排序的类，是什么意思呢？ 来查看AnnotationConfigUtils,第一个属性：CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME，这个类中查找，找到ConfigurationClassPostProcessor,查看这个类的实现类图，发现实现PriorityOrdered接口
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				    //添加到currentRegistryProcessors，当前正在处理的BeanDefinitionRegistryPostProcessor,集合中只有一个元素
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					//添加到processedBeans(已经处理过的PostProcessed)，集合中也是只有一个元素
					processedBeans.add(ppName);
				}
			}
			//我们会写atOrder,对PostProcessors进行排序，方面逻辑也比较简单，取到getDependencyComparator默认的排序器，然后进行排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			//添加到集合registryProcessors
			registryProcessors.addAll(currentRegistryProcessors);
			//执行的是 当前正在处理的currentRegistryProcessors集合中的元素，方法内的逻辑是：通过系统方法System.identityHashCode得到哈希code值registryId，-> processConfigBeanDefinitions(属于ConfigurationClassPostProcessor，这个类很重要，在AnnotationgConfigUtils这个类中有属性ConfigurationClassPostProcessor这个类，ConfigurationClassPostProcessor有ProcessConfigBeanDefinitions具体的对象，拿过来进行具体的处理 -》 register是谁呢？是DefaultListableBeanFactory,查看类继承关系里面有BeanDefinitionRegistry -> registry.getBeanDefinitionNames现在有18个 )
    		//在spring的源码中有很多set集合、map集合，起到缓存的效果，先看一下有没有这个对象，如果有拿过来直接用；如果没有，重新再走一遍流程得到这些对象
    		//集合的作用就是提交运行效率
    		//方法的细节文档在下面
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			
			//跟上面的代码一样，postProcessorNames也只有一个，是internalConfigurationAnnotationProcessor
			//为什么要执行两次呢？两次遍历是不一样的，一个是Ordered，一个是PriorityOrdered；先执行PriorityOrdered子类，子类执行级高
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
			    //processedBeans是为了防止重复执行的，所以下面这个循环不用执行
			    //这块是Ordered是父类，PriorityOrdered是子类
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			//currentRegistryProcessors为空，所以不执行
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false; 
				postProcessorNames =  beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				//postProcessorNames还是internalConfigurationAnnotationProcessor
				for (String ppName : postProcessorNames) {
				    //processedBeans已经有这个类，所以不执行
				    //如果PostProcessor既没有实现Ordered，也没有实现PriorityOrdered,那么进行处理，相当于三层过滤
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}
			
			
            //***对Bean的处理(只是添加)---start*******
			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			//这两个方法是比较麻烦的，传入的参数不一样的
			//registryProcessors里面是ConfigurationClassPostProcessor类
			//regularPostProcessors为空
			//invokeBeanFactoryPostProcessors -> postProcessBeanFactory(通过identityHashCode计算哈希code得到factoryId,上面的是registerId是不一样的)，然后往beanFactory中添加BeanPostProcessor(之前是没有往beanFactory中添加过BeanPostProcessor的，之前只是注册了一些BeanDefinition的map集合，这些BeanDefinition不是实例化的对象)，现在开始往beanFactory中注册ImportAwareBeanPostProcessor对象【刚刚在外层处理的是BeanFactoryPostProcessor，现在是对BeanPostProcessor的处理，是Bean的增强类】，【注意这块只是添加了BeanPostProcessor，并没有进行执行】
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

        //*****对BeanFactoryPostProcessor的处理

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		
		//postProcessorNames中有internalConfigurationAnnotationProcessor，还有internalEventListenerProcessor(处理监听)、PropertySourcesPlaceholdeerConfigurer(处理资源文件)两个
		//到AnnotationConfigurationUtils中找internalEventListenerProcessor，对应的是EventListenerMethodProcessor类，查看该类的类图,没有实现Order相关的接口
		//到AnnotationConfigurationUtils中找PropertySourcesPlaceholdeerConfigurer，对应的是PropertySourcesPlaceholderConfigurer类，查看该类的类图,实现了相关的PriorityOrdered接口
		
		
		//分别为优先排序、排序的、不排序的
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();//有一个值：PropertySourcesPlaceholdeerConfigurer
		List<String> orderedPostProcessorNames = new ArrayList<>();//没有值
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();//有一个值：internalEventListenerProcessor
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase abov e
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		//自己查看processProperties的处理过程
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);
        
        
        //没有数据，不执行
		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);


        
		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		//自己看处理过程
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```

#### doGetBeanNamesForType

```
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
	List<String> result = new ArrayList<>();

	// Check all bean definitions.
	for (String beanName : this.beanDefinitionNames) {
		// Only consider bean as eligible if the bean name
		// is not defined as alias for some other bean.
		if (!isAlias(beanName)) {
			try {
			    //返回一个RootBeanDefinition，就是返回一个根上来的BeanDefinition，包含对应的父子继承关系
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// Only check bean definition if it is complete.
				if (!mbd.isAbstract() && (allowEagerInit ||
						(mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading()) &&
								!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
					//判断是否为FactoryBean
					boolean isFactoryBean = isFactoryBean(beanName, mbd);
					//获取包装好的BeanDefinition，这个东西是啥？包装好的RootBeanDefinition
					BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
					boolean matchFound = false;
					boolean allowFactoryBeanInit = allowEagerInit || containsSingleton(beanName);
					boolean isNonLazyDecorated = dbd != null && !mbd.isLazyInit();
					if (!isFactoryBean) {
						if (includeNonSingletons || isSingleton(beanName, mbd, dbd)) {
							matchFound = isTypeMatch(beanName, type, allowFactoryBeanInit);
						}
					}
					else  {
						if (includeNonSingletons || isNonLazyDecorated ||
								(allowFactoryBeanInit && isSingleton(beanName, mbd, dbd))) {
							matchFound = isTypeMatch(beanName, type, allowFactoryBeanInit);
						}
						if (!matchFound) {
							// In case of FactoryBean, try to match FactoryBean instance itself next.
							beanName = FACTORY_BEAN_PREFIX + beanName;
							matchFound = isTypeMatch(beanName, type, allowFactoryBeanInit);
						}
					}
					
					//判断当前集合中的BeanDefinition是不是我们要求的类，18个类只有一个符合，就是internalConfigurationAnnotationProcessor
					if (matchFound) {
						result.add(beanName);
					}
				}
			}
			catch (CannotLoadBeanClassException | BeanDefinitionStoreException ex) {
				if (allowEagerInit) {
					throw ex;
				}
				// Probably a placeholder: let's ignore it for type matching purposes.
				LogMessage message = (ex instanceof CannotLoadBeanClassException) ?
						LogMessage.format("Ignoring bean class loading failure for bean '%s'", beanName) :
						LogMessage.format("Ignoring unresolvable metadata in bean definition '%s'", beanName);
				logger.trace(message, ex);
				onSuppressedException(ex);
			}
		}
	}

    //再次检查，manualSingletonNames，之前没有细看，之前会有一个通用的singleton的单例实例名称，包含了三个属性名称：environment、systemProperties，systemEnvironment
    //没有匹配的BeanDefiniton
	// Check manually registered singletons too.
	for (String beanName : this.manualSingletonNames) {
		try {
			// In case of FactoryBean, match object created by FactoryBean.
			if (isFactoryBean(beanName)) {
				if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
					result.add(beanName);
					// Match found for this bean: do not match FactoryBean itself anymore.
					continue;
				}
				// In case of FactoryBean, try to match FactoryBean itself next.
				beanName = FACTORY_BEAN_PREFIX + beanName;
			}
			// Match raw bean instance (might be raw FactoryBean).
			if (isTypeMatch(beanName, type)) {
				result.add(beanName);
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Shouldn't happen - probably a result of circular reference resolution...
			logger.trace(LogMessage.format("Failed to check manually registered singleton with name '%s'", beanName), ex);
		}
	}
    
    //最后只有result中的一个internalConfigurationAnnotationProcessor
	return StringUtils.toStringArray(result);
}
```
	
#### getMergedLocalBeanDefinition	
    	
```
//Return a merged RootBeanDefinition, traversing the parent bean definition,if the specified bean corresponds to a child bean definition.
//返回一个RootBeanDefinition，就是返回一个根上来的BeanDefinition，包含对应的父子继承关系
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
		// Quick check on the concurrent map first, with minimal locking.
		RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
		if (mbd != null && !mbd.stale) {
			return mbd;
		}
		//这个就不要看了，容易看懵
		return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}
```


#### processConfigBeanDefinitions -》 处理ConfigurationClassPostProcessor类

processConfigBeanDefinitions(属于ConfigurationClassPostProcessor，这个类很重要，在AnnotationgConfigUtils这个类中有属性ConfigurationClassPostProcessor这个类，ConfigurationClassPostProcessor有ProcessConfigBeanDefinitions具体的对象，拿过来进行具体的处理 
```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
		    //register是谁呢？是DefaultListableBeanFactory,查看类继承关系里面有BeanDefinitionRegistry 
		    //根据名称来进行获取，现在有18个
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			
			//查看是否有参数：configurationClass
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			//metadataReaderFactory=CachingMetadataReaderFactory
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			    //往集合configCandidates添加具体的对象BeanDefinitionHolder
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		//有4个符合的对象，分别是：bookDao/empDao/bookService/multService,说明可以识别 类上加了相关的注解：@Service，@Repository 或者 通过配置文件配置的<bean>  或者内部的类
		if (configCandidates.isEmpty()) {
			return;
		}

		// Sort by previously determined @Order value, if applicable
		//排序
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

		// Detect any custom bean name generation strategy supplied through the enclosing application context
		//单例bean的注册
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
						AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					this.componentScanBeanNameGenerator = generator;
					this.importBeanNameGenerator = generator;
				}
			}
		}

		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}

		// Parse each @Configuration class
		//解析@Configuration配置类
		//@Configuration这个注解 -》 @AliasFor(annotation = Component.class) -》 @Configuration在Springboot中经常用，在spring中使用比较少
		//SpringBoot中@SpringBootApplication -》 @SpringBootConfiguration -》 @Configuration -》 
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);
        
        //包含4个元素
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
		
		    //查看BeanDefinitionHolder是否为AnnotatedBeanDefinition，如果是进行解析，在doProcessConfigurationClass方法(带do的都是实际干活的)中，判断是否为@Component注解、有没有@PropertySources注解、有没有@ComponentScans注解、@Import、@ImportSource、@Bean，
		    //doProcessConfigurationClass这一步到底是干嘛呢？我们现在有一堆的BeanDefinition（来源有使用了注解、xml配置、内部使用的）；我要查看使用了注解的BeanDefinition是否包含了@Configuration，如果包含了，接着查看是否有其他属性的引入- 》 引入是啥意思？ 在Springboot中 @SpringBootApplication，还有别的其他的注解，会把这些注解依赖的其他注解以及依赖的bean对象进行一个整体的解析-》理解这个便于理解Springboot源码
		    //如果还不能理解，可以查看工程：spring_annotation_config，查看具体的配置方式
            //原来我们在使用配置的时候，第一种是通过xml配置文件；后来为了操作方便使用了@Annotation这种方式；第三种是使用配置类，可以不使用配置文件，直接使用配置类
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			//将已经解析的类移除
			configClasses.removeAll(alreadyParsed);

            //后面的不需要细看了，直接返回
			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			// Clear cache in externally provided MetadataReaderFactory; this is a no-op
			// for a shared cache since it'll be cleared by the ApplicationContext.
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}
```


#### doProcessConfigurationClass

//doProcessConfigurationClass这一步到底是干嘛呢？我们现在有一堆的BeanDefinition（来源有使用了注解、xml配置、内部使用的）；我要查看使用了注解的BeanDefinition是否包含了@Configuration，如果包含了，接着查看是否有其他属性的引入- 》 引入是啥意思？ 

在Springboot中 @SpringBootApplication，还有别的其他的注解，会把这些注解依赖的其他注解以及依赖的bean对象进行一个整体的解析-》理解这个便于理解Springboot源码

//如果还不能理解，可以查看工程：spring_annotation_config，查看具体的配置方式

//原来我们在使用配置的时候，第一种是通过xml配置文件；后来为了操作方便使用了@Annotation这种方式；第三种是使用配置类，可以不使用配置文件，直接使用配置类

```
@Nullable
	protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {

		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass);
		}

		// Process any @PropertySource annotations
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// Process any @Import annotations
		//具体的解析过程不看了，可以查看springboot的源码，里面有具体的解析过程
		processImports(configClass, sourceClass, getImports(sourceClass), true);

		// Process any @ImportResource annotations
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}

		// No superclass -> processing is complete
		return null;
	}
```

#### 源码中的注解是怎么解析的

上面的源码中已经讲了

举一个常见的例子，在学反射的时候，获取到Class对象，有一个getAnnotations方法，能获取到你当前类里面的注解，拿到这些注解可以做一些识别和处理， 这是反射底层做的

JDK中提供了四个元注解，@Target   @Retention   @Documented   @Component

RententionPolicy有Source（源代码层面）、Class（字节码层面）、Runtime(运行时的)

javase的时候讲过这个注解

![201](6259EBA8DAAE4DE2B59147A2499C398C)


#### postProcessBeanFactory（ConfigurationClassPostProcessor）

postProcessBeanFactory()，然后
```
/**
	 * Prepare the Configuration classes for servicing bean requests at runtime
	 * by replacing them with CGLIB-enhanced subclasses.
	 */
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		//通过identityHashCode计算哈希code得到factoryId,上面的是registerId是不一样的
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}

		enhanceConfigurationClasses(beanFactory);
		//往beanFactory中添加BeanPostProcessor(之前是没有往beanFactory中添加过BeanPostProcessor的，之前只是注册了一些BeanDefinition的map集合，这些BeanDefinition不是实例化的对象)，
		//现在开始往beanFactory中注册ImportAwareBeanPostProcessor对象【刚刚在外层处理的是BeanFactoryPostProcessor，现在是对BeanPostProcessor的处理，是Bean的增强类】
		//【注意这块只是添加了BeanPostProcessor，并没有进行执行】
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
```


## registerBeanPostProcessors

完成了什么功能？完成BeanPostProcessor的注册功能,这时候并没有执行，在调用我们的实例化对象的时候才会执行

```
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
        
        //getBeanNamesForType是不是很熟悉，之前是BeanDefinitionRegistryPostProcessor或BeanFactoryPostProcessor,现在是BeanPostProcessor
		String[] postProcessorNames =
		//值由三个：internalAutoWiredAnnotationProcessor、internalCommonAnntationProcessor、internalAutoProxyCreator
		//之前我们常说spring的aop是基于动态代理实现的，那么它是基于哪部分实现的呢？它是基于BeanPostProcessor实现的，不是基于BeanFactoryPostProcessor实现的，这块一定要描述清楚了
		beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		//添加了一个BeanPostProcessorChecker检查器，查看这个类（作用是当BeanProcessor初始化的时候记录一些日志信息）
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		//又看着这些东西，熟不熟；在处理BeanFactoryPostProcessor的时候
		//PriorityOrdered、Ordered、nonOrdered
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				//MergedBeanDefinitionPostProcessor之前也见过，跟Root相关，查看这个类的注释：Post-processor callback interface for <i>merged</i> bean definitions at runtime.回调方法在运行时合并了beanDefinition
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		//处理过程，beanPostProcessors中有7个，第五个是CommonAnnotationBeanPostProcessor,之前4个为：ApplicationContextAwareProcessor、applicationListenerDetector、ImportAwareBeanPostProcessor、BeanPostProcessorChecker;
		//ApplicationContextAwareProcessor、applicationListenerDetector之前代码加进来的
		//ImportAwareBeanPostProcessor是BeanFactory中加进来的
		//BeanPostProcessorChecker上面加进来的
		//逻辑过程：删除一个，再添加一个，双重检验的过程
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		//nonOrderedPostProcessors为空
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
		
		//最后还是7个
	}
```

## initMessageSource()

完成国际化的消息通知，一般不会问


```
/**
	 * Initialize the MessageSource.
	 * Use parent's if none defined in this context.
	 */
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//MESSAGE_SOURCE_BEAN_NAME=messageSource
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			//新建一个DelegatingMessageSource
			DelegatingMessageSource dms = new DelegatingMessageSource();
			//设置父类的MessageSource
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			//注册为一个Singleton对象
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}
```

## initApplicationEventMulticaster

完成事件监听器的通知

```
/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//APPLICATION_EVENT_MULTICASTER_BEAN_NAME=applicationEventMulticaster
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
		    //创建一个事件监听器，并注册为单例
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

## finishBeanFactoryInitialization


```
/**
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		//设置为空的ClassLoader
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		//难点来了，实例化剩余的单例对象
		beanFactory.preInstantiateSingletons();
	}
```

### beanFactory.preInstantiateSingletons()


```
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		//所有的BeanDefinition名称，一共是18个
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
		    //又看到了
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			//不是抽象的、不是单例的、不是懒加载的
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
				    //难点
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		//bean创建之前，要提前做哪些初始化的工作
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

#### getBean -》 doGetBean

加了一系列复杂的逻辑判断，参数判断，实际就是通过反射.newInstance();

因为是框架需要考虑的东西比较多，实际我们自己做就是Proxy.newInstance()

```
@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```


```
/**
	 * Return an instance, which may be shared or independent, of the specified bean.
	 * @param name the name of the bean to retrieve
	 * @param requiredType the required type of the bean to retrieve
	 * @param args arguments to use when creating a bean instance using explicit arguments
	 * (only applied when creating a new instance as opposed to retrieving an existing one)
	 * @param typeCheckOnly whether the instance is obtained for a type check,
	 * not for actual use
	 * @return an instance of the bean
	 * @throws BeansException if the bean could not be created
	 */
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
        
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			//是否为多例实现
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
			    //标记该bean是创建完成的
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				//指定dependOn，优先执行
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				//判断是否为Singleton对象
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
						    //doCreateBean中断电，查看BookDao的创建过程
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

##### createBean


```
@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
		    //准备覆盖的方法
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance. 返回目标类的代理类
			//实例对象初始化之前，要处理的基本操作
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
		    //doCreateBean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```


##### doCreateBean  -》Eagerly cache singletons能够帮我们解决setter方式的循环依赖问题



```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
		    //判断是否为单例的，如果是，将缓存中的实例移除掉
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
		    //
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references；Eagerly cache singletons能够处理我们的循环引用问题
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//跳过中间的步骤，到达这,
		//Spring中非常恶心的东西，循环引用，A引用B,B引用A，那么应该在创建具体对象时应该先创建谁，后创建谁？
		//创建对象有两种注入的方式，一种是构造器注入，一种是setter注入；如果是构造器注入那么解决不了；如果是setter注入的话，在spring中有一个预暴露Exposure，提前暴露出来，解决循环依赖/引用的问题
		//***************Eagerly cache singletons能够帮我们解决setter方式的循环依赖问题
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```


##### createBeanInstance


```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
		    //是否自动装配
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		//构造器处理
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		//如果没有定义构造方法，调用默认的构造方法
		return instantiateBean(beanName, mbd);
	}
```

##### instantiateBean


```
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
			    //返回一个策略
			    //getInstantiationStrategy返回一个Cglib开头的策略，
			    //instantiate
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

##### instantiate


```
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			//加锁
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
				    //getBeanClass返回Class，已经运用了反射
					final Class<?> clazz = bd.getBeanClass();
					//BookDao不是接口
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
						    //返回定义好的构造器
							constructorToUse = clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			//又改实例化了
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```


##### BeanUtils.instantiateClass


```
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		Assert.notNull(ctor, "Constructor must not be null");
		try {
			ReflectionUtils.makeAccessible(ctor);
			if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
				return KotlinDelegate.instantiateClass(ctor, args);
			}
			else {
				Class<?>[] parameterTypes = ctor.getParameterTypes();
				Assert.isTrue(args.length <= parameterTypes.length, "Can't specify more arguments than constructor parameters");
				Object[] argsWithDefaultValues = new Object[args.length];
				for (int i = 0 ; i < args.length; i++) {
					if (args[i] == null) {
						Class<?> parameterType = parameterTypes[i];
						argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
					}
					else {
						argsWithDefaultValues[i] = args[i];
					}
				}
				//熟不熟？使用反射时，newInstance;
				//虽然之前的代码看起来很繁琐，但是实际就是运用反射来生成一个基本对象
				return ctor.newInstance(argsWithDefaultValues);
			}
		}
		catch (InstantiationException ex) {
			throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
		}
		catch (IllegalAccessException ex) {
			throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
		}
		catch (IllegalArgumentException ex) {
			throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
		}
		catch (InvocationTargetException ex) {
			throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
		}
	}
```

## finishRefresh


```
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		//清除缓存
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		//初始化生命周期的操作
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```


# 一系列的Aware接口是在哪里作用的 -》 BeanFactory

BeanFactory 最上面有一系列Aware的接口实现，注释中有初始化过程


```
* <p>Bean factory implementations should support the standard bean lifecycle interfaces
 * as far as possible. The full set of initialization methods and their standard order is:
 * <ol>
 * <li>BeanNameAware's {@code setBeanName}
 * <li>BeanClassLoaderAware's {@code setBeanClassLoader}
 * <li>BeanFactoryAware's {@code setBeanFactory}
 * <li>EnvironmentAware's {@code setEnvironment}
 * <li>EmbeddedValueResolverAware's {@code setEmbeddedValueResolver}
 * <li>ResourceLoaderAware's {@code setResourceLoader}
 * (only applicable when running in an application context)
 * <li>ApplicationEventPublisherAware's {@code setApplicationEventPublisher}
 * (only applicable when running in an application context)
 * <li>MessageSourceAware's {@code setMessageSource}
 * (only applicable when running in an application context)
 * <li>ApplicationContextAware's {@code setApplicationContext}
 * (only applicable when running in an application context)
 * <li>ServletContextAware's {@code setServletContext}
 * (only applicable when running in a web application context)
 * <li>{@code postProcessBeforeInitialization} methods of BeanPostProcessors
 * <li>InitializingBean's {@code afterPropertiesSet}
 * <li>a custom init-method definition
 * <li>{@code postProcessAfterInitialization} methods of BeanPostProcessors
 * </ol>
```
