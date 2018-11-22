# IOC容器创建bean过程

- 缓存中获取单例实例 获取到直接返回

- 递归调用父容器获取bean方法

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

     - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation ==如果方法返回的不为null则创建对象完毕==
       - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation 调用 InstantiationAwareBeanPostProcessor接口实现类的postProcessBeforeInstantiation方法进行直接返回bean实例 如果有一个返回不为空直接结束 
       - 如果上列操作完毕都返回的不是null 则调用所有的BeanPostProcessor接口实现类的postProcessAfterInitialization方法进行直接返回bean实例 如果有一个返回不为空直接结束 

- 调用org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance进行bean的实例化

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors 进行所有MergedBeanDefinitionPostProcessor接口实现类的postProcessMergedBeanDefinition方法调用

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean  对bean的属性赋值 ==这个不包括注解的依赖注入==

     - 所有 InstantiationAwareBeanPostProcessor接口实现类的postProcessAfterInstantiation方法调用 返回false终止循环调用
     - 判断org.springframework.beans.factory.support.AbstractBeanFactory#beanPostProcessors属性集合中是否存在InstantiationAwareBeanPostProcessor实现类存在则 循环调用InstantiationAwareBeanPostProcessor接口实现类的postProcessProperties方法对bean的属性配置进行修改 如果该方法返回null 那么将调用postProcessPropertyValues方法对bean的属性配置进行重新赋值 如果还是null 该populateBean 方法结束运行
     - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyPropertyValues 应用上面一系列操作后的bean的属性配置进行对bean进行配置

- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)初始化bean  

    - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory

      ​	#applyBeanPostProcessorsBeforeInitialization 

      ​	所有BeanPostProcessor接口实现类 postProcessBeforeInitialization方法调用     

    - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory

      ​	#invokeInitMethods		

      ​	 实现InitializingBean的方法或者自定义的初始化方法调用

    - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#

      ​	applyBeanPostProcessorsAfterInitialization	 

      ​        所有BeanPostProcessor接口实现类 postProcessAfterInitialization方法调用   

- org.springframework.beans.factory.support.AbstractBeanFactory#registerDisposableBeanIfNecessary 注册销毁bean如果有需要


# spring接口用法

## BeanPostProcessor

org.springframework.beans.factory.config.BeanPostProcessor

```java
public interface BeanPostProcessor {

   /**应用这个BeanPostProcessor到给定的新实例初始化方法(例如 InitializingBean的 			         afterPropertiesSet 方法或者 一个自定义的 init-method)之前调用
    */
   @Nullable
   default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

   /**应用这个BeanPostProcessor到给定的新实例初始化方法(例如 InitializingBean的 			         afterPropertiesSet 方法或者 一个自定义的 init-method)之后调用
    */
   @Nullable
   default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

}
```

## MergedBeanDefinitionPostProcessor

接口全名 org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor

定义如下:

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

	//这个方法在spring调用bean构造器之后 初始化之前 
	void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

}
```

用例:AutowiredAnnotationBeanPostProcessor类实现这个接口用来做依赖注入配置的构建

## InstantiationAwareBeanPostProcessor

org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
	
	@Nullable
	default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
    
	default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}
    //在执行org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
    //#populateBean方法的之前调用该方法对bean属性赋值的配置进行更改 ，返回更改后的属性配置
	@Nullable
	default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
			throws BeansException {

		return null;
	}
	//过时的方法可以当postProcessProperties方法返回null的时候使用该方法和postProcessProperties      方法用法一样
	@Deprecated 
	@Nullable
	default PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		return pvs;
	}

}
```

用例:

- org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#setCustomTargetSourceCreators

* org.springframework.aop.framework.autoproxy.target.LazyInitTargetSourceCreator





# Autowired原理

## 准备

### 了解方法:

- org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition 
  //这个方法在spring调用bean构造器之后 初始化之前 

- org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessProperties

  后置处理属性值 在工厂应用这个指定bean属性之前




### 依赖数据配置准备

检测哪些注解需要依赖注入是autowiredAnnotationTypes 这个属性决定 初始化代码如下:

```java
this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
```

可以根据以上代码看出添加了3个注解 Autowired.class,Value.class,javax.inject.Inject（如果存在）

我们来看看org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor这个类

这个类就是做依赖注入的

一下是 对postProcessMergedBeanDefinition 的实现

```java
	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        //获取依赖注入数据
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		//检测配置成员
        metadata.checkConfigMembers(beanDefinition);
	}

//返回依赖注入的数据配置
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
		// Fall back to class name as cache key, for backwards compatibility with custom callers.
		String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
		// Quick check on the concurrent map first, with minimal locking.
		InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    	//判断需不需要刷新
		if (InjectionMetadata.needsRefresh(metadata, clazz)) {
			synchronized (this.injectionMetadataCache) {
				metadata = this.injectionMetadataCache.get(cacheKey);
				if (InjectionMetadata.needsRefresh(metadata, clazz)) {
					if (metadata != null) {
						metadata.clear(pvs);
					}
                    //重新构建
					metadata = buildAutowiringMetadata(clazz);
                    //添加缓存
					this.injectionMetadataCache.put(cacheKey, metadata);
				}
			}
		}
		return metadata;
	}

//构建依赖注入数据配置
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
		List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
		Class<?> targetClass = clazz;
		//循环查找父类
		do {
			final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
			//遍历所有字段 判断是否有存在于autowiredAnnotationTypes里面类型的注解
			ReflectionUtils.doWithLocalFields(targetClass, field -> {
				AnnotationAttributes ann = findAutowiredAnnotation(field);
				if (ann != null) {
					if (Modifier.isStatic(field.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static fields: " + field);
						}
						return;
					}
					boolean required = determineRequiredStatus(ann);
					currElements.add(new AutowiredFieldElement(field, required));
				}
			});
			//遍历所有方法  判断是否有存在于autowiredAnnotationTypes里面类型的注解
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
					if (Modifier.isStatic(method.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static methods: " + method);
						}
						return;
					}
					if (method.getParameterCount() == 0) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation should only be used on methods with parameters: " +
									method);
						}
					}
					boolean required = determineRequiredStatus(ann);
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			});

			elements.addAll(0, currElements);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return new InjectionMetadata(clazz, elements);
	}



//判断所有的依赖注入成员是否存在于beandefinition的externallyManagedConfigMembers属性中，不存在的添加到依赖注入配置的checkedElements属性中(后续方法循环这个属性进行依赖注入)
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
		Set<InjectedElement> checkedElements = new LinkedHashSet<>(this.injectedElements.size());
		for (InjectedElement element : this.injectedElements) {
			Member member = element.getMember();
			if (!beanDefinition.isExternallyManagedConfigMember(member)) {
				beanDefinition.registerExternallyManagedConfigMember(member);
				checkedElements.add(element);
				if (logger.isTraceEnabled()) {
					logger.trace("Registered injected element on class [" + this.targetClass.getName() + "]: " + element);
				}
			}
		}
		this.checkedElements = checkedElements;
	}
```



**根据上面的代码 spring把注入需要依赖注入的数据配置构建好了**

### 依赖配置注入

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessProperties实现了

org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor的postProcessProperties方法

```java
	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        //从缓存中获取依赖属性配置
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
            //执行依赖注入
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}


public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> checkedElements = this.checkedElements;
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
            //循环遍历每个注入属性进行注入
			for (InjectedElement element : elementsToIterate) {
				if (logger.isTraceEnabled()) {
					logger.trace("Processing injected element of bean '" + beanName + "': " + element);
				}
				element.inject(target, beanName, pvs);
			}
		}
	}



```



#### 属性依赖注入实现

```java
		//这是一个属性的注入过程
		@Override
		protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
			Field field = (Field) this.member;
			Object value;
            //判断时候缓存 有使用缓存
			if (this.cached) {
				value = resolvedCachedArgument(beanName, this.cachedFieldValue);
			}
			else {
				DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
				desc.setContainingClass(bean.getClass());
				Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
				Assert.state(beanFactory != null, "No BeanFactory available");
				TypeConverter typeConverter = beanFactory.getTypeConverter();
				try {
                    //解析得到这个属性注入需要的值
					value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
				}
				catch (BeansException ex) {
					throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
				}
				synchronized (this) {
                    //没有缓存添加缓存
					if (!this.cached) {
						if (value != null || this.required) {
							this.cachedFieldValue = desc;
							registerDependentBeans(beanName, autowiredBeanNames);
							if (autowiredBeanNames.size() == 1) {
								String autowiredBeanName = autowiredBeanNames.iterator().next();
								if (beanFactory.containsBean(autowiredBeanName) &&
										beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
									this.cachedFieldValue = new ShortcutDependencyDescriptor(
											desc, autowiredBeanName, field.getType());
								}
							}
						}
						else {
							this.cachedFieldValue = null;
						}
						this.cached = true;
					}
				}
			}
            //利用反射对属性进行注入
			if (value != null) {
                //如果属性不能直接访问(例如 private) 设置成直接访问
				ReflectionUtils.makeAccessible(field);
				field.set(bean, value);
			}
		}
	}
```

#### 方法依赖注入实现

```java
		@Override
		protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
			if (checkPropertySkipping(pvs)) {
				return;
			}
			Method method = (Method) this.member;
			Object[] arguments;
            //如果有缓存直接缓存获取
			if (this.cached) {
				// Shortcut for avoiding synchronization...
				arguments = resolveCachedArguments(beanName);
			}
			else {
				Class<?>[] paramTypes = method.getParameterTypes();
				arguments = new Object[paramTypes.length];
				DependencyDescriptor[] descriptors = new DependencyDescriptor[paramTypes.length];
				Set<String> autowiredBeans = new LinkedHashSet<>(paramTypes.length);
				Assert.state(beanFactory != null, "No BeanFactory available");
				TypeConverter typeConverter = beanFactory.getTypeConverter();
				//循环解析每个需要的参数
                for (int i = 0; i < arguments.length; i++) {
					MethodParameter methodParam = new MethodParameter(method, i);
					DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
					currDesc.setContainingClass(bean.getClass());
					descriptors[i] = currDesc;
					try {
						Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
						if (arg == null && !this.required) {
							arguments = null;
							break;
						}
						arguments[i] = arg;
					}
					catch (BeansException ex) {
						throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(methodParam), ex);
					}
				}
                //添加缓存
				synchronized (this) {
					if (!this.cached) {
						if (arguments != null) {
							Object[] cachedMethodArguments = new Object[paramTypes.length];
							System.arraycopy(descriptors, 0, cachedMethodArguments, 0, arguments.length);
							registerDependentBeans(beanName, autowiredBeans);
							if (autowiredBeans.size() == paramTypes.length) {
								Iterator<String> it = autowiredBeans.iterator();
								for (int i = 0; i < paramTypes.length; i++) {
									String autowiredBeanName = it.next();
									if (beanFactory.containsBean(autowiredBeanName) &&
											beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
										cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
												descriptors[i], autowiredBeanName, paramTypes[i]);
									}
								}
							}
							this.cachedMethodArguments = cachedMethodArguments;
						}
						else {
							this.cachedMethodArguments = null;
						}
						this.cached = true;
					}
				}
			}
            
			if (arguments != null) {
				try {
                    //如果方法不能直接方法 设置可以访问
					ReflectionUtils.makeAccessible(method);
                    //反射执行方法设置属性
					method.invoke(bean, arguments);
				}
				catch (InvocationTargetException ex) {
					throw ex.getTargetException();
				}
			}
		}
```



### 总结

就是org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor对以下2个接口的方法进行了实现 

第一个进行依赖注入配置数据的构建

第二个 进行根据配置数据进行依赖注入 

- org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition 
  //这个方法在spring调用bean构造器之后 初始化之前 代码如下:

- org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor#postProcessProperties

  后置处理属性值 在工厂应用这个指定bean属性之前



