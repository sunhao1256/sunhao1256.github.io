---
title: "Spring问题整理"
date: 2022-01-11T14:43:18+08:00
tags: ['Spring']
---





```java
@Override
@Nullable
public Object invoke(MethodInvocation mi) throws Throwable {
   if (!(mi instanceof ProxyMethodInvocation)) {
      throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
   }
   ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
   ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
   JoinPointMatch jpm = getJoinPointMatch(pmi);
   return invokeAdviceMethod(pjp, jpm, null, null);
}
```

# 当在Sping中配置的Bean存在相互依赖，Spring是怎么处理的

针对原型Bean直接抛出异常，不支持。

单例Bean，Spring使用3个Map做缓存，来处理。

分别是：一级缓存Spring最终保存的单例对象Map，二级缓存建造Spring单例对象的匿名工厂对象返回的就是三级缓存需要的，三级缓存是允许提前被依赖的单例对象。

# 阐述一个Bean获取的流程

- 尝试获取单例Bean
- 检查一级缓存是否有，没有的话，检查当前获取的Bean是否正在创建，如果正在创建即出现了Bean互相依赖情况，检查三级缓存是否已经有提前可被依赖的对象，如果没有的话，检查二级缓存是否有其工厂，有的话，使用工厂，实例化这个Bean，放入三级缓存里。供其他Bean依赖使用
- 没获取到，可能是原型Bean，也可能是单例Bean没有实例化
- 检查如果是原型Bean，而且正在创建中，即出现了原型Bean被依赖的情况，直接抛出异常
- 准备BeanDefinition，如果档期工厂没有相应的BD，而且父工厂又存在BD，使用父工厂的getBean方法去获取Bean
- 标记Bean创建过了
- 从当前工厂读取BD,并且转为RootBeanDefinition，获取期间，还要检查父工厂是否也有该Bean的BD，有的话，以父工厂得BD为基础，子工厂得BD覆盖掉其属性
- 检查BD是不是抽象的，无法实例化的类，抛出异常
- 检查BD中得DependsOn属性，针对所有Depend，循环实例化，如果检查到有Depend得Bean又依赖于当前目标Bean，抛出异常，互相提前依赖了。并且建立相关关系，所以DependOn意义是，依赖于一个完全实例化完成后的Bean
- 如果是单例的话，开始创建单例Bean，创建匿名工厂对象
- 标记单例Bean正在被创建
- 使用工厂对象去调用getObject方法
- 实际上执行了createBean方法
- 根据之前的RootBD，解析出需要实例化的Class对象
- 检查MethodOverrides目标方法是否存在Class对象中
- **在实例化对象之前，给InstantiationAwareBeanPostProcessor机会去改变实例，调用其postProcessBeforeInstantiation，AOP就是在这里实现的，此外，如果返回了，还会调用BPP的postProcessAfterInitialization，但不会调用postProcessBeforeInitialization了**
- 如果没有被InstantiationAwareBeanPostProcessor改变了的话，开始进入真正的实例化方法
- 实例化一个BeanWrapperImpl去封装实例
- 解析Class对象，确定Class对象有Public修饰符
- 如果有FactoryMethod的话，直接调用FactoryMethod返回实例，封装在BeanWrapperImpl，这里面也会初始化initBeanWrapper，将属性编辑器注入到BeanWrapperImpl身上，用于后续的属性注入
- 开始解析构造函数或者是FactoryMethod，如果解析过了，直接去实例化
- 否则进入构造函数解析
- 解析之前，看BPP有没有提供了构造函数，即SmartInstantiationAwareBeanPostProcessor的determineCandidateConstructors方法执行，如果返回了构造函数，就用BPP的了。
- 没有的话，进入默认的解析，依赖先看缓存里有没有解析过的参数，因为构造方法注入的话，很消耗性能，没有缓存的话，先看用户获取bean时有没有传入args，即构造函数的参数。没有的话，而且只有一个候选的构造函数，就直接用使用无参的了，没有的话，先去解析参数，construct-arg，既可以时Index，也可以是name。根据用户传入的arg长度，去解析。
- 最后解析完成后，使用实例化策略去实例化即可，这里也可使用cglib去处理，然后封装在BeanWrapperImpl中
- 至此，BeanWapper里已经包含了我们的目标对象的实例了
- 然后创建二级缓存，将上一步BeanWapper里的实例，作为二级缓存返回的对象，加载缓存里
- 至此，二级缓存的工厂加入 了。当在一开始获bean，一级获取不到，获取二级有工厂的时候，就会把BeanWapper的实例暴露出去，供后续使用
- 然后开始初始化实例
- 将上面暴露出来的示例进行属性注入
- 给InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation在属性注入之前最后一次机会，去改变Bean，并且阻止Bean的属性注入
- 判断属性注入的是byName还是byType，针对所有的非简单的属性，还有排除所有的ignoredDependencyInterfaces中的接口。进行getBean操作，保存到PropertyValues中
- 使用InstantiationAwareBeanPostProcessor的postProcessProperties，可以进行修改属性。继续使用postProcessPropertyValues，继续可以更该属性
- 得到所有属性后，应用属性到Bean实例身上，在应用属性的时候，会找到前工厂里的所有的TypeConverter去将属性变为需要的属性，如果变不成会报错
- 至此，属性赋值完毕
- 复制完毕后，开始初始化Bean，先激活所有的aware方法，
- 调用BPP的postProcessBeforeInitialization初始化之前方法，记住，这里的初始化，是Bean已经实例化之后的事情了，是执行其他事情的初始化
- 执行afterPropertiesSet方法，在执行init-methods方法
- 调用BPP的postProcessAfterInitialization初始化以后方法
- 至此返回暴露的bean，即getBean结束
- 最后处理销毁的方法，即出发destory-method的方法

# Spring是如何处理掉循环依赖的

- 针对非单例Bean出现循环依赖直接抛出异常
- 单例Bean
- Spring存在3个缓存Map
  - Spring完全生成好的BeanMap，key是Bean的name，Value是实例对象
  - Spring生成Bean的工厂Map，key是Bean的name，value是实现了ObjectFactory接口的实例对象
  - Spring尚未初始化，即赋予属性或者其他初始化动作的Bean实例Map，key是Bean的name，value是工厂map的工厂的返回值，即ObjectFactory的getObject方法结果
- 假设存在对象A依赖于对象B，对象Bean依赖于对象A
  - Spring根据A的name，首先取BeanMap里找是否有A的实例，没有的话，检查A是否正在创建，如果正在创建，则说明出现了循环依赖。（需要获取A，发现A又在创建，表名有其他bean需要A），尝试从可提前依赖的BeanMap获取EarlyBeanReference，如果没有，则尝试从工厂Map里找A对应的工厂对象，如果有工厂对象，则调用工厂对象进行返回，并且将工厂返回的Bean实例作为EarlyBeanReference，放入未完全实例化结束BeanMap里，删除工厂Map对应的value。此时工厂Map为空。
  - 此时，A没有正在创建，继续
  - 标记A正在创建，根绝BeanDefinition生成Bean实例对象，（此时对象实例已经生成完毕，但是还没有初始化），并且把A的创建工厂，放入工厂Map，而这个创建工厂getObject返回值就是刚才生成的实例对象，并且给SmartInstantiationAwareBeanPostProcessor接口机会取改变这个EarlyBeanReference对象。
  - 得到实例化后的A对象，开始注入A的属性，发现A的属性b，需要B对象。
  - B对象开始获取（此时，A还没有结束，即一级缓存中没有A，二级缓存中有A的工厂Map）
  - B的获取如上述一致，
  - 直至B实例化结束，开始注入B的属性，发现B的属性a，需要A对象
  - 又到了A对象开始获取
  - 此时，进入第一个流程，发现一级缓存里没有A，而A又正在创建中，出现循环依赖，去二级缓存里找A的工厂Map，调用工厂Map方法去，得到了EarlyBeanReference，放入三级缓存里，返回回去
  - 即此时，B注入属性成功，并且返回了一个EarlyBeanReference，即当前正在创建的A对象实例。
  - B注入成功属性后，B实例化完全结束，结束后，清除B的二三级缓存，加入一级缓存并返回
  - 此时回到了A的注入B属性逻辑中，A得到了B实例。而这个实例里的A属性对象，和当前获取A的对象是一个
  - A继续完成初始化动作，最后A实例化完全结束，清楚A的二三级缓存，加入一级缓存并返回
- 只有2个缓存行吗？为什么一定要3个
  - BeanMap无用质疑是需要的
  - 如果只有工厂Map而没有，可提前依赖的BeanMap的话，那么在一开始从缓存中获取Bean，一级缓存无法获取到，直接就有工厂Bean，一旦有工厂就调用工厂返回的值，这样是不行的，因为在工厂调用Bean的时候，有很多动作就会进行重复，比如工厂获取的时候，可以给SmartInstantiationAwareBeanPostProcessor机会去更改EarlyBeanReference对象，重复执行了。第二，与工厂模式的思想违背，工厂只需要制造一次，而不是每次都制造。
  - 如果只有可提前依赖的BeanMap，而没有工厂Map。实际上是可以的，只不过没有工厂的话，会将大部分工作都抛给创建Bean的流程里，例如SmartInstantiationAwareBeanPostProcessor等工厂应该负责的工作

# ApplicationContext的Refresh方法

- Enviroment，环境参数，根据不同的环境，实现Environment不同的子类，例如Web环境会实现，StandardServletEnvironment，默认是实现StandardEnvironment，包含很多环境变量，系统变量，java环境变量，Servlet环境变量
- 创建beanFactory作为成员变量，ApplicationContext自身也实现了BeanFactory接口，只不过具体实现的方法是成员变量的beanFactory的方法、
- 填充工厂
  - 增加SPEL表达式解析器
  - 属性编辑器注入
  - 增加一个ApplicationContextAwareProcessor的BPP，在Bean实例化之后，激活实现了aware接口的方法的一个BPP
  - 配置忽略某些类型的属性自动注入，增加某些类型自动注入
- postProcessBeanFactory：留给子类去实现
- 记录启动路径
- 激活BeanFactoryPostProcessor，invokeBeanFactoryPostProcessors
- 注册BeanPostProcessor
- 初始化国际化文件
- 初始化initApplicationEventMulticaster，事件传送器，用于发送事件
- 注册事件监听器
- 设置ConversionService
- 锁定所有BeanDefinitions，防止改变
- 实例化剩下所有的no-lazy实例
- 调用所有实现LifeCycle接口的bean
- 发送ContextRefreshedEvent事件

# BeanFactoryPostProcessor和BeanPostProcessor区别

- BeanFactoryPostProcessor是可以修改Bean的元数据，是控制BeanFactory的，而BeanPostProcessor是Bean实例的处理器，可以修改Bean的实例

- 以PropertySourcesPlaceholderConfigurer为例

  很明显，作用就是对于Beanfactory的所有Beandefintion进行处理，即在xml里中配置的${}属性

  ```java
  	@Override
  	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
  		try {
  			Properties mergedProps = mergeProperties();
  
  			// Convert the merged properties, if necessary.
  			convertProperties(mergedProps);
  
  			// Let the subclass process the properties.
  			processProperties(beanFactory, mergedProps);
  		}
  		catch (IOException ex) {
  			throw new BeanInitializationException("Could not load properties", ex);
  		}
  	}
  
  
  	@Override
  	protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
  			throws BeansException {
  
  		StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
  		doProcessProperties(beanFactoryToProcess, valueResolver);
  	}
  
  	protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
  			StringValueResolver valueResolver) {
  
  		BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
  
  		String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
  		for (String curName : beanNames) {
  			// Check that we're not parsing our own bean definition,
  			// to avoid failing on unresolvable placeholders in properties file locations.
  			if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
  				BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
  				try {
  					visitor.visitBeanDefinition(bd);
  				}
  				catch (Exception ex) {
  					throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
  				}
  			}
  		}
  
  		// New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
  		beanFactoryToProcess.resolveAliases(valueResolver);
  
  		// New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
  		beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
  	}
  
  
  	public void visitBeanDefinition(BeanDefinition beanDefinition) {
  		visitParentName(beanDefinition);
  		visitBeanClassName(beanDefinition);
  		visitFactoryBeanName(beanDefinition);
  		visitFactoryMethodName(beanDefinition);
  		visitScope(beanDefinition);
  		if (beanDefinition.hasPropertyValues()) {
  			visitPropertyValues(beanDefinition.getPropertyValues());
  		}
  		if (beanDefinition.hasConstructorArgumentValues()) {
  			ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
  			visitIndexedArgumentValues(cas.getIndexedArgumentValues());
  			visitGenericArgumentValues(cas.getGenericArgumentValues());
  		}
  	}
  
  ```

- 上面的例子即可以看出BeanFactoryPostProcessor是对BeanFactory进行操作，是Bean还没实例化之前的动作

# 原理Spring AOP

- Xml方式中，aspectj-autoproxy，标签默认创建一个AnnotationAwareAspectJAutoProxyCreator的BeanDefinition，注册到BeanDefinitionRegistry中
- 通过事件发送，通知到注册了
- 在Application的refresh中的finishBeanFactoryInitialization，会实例化该bean

- AnnotationAwareAspectJAutoProxyCreator实现了Spring的BPP接口

- 正常情况下，AOP会在postProcessAfterInitialization方法执行后，wrapIfNecessary，会去找当前容器里所有的增强器，一个增强器里包含了，切入点，切入方法

- 首先会找到容器里所有的增强器，做缓存处理，对于每一个要获取的Bean都会走这个方法。具体的寻找方法，包括Spring老式的配置，即在XML里配置Bean，使用aop:config里的aop:ref和aop:point_cut，配置出来的，以及现在十分方便的使用aspectJ注解的@Aspect配合@PointCut，@Around，@Before，@After，@Throw

- 然后根据Bean的属性。检查找到的增强其里面是否有满足条件的。即查看是否满足AOP，表达式的，方法

- 如果这个bean有满足的增强器

- 则取获取代理对象，Spring有2中实现代理的方式，JDK自带的Proxy类，以及CGlib

- JDK和GClib区别

  - JDK只能针对实现了接口的类，生成代理，实现同样的接口方法，但里面调用的还是目标对象的方法
  - CGLIB是针对类实现代理，通过修改字节码方式，对目标对象生成一个其子类，覆写父类的方法，从而达到代理，所以类不能声明为final

- Spring可以强制使用cglib，配置proxyTargetClass属性为true即可

- 根据上述找到的增强器，Spring根据配置，创建对应的AopProxy，JdkDynamicAopProxy或者CglibAopProxy

- 调用AopProxy的getProxy方法

- JdkDynamicAopProxy

  - 使用JDK自带的Proxy.newProxyInstance方式创建代理对象

    ```java
    Proxy.newProxyInstance(classLoader, this.proxiedInterfaces, this);
    ```

  - 最后一个参数this，说明JdkDynamicAopProxy肯定实现了InvocationHandler，最后执行的目标方法就是invoke方法

  - 在这里面处理了expose-proxy属性

    ```java
    //AopContext 自身无法获取代理对象的处理 exposeProxy
    if (this.advised.exposeProxy) {
       // Make invocation available if necessary.
       oldProxy = AopContext.setCurrentProxy(proxy);
       setProxyContext = true;
    }
    ```

  - 然后根据传入的增强器，使用ReflectiveMethodInvocation封装，方法链，逐个调用。不同的advistor使用不同的实现类去实现，例如

    - ```
      AspectJAroundAdvice
      ```

      ```java
      @Override
      @Nullable
      public Object invoke(MethodInvocation mi) throws Throwable {
         if (!(mi instanceof ProxyMethodInvocation)) {
            throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
         }
         ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
         ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
         JoinPointMatch jpm = getJoinPointMatch(pmi);
         return invokeAdviceMethod(pjp, jpm, null, null);
      }
      ```

      这也就是为什么@Around的方法，入参可以是ProceedingJoinPoint

      ```java
      invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));
      ```

  - 直至所有的方法链调用结束
  - 最后执行目标方法
  - 处理返回对象

# SpringMVC流程

SpringMvc会使用DispatcherServlet作为请求的入口。

作为一个Servlet，init方法，执行后，DispatcherServlet会创建自己的IOC容器，并且把ServletContext中的IOC作为父容器，所以MVC的相关内容，例如Controller可以放在MVC的IOC中，而其余的，dao，service可以放在SevletContext里的IOC，好处就是隔离开了M层，如果要还Struts，可以直接更换，而不用考虑dao和service了。

init方法，除了创建MVC自己的容器，还根据DispatchServlet.properties里的内容，进行一系列的初始化动作。

DispatcherServlet继承于FrameworkServlet。集成HttpServlet，最后从Servlet会执行	到DispatchServlet的processRequest方法。processRequest中，Spring会将当前的请求requestAttributes放在RequestContextHolder中用ThreadLocal保存，所以后续的Controller都可以使用。

随后走到doDispatch中，做一些必要的检查，然后根据容器里的handerMapping找到对应的hanlder，使用HandlerExecutionChain将符合该请求的拦截器，封装在一起，跨域也是在这里增加另一个拦截器

准备好HandlerExecutionChain之后，进入流程。先调用所有的拦截器preHandler方法

Handler有很多种，最后使用适配器模式，创建handlerAdapter去调用不同handler的具体执行方法。从而得到modelAndView

然后调用拦截器的postHandler方法

得到modelAndView，之后会使用ViewRovler解析view，最后获取requestDispatcher，forward或者是重定向到新的路径，并且把model，放入request的attribute里，从而进入下一个请求了，可以是reward也可以是redirect

## 为什么在Controller的方法参数里，写HttpServletRequest就可以获得到请求对象，以及@RequestBody和@RequestParam

不管是xml中配置<mvc:annotation-driven/>，还是使用注解@EnableWebMvc，本质上都是在容器中注入了RequestMappingHandlerMapping这样的实例。实例实现了initalizingBean接口，在容器中实例化的时候，会将容器中所有的Bean找出来，检查Class是否被@Controller或者@RequestMapping修饰，找到即作为一个handler。

然后去找到Hanlder对应的adapter，RequestMappingHandlerAdapter，Adapter的handler方法。

hanlder方法的核心是，invokeHandlerMethod，通过反射调用目标方法，并且通过一些列配置，将方法需要的配置传入进去，其中解析方法参数的就是一些列的argumentResolvers。

Spring依然是在RequestMappingHandlerAdapter的afterPropertiesSet()方法之后，。初始化了argumentResolvers

如下

```java
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
   List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

   // Annotation-based argument resolution
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
   resolvers.add(new RequestParamMapMethodArgumentResolver());
   resolvers.add(new PathVariableMethodArgumentResolver());
   resolvers.add(new PathVariableMapMethodArgumentResolver());
   resolvers.add(new MatrixVariableMethodArgumentResolver());
   resolvers.add(new MatrixVariableMapMethodArgumentResolver());
   resolvers.add(new ServletModelAttributeMethodProcessor(false));
   resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new RequestHeaderMapMethodArgumentResolver());
   resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new SessionAttributeMethodArgumentResolver());
   resolvers.add(new RequestAttributeMethodArgumentResolver());

   // Type-based argument resolution
   resolvers.add(new ServletRequestMethodArgumentResolver());
   resolvers.add(new ServletResponseMethodArgumentResolver());
   resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   resolvers.add(new RedirectAttributesMethodArgumentResolver());
   resolvers.add(new ModelMethodProcessor());
   resolvers.add(new MapMethodProcessor());
   resolvers.add(new ErrorsMethodArgumentResolver());
   resolvers.add(new SessionStatusMethodArgumentResolver());
   resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
   if (KotlinDetector.isKotlinPresent()) {
      resolvers.add(new ContinuationHandlerMethodArgumentResolver());
   }

   // Custom arguments
   if (getCustomArgumentResolvers() != null) {
      resolvers.addAll(getCustomArgumentResolvers());
   }

   // Catch-all
   resolvers.add(new PrincipalMethodArgumentResolver());
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
   resolvers.add(new ServletModelAttributeMethodProcessor(true));

   return resolvers;
}
```

通过 HandlerMethodArgumentResolver的supportsParameter方法

以RequestParam为例

首先RequestParamMethodArgumentResolver的父类AbstractNamedValueMethodArgumentResolver，进行校验，校验Hanlder的参数是否是必填，会将@RequestParams的required进行校验，从HttpServletRequest里的parameter进行校验。校验成功，会通过反射执行handler，并将参数传入Handler中。

@RequestBody也是一样，使用的RequestResponseBodyMethodProcessor去处理参数的

在处理@RequestBody的时候，RequestMappingHandlerAdapter的所有messageConverters，找到符合的，我们常用的application/json就是MappingJackson2HttpMessageConverter，需要引入jackson的核心包

至于其他的Httpservlet都是是同对应的argumentResolve处理的

# SpringBoot的工作原理

Spring在诞生之初，配置类一直是令开发者吐槽的模块，当java5推出注解后，可以使用java类进行配置了，但依然很繁琐。而SpringBoot的最大的一个特点，就是解决复杂的配置。

SpringBoot推出了一系列注解，他们可以将各类Spring boot starter，无感的注入到容器中。

SpringBoot在SpringBoot的run方法里，完成了一些列工作。

首先是构造方法会根据当前类路径下是否包含webflux，或者servlet相关类，从而决定实例化的容器对象。

以常见的servlet为例，SpringBoot会实例化AnnotationConfigServletWebServerApplicationContext

而在SpringBoot构造函数触发时，会寻找类路径下META/Spring.factories文件，下所有的ApplicationContextInitializer实例，和ApplicationListener实例，并且放入当前对象的属性中

- 核心的run方法，里首先还是从spring.factories里找到所有的SpringApplicationRunListener，调用其starting方法，
- 然后封装main方法的arg参数，校验环境，即我们可以在外部指明spring的profile.active=prod就是在这里指明的
- 准备好main的arg以及sytem的相关属性后，会调用所有SpringApplicationRunListener的environmentPrepared，也就是在这里面进行了Spring配置文件，application.yml或者是application.properties的处理
- 打印Banner

- 根据上述得到的实例化Class，实例化IOC容器

- 从spring.factories里找到SpringBootExceptionReporter用于异常打印

- 准备名称生成器，资源加载器，类加载器等。

- 调用Spring.factories里的ApplicationContextInitializer的initialize方法

- 调用ApplicationContextInitializer的contextPrepared方法

- 打印启动类信息，打印profile信息

- 加载启动类的注解信息，即启动类也可以是一个组件，**注入到容器中**，默认情况下，@SpringBootApplication里的@SpringBootConfiguration里有@Component注解，Spring会通过注解方式将其注入到容器中，细节就不说了，核心是@SpringBootConfiguration身上有@Configuration，而且有@Import注解，引入的是AutoConfigurationImportSelector

   SpringBoot在启动的时候，注入了ConfigurationClassPostProcessor该类实现了BeanDefinitionRegistryPostProcessor会在容器的refresh里的invokeBeanFactoryPostProcesser，里触发

  postProcessBeanDefinitionRegistry方法，会找到容器里所有的@Configuration实例，这里此时容器里有的就是启动类本身，然后会处理，@Configuration上的@Import，并且执行@Import类的process方法，即，最后依然调用的是selectImports方法，得到的就是Spring.factories里配置的**org.springframework.boot.autoconfigure.EnableAutoConfiguration=**的值，然后逐步解析所有的类，这样容器里就有第三方调用的所有信息了

  ```java
  public void process(AnnotationMetadata annotationMetadata,
        DeferredImportSelector deferredImportSelector) {
     Assert.state(
           deferredImportSelector instanceof AutoConfigurationImportSelector,
           () -> String.format("Only %s implementations are supported, got %s",
                 AutoConfigurationImportSelector.class.getSimpleName(),
                 deferredImportSelector.getClass().getName()));
     AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
           .getAutoConfigurationEntry(getAutoConfigurationMetadata(),
                 annotationMetadata);
     this.autoConfigurationEntries.add(autoConfigurationEntry);
     for (String importClassName : autoConfigurationEntry.getConfigurations()) {
        this.entries.putIfAbsent(importClassName, annotationMetadata);
     }
  }
  ```

  ```java
  protected AutoConfigurationEntry getAutoConfigurationEntry(
        AutoConfigurationMetadata autoConfigurationMetadata,
        AnnotationMetadata annotationMetadata) {
     if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
     }
     AnnotationAttributes attributes = getAttributes(annotationMetadata);
     List<String> configurations = getCandidateConfigurations(annotationMetadata,//获取到所有的/META-INF/spring-factories中的configuration
           attributes);
     configurations = removeDuplicates(configurations);//删除重复的
     Set<String> exclusions = getExclusions(annotationMetadata, attributes);
     checkExcludedClasses(configurations, exclusions);//根据上面的属性excclusion
     configurations.removeAll(exclusions);
     configurations = filter(configurations, autoConfigurationMetadata);
     fireAutoConfigurationImportEvents(configurations, exclusions);//通知导入成功
     return new AutoConfigurationEntry(configurations, exclusions);
  }
  ```

# Spring事务

- Spring使用注解@Transactional放在方法，或者类上
- @Transactional属性有
  - transactionManager指明事务管理器，在多数据源的时候，需要指明
  - propagation传播行为默认是Propagation.REQUIRED，即当前如果存在事务，则用同一个事务，否则开启一个新事务
  - isolation隔离级别，默认是数据库的隔离级别，mysql默认时可重复读，幻读是通过MVCC和间隙锁解决的
  - timeout超时时间
  - readOnly是否只读
  - rollbackFor发生什么异常才会会滚，默认是RuntimeException
  - rollbackForClassName
  - noRollbackFor
  - noRollbackForClassName
- 原理，@Transactional注解是基于AOP原理，容器在获取相应bean的时候，会去使用wrapifNecessary包装bean，Spring的流程是，从容器里找到所有的adviser，而@Transactional会在容器里注入BeanFactoryTransactionAttributeSourceAdvisor，从而最后得到的是代理对象，而代理对象针对于满足切面条件的bean，会做处理，@Transactional就是切面条件，会在目标方法执行之前开启一个事务，执行之后提交或者回滚一个事务，操作事务的都是TransactionManager

- 提问

  - 如果在一个方法里异常被try-catch了，还会会滚吗？不会了，因为Sping处理提交事务是在切面的AfterRunning之后提交的，在那之前会使用try-catch捕捉，而目标自己捕捉过的话，Spring就捕捉不到了，除非在catch里再抛出

  - 如果自己使用TransactionManager重新在获取一个事务，手动提交之前，抛出异常的话，会会滚吗？不会，会被Spring捕捉到

  - 如果在抛出异常之前，就手动提交了，Spring会回滚吗，可能会 也可能不会，这和Spring事务的传播行为有关，默认情况下传播行为是，如果当前有事务了，就用当前事务，否则开启一个新事务。

    如果有@Transactional注解了，意味着Spring已经开启了一个事务了，后续自己手动开启的事务，就是之前的事务了，并非一个新事务，Spring在提交事务之前，会判断是否为一个新事务，是的话，才会进行提交，否则不会有动作。所有如果传播行为不变的话，会回滚的，如果是强制开启新的事务的传播行为，则不会会滚

# 讲讲你会的设计模式，Spring里面用到了哪些设计模式

# Java中的Date是线程安全的吗？你是如何保证线程安全的

# 讲讲ThreadLocal 和java的内存模型吧



# WebAsyncTask

WebAsyncTask，是基于Servlet3.0异步Servlet的特性，用于提高容器的吞吐量，客户端请求进来时，当前的容器分配的线程会直接返回，并会开启一个新线程用于处理客户端请求，当新线程结束时，客户端才会收到请求。而容器的线程就可以回收用于新的请求，提高了吞吐量



## 拦截器和过滤器的区别

首先，拦截器与过滤器不是一起的，拦截器是Spring的，而过滤器filter是javax提供的，filter会在拦截器之前执行，拦截器可以达到更细粒控制请求。拦截器可以获得到具体执行的handler，可以对handler做个性化操作。过滤器可以做的，拦截器都可以做到，拦截器做的，过滤器做不了。



# 聊一下mybatis

## 什么是mybatis的懒加载

## mybatis的mapper是如何与数据库连接的

# 什么是索引，索引为什么可以加查询

# Innodb索引实现

# B树和B+树

# Redis使用，部署，哨兵

# TCP三次握手

# 乐观锁，悲观锁，举例

# 设计模式

# Redis扩容

# CAS概念，原子类的实现

# Synchronize 底层原理

# AQS

# 网络模型，你知道的网络协议

# HTTPS的连接过程，SSL

# 单链表算法，反转，查找

# jvm垃圾回收机制

# 工作流引擎

# Spring MVC webFlux

# netty

# redis为什么这么快

# Mysql的explain

# K8S

# 讲一讲反射，以及你工作中用到的地方

java反射，提供给我们方式去动态创建类，或者是动态执行方法，当我们因为某些条件，不知道目标应该执行什么方法的时候，就可以使用反射。

例如，请假，如果超过2天，就执行某方法，否则执行另一个方法，我们就可以用反射

反射也被大量应用于mybatis和spring之中，spring中aop，创建bean的动作都是反射，因为他们无论是执行方法还是构建实例都是未知的参数和条件