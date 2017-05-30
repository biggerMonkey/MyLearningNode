加载静态成员/代码块->加载非静态成员/代码块->调用构造方法

所有均是从父类到子类

[SpringMVC启动源码分析参考](http://lib.csdn.net/article/java/32247)


DispatcherServlet->FrameworkServlet->(extends)HttpServletBean (impl)ApplicationContextAware->HttpServlet->GenericServlet

DispatcherServlet->

```
static {
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH,
            DispatcherServlet.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}catch (IOException ex) {
			throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " 
			+ ex.getMessage());
		}
	}
```

DispatcherServlet被创建后，调用init方法

GenericServlet->init()方法为空

HttpServlet没有实现init()方法

HttpServletBean ->
```
		// Set bean properties from init parameters.
		try {
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(),
            this.requiredProperties);
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new
            ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader,
            getEnvironment()));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			logger.error("Failed to set bean properties'" +getServletName() + "'", ex);
			throw ex;
		}
		// Let subclasses do whatever initialization they like.
		initServletBean();
		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
```
FrameworkServlet->**initServletBean();**

