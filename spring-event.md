# spring event 原理

## 容器刷新过程

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
       //初始化earlyApplicationEvents来保存提前发布的事件
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
          //初始化可以分发事件的Multicaster
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
          //将容器中的事件监听器注册到multicaster中，earlyApplicationEvents中的事件注册到 			   //multicaster并将earlyApplicationEvents复制为空
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
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
         resetCommonCaches();
      }
   }
}
```

## ApplicationEventMulticaster

时间分发就是靠这个轮播器实现的

初始化在容器刷新方法refresh()

  

```java
// Initialize event multicaster for this context.
initApplicationEventMulticaster();


protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//先判断容器中时候存在名称applicationEventMulticaster的bean 存在则调用容器的getBean方法		//初始化
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
    //不存在初始化一个SimpleApplicationEventMulticaster的注册到容器中
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
						APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
						"': using default [" + this.applicationEventMulticaster + "]");
			}
		}
	}
```

## earlyApplicationEvents属性介绍

ApplicationEventPublisher实现类里面有这个earlyApplicationEvents属性 表示的是ApplicationEventMulticaster这个对象还没有初始化的时候发布的事件都会添加到这个属性里面，当ApplicationEventMulticaster初始化结束再注册事件监听的时候会将该属性里面的所有事件添加到ApplicationEventMulticaster对象当中具体如下:

### earlyApplicationEvents的初始化

容器刷新方法中有一个准备刷新的发放prepareRefresh

```java
protected void prepareRefresh() {
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // Initialize any placeholder property sources in the context environment
   initPropertySources();

   // Validate that all properties marked as required are resolvable
   // see ConfigurablePropertyResolver#setRequiredProperties
   getEnvironment().validateRequiredProperties();

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   //初始化this.earlyApplicationEvents 用来缓存提前的事件(multicaster还没初始化前发布的事件)
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### earlyApplicationEvents销毁

容器刷新方法中有一个注册监听的方法registerListeners

```java
protected void registerListeners() {
   // Register statically specified listeners first.
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
    //获取容器中的所有事件监听注册到Multicaster中
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
    //这个时候multicaster 已经初始化完成不需要这个属性保存 所以清空并将里面的事件添加到multicaster 	//中
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

## ApplicationEventPublisher

时间发布器  容器实现这个接口

看看一个时间是如何发布的

```java
publishEvent(event, null);

protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
       //判断发布的事件是不是一个ApplicationEvent实现类是直接转化赋值给applicationEvent
       //如果不是则创建一个PayloadApplicationEvent对象包装一下复制给applicationEvent
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
    	//不为空的时间表示ApplicationEventMulticaster 还没有初始化 把事件缓存起来
    	//prepareRefresh()这个方法里面为earlyApplicationEvents复制
    	//registerListeners()将earlyApplicationEvents里面的所有时间添加到
         //ApplicationEventMulticaster里面去 具体细节请看earlyApplicationEvents属性介绍
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
            //获取轮播器发布事件
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}





@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        //获取到所有可以匹配改事件的监听 循环调用
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            //获取多线程执行器
			Executor executor = getTaskExecutor();
            //不为空多线程执行
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {//该线程直接调用	
				invokeListener(listener, event);
			}
		}
	}


protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    	//获取异常处理器
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
                //调用事件监听器
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
                //执行异常处理
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}

@SuppressWarnings({"unchecked", "rawtypes"})
	private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
            //调用事件监听器将事件传递过去
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				// -> let's suppress the exception and just log a debug message.
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
```



# 总结

以上大致就是事件发布的大致流程

# 注解EventListener

注释里面有说明 实现查看 EventListenerMethodProcessor类