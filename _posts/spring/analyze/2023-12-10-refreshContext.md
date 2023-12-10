---
title: SpringApplication refreshContext
date: 2023-12-10 20:00:00 +0800
categories: [Spring, Analyze]
tags: [spring, code, analyze]
render_with_liquid: false
---

```java
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {
    //이전 코드 생략

    //Context Refresh 단계
    refreshContext(context);

    //이후 코드 생략

    return context;
  }
}
```

SpringApplication 클래스의 run메소드는 refreshContext라는 단계를 가지고 있다.
들어가기 앞서서 refresh에 대한 내용을 조금 짚어보면.

```java
public class MySpringApplication {
  public static void run(Class<?> applicationClass, String[] args) {
    AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext() {
      @Override
      protected void onRefresh() {
        super.onRefresh();

        ServletWebServerFactory serverFactory = this.getBean(ServletWebServerFactory.class);
        DispatcherServlet dispatcherServlet = this.getBean(DispatcherServlet.class);
        dispatcherServlet.setApplicationContext(this);

        WebServer webServer = serverFactory.getWebServer(servletContext ->
          servletContext.addServlet("dispatcherServlet", dispatcherServlet
          ).addMapping("/*"));
        webServer.start();
      }
    };
    applicationContext.register(applicationClass);
    applicationContext.refresh();
  }
}
```

이 코드에서 볼 수 있듯이 스프링에 빈을 등록하기 위해서는 `register`하는 과정과 `refresh`하는 과정이 필요하다.

* `register` : `ㅁpplicationContext`에게 이 클래스를 빈으로 등록할 것이라고 알려주는 것
* `refresh` : `register`에서 알려준 클래스들을 실제 빈으로 등록한다

이제 refresh에 대한 내용을 알았의니 refreshContext에 대해서 알아보자.

<br>

# refreshContext 동작 과정

```java
public class SpringApplication {
  private boolean registerShutdownHook = true;

  private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
      shutdownHook.registerApplicationContext(context);
    }
    refresh(context);
  }

  protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
  }
}
```

refresh 메소드를 보면 제일 먼저 `registerShutdownHook`의 값을 확인하는 것을 볼 수 있다.
`ShutdownHook`은 **프로그램의 종료를 감지하여 프로그램 종료 시에 후처리 작업을 진행하는 기술**이다.
**프로그램이 실행되다가 프로세스가 죽어버리면 연결 종료, 자원 반납 등의 처리를 못하게 되는데**, ShutdownHook를 사용함으로써 **프로그램이 종료되어도 별도의 스레드가 올바른 프로그램 종료(
Graceful Shutdown)를 위한 작업을 처리할 수 있다**.

<br>

스프링의 경우 대표적으로 `DisposableBean 인터페이스`나 `@PreDestroy`또는 `@Bean 소멸자 메소드`를 지정해 소멸자를 구현할 수 있는데,
`SpringApplication`이 여기서 `ShutdownHook`를 등록함으로써 애플리케이션이 갑자기 죽어도 별도의 스레드를 통해 정상적으로 소멸자 메소드를 처리할 수 있는 것이다.

* 하지만 `registerShutdownHook = false`로 설정한다면 소멸자 메소드가 처리되지 않는다.

<br>

위 과정을 거친 후에 진짜로 ApplicationContext를 refresh해준다.
예시를 들기 위해 한가지 ApplicationContext를 가져와서 보자.

```java
public class ReactiveWebServerApplicationContext extends GenericReactiveWebApplicationContext
  implements ConfigurableWebServerApplicationContext {

  @Override
  public final void refresh() throws BeansException, IllegalStateException {
    try {
      super.refresh();
    } catch (RuntimeException ex) {
      WebServerManager serverManager = this.serverManager;
      if (serverManager != null) {
        serverManager.getWebServer().stop();
      }
      throw ex;
    }
  }
}
```

부모 클래스인 `AbstractApplicationContext`의 refresh 메소드 내부에는 템플릿 메소드 패턴이 적용되어 있다.
위의 코드를 보면 왜 refresh 내부에서 템플릿 메소드 패턴을 적용했는지 알 수 있다.
애플리케이션 컨텍스트는 애플리케이션 타입에 따라 서로 다른 웹 서버를 생성해야 한다.
대표적으로 ServletWebServerApplication이나 ReactiveWebServerApplicationContext의 전반적인 refresh로직은 동일하다.
하지만 이를 모든 자식 클래스에 구현하면 중복이 발생하기 때문에 스프링은 전반적인 refresh 로직을 부모 클래스인 AbstractApplicationContext에 만들어두고, 애플리케이션 타입 별로 달라져야
하는 부분은 자식이 onRefresh 메소드를 오버라이딩 하고 로직을 구현하도록 하고 있다.

```java
public class ReactiveWebServerApplicationContext {
  @Override
  public final void refresh() throws BeansException, IllegalStateException {
    try {
      super.refresh();
    } catch (RuntimeException ex) {
      WebServerManager serverManager = this.serverManager;
      if (serverManager != null) {
        serverManager.getWebServer().stop();
      }
      throw ex;
    }
  }

  @Override
  protected void onRefresh() {
    super.onRefresh();
    try {
      createWebServer();
    } catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start reactive web server", ex);
    }
  }
}

public class ServletWebServerApplicationContext {
  @Override
  public final void refresh() throws BeansException, IllegalStateException {
    try {
      super.refresh();
    } catch (RuntimeException ex) {
      WebServer webServer = this.webServer;
      if (webServer != null) {
        webServer.stop();
        webServer.destroy();
      }
      throw ex;
    }
  }

  @Override
  protected void onRefresh() {
    super.onRefresh();
    try {
      createWebServer();
    } catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start web server", ex);
    }
  }
}
```

<br>

# refresh 동작 과정

refresh는 `start-up 메소드`이며 싱글톤 빈으로 등록할 클래스들을 찾아서 생성하고 후처리 하는 단계이다.
여기서 후처리는 `@Value`, `@PostConstruct`, `@Autowired`등을 처리하는 것이다.
실제 refresh 로직은 부모 클래스인 AbstractApplicationContext에 구현되어있다.

```java
//스프링 3.2.0
public abstract class AbstractApplicationContext {
  @Override
  public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
      StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

      //1. refresh 준비 단계
      prepareRefresh();

      //2. BeanFactory 준비 단계
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();   //내부 빈 팩토리를 새로 고치도록 서브클래스에 지시한다
      prepareBeanFactory(beanFactory);

      try {
        //3. 후처리 진행
        postProcessBeanFactory(beanFactory);    //컨텍스트 서브클래스에서 Bean 팩토리의 사후 처리를 허용한다

        StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");

        //4. BeanFactoryPostProcessor 실행
        invokeBeanFactoryPostProcessors(beanFactory);   //컨텍스트에서 Bean으로 등록된 팩토리 프로세서를 호출한다

        //5. BeanPostProcessor 등록
        registerBeanPostProcessors(beanFactory);    //빈 생성을 가로채는 BeanPostProcessor를 등록한다
        beanPostProcess.end();

        //6. MessageSource 및 Event Multicaster 초기화
        initMessageSource();                //이 컨텍스트에 대한 MessageSource를 초기화한다
        initApplicationEventMulticaster();  //이 컨텍스트에 대한 Event Multicaster를 초기화한다

        //7. onFresh() 웹 서버 생성
        onRefresh();    //특정 컨텍스트 하위 클래스에서 다른 특수 Bean을 초기화한다

        //8. ApplicationListener 조회 및 등록
        registerListeners();    //Listener Bean을 확인하고 등록한다

        ///9. 빈들의 인스턴스화 및 후처리
        finishBeanFactoryInitialization(beanFactory);   //나머지(lazy-init가 아닌) 싱글톤을 모두 인스턴스화한다

        //10. refresh 마무리 단계
        finishRefresh();    //해당 이벤트를 게시합니다.
      } catch (RuntimeException | Error ex) {
        if (logger.isWarnEnabled()) {
          logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
        }
        //dangling resources를 방지하기 위해 생성된 싱글톤을 Destroy한다
        // Destroy already created singletons to avoid dangling resources.
        destroyBeans();

        // active 플래그를 리셋한다
        cancelRefresh(ex);

        //호출자에게 예외를 전파한다
        throw ex;
      } finally {
        contextRefresh.end();
      }
    }
  }
}
```

refresh 메소드 호출을 통해 모든 객체들이 싱글톤으로 인스턴스화 되는데, 만약 에러가 발생히면 등록된 빈들을 모두 제거한다.
즉 refresh 단계를 거치면 모든 빈이 인스턴스화 되었거나 어떠한 빈도 존재하지 않거나 둘 중 하나인 상태가 된다.

<br>

# 1. refresh 준비 단계

> ApplicationContext의 상태를 active로 변경하는 등의 준비 작업

```java
public abstract class AbstractApplicationContext {
  protected void prepareRefresh() {
    //active 상태로 바꾸는 작업
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    if (logger.isDebugEnabled()) {
      if (logger.isTraceEnabled()) {
        logger.trace("Refreshing " + this);
      } else {
        logger.debug("Refreshing " + getDisplayName());
      }
    }

    //컨텍스트 환경에서 placeholder 속성 소스를 초기화한다
    // Initialize any placeholder property sources in the context environment.
    initPropertySources();


    //필수로 표시된 모든 속성이 확인 가능한지 확인한다
    //ConfigurablePropertyResolver#setRequiredProperties를 참조하라
    // Validate that all properties marked as required are resolvable:
    getEnvironment().validateRequiredProperties();

    //refresh 전 ApplicationListener 저장
    if (this.earlyApplicationListeners == null) {
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    } else {
      //local application listeners를 새로 refresh 전 상태로 재설정합니다.
      this.applicationListeners.clear();
      this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    //초기 ApplicationEvents 수집을 허용한다
    //multicaster가 사용 가능하게되면 publish된다
    this.earlyApplicationEvents = new LinkedHashSet<>();
  }
}
```

여기서 제일 처음에 애플리케이션 컨텍스트의 상태 중 active를 true로 바꾸는 것은 중요하다.
그 이유는 빈 팩토리에서 빈을 꺼내는 작업은 active 상태가 true일 대만 가능하기 때문이다.
대표적으로 getBean메소드를 보면 active 상태가 아니면 throw하도록 되어있다.

```java
public abstract class AbstractApplicationContext {
  @Override
  public <T> T getBean(Class<T> requiredType) throws BeansException {
    assertBeanFactoryActive();
    return getBeanFactory().getBean(requiredType);
  }

  protected void assertBeanFactoryActive() {
    if (!this.active.get()) {
      if (this.closed.get()) {
        throw new IllegalStateException(getDisplayName() + " has been closed already");
      } else {
        throw new IllegalStateException(getDisplayName() + " has not been refreshed yet");
      }
    }
  }
}
```

아직 빈이 인스턴스화 되지 않았기 때문에 빈은 존재하지 않지만 그래도 빈을 찾는 행위가 가능해졌다.
이루에 propertySource와 pre-refresh애플리케이션 리스너들, ApplicationEvents를 초기화한다.

<br>

# 2. Beanfactory 준비 단계

실제 스프링의 빈들은 beanFactory에서 관리되고 애플리케이션 컨텍스트는 빈 관련 요청이 오면 이를 beanFactory로 위임한다.
prepareBeanFactory 메소드에서는 beanFactory가 동작하기 위한 준비 작업들이 진행된다.

```java
public abstract class AbstractApplicationContext {
  protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //컨텍스트의 클래스 로더 등을 사용하도록 내부 Bean 팩토리에 지시한다
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    //컨텍스트 콜백으로 Bean 팩토리를 구성한다
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    //BeanFactory 인터페이스는 일반 팩토리에서 확인 가능한 유형으로 등록되지 않았다
    //MessageSource가 autowiring을 찾기 위해 빈으로 등록되었다
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    //내부 Bean을 감지하기 위한 post-processor를 ApplicationListener로 등록합니다.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    //LoadTimeWeaver를 감지하고 발견되면 weaving을 준비합니다.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      //Type-Match를 위해 임시 ClassLoader를 설정합니다.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
      beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
  }
}
```

가장 먼저 빈 클래스들을 불러오기 위한 클래스 로더를 셋 해주고 프로퍼티를 처리하는 spEL Resolver와 PropertyEditory를 빈으로 추가한다.
그리고 위존성 주입을 무시할 인터페이스들(Aware Interface)을 등록하고 BeanFactory와 ApplicationContext같은 특별한 타입들을 빈으로 등록하기 위한 작업을 진행한 후에 환경 변수들을
빈으로 등록하며 마무리한다.

<br>

Aware 인터페이스는 스프링 프레임워크의 내부 동작에 접근하기 위한 인터페이스이다.
Aware 인터페이스에는 접근할 대상을 주힙해주는 수정자 메소드(Setter)가 존재한다.
예를 들어 ApplicationContextAware에는 setApplicationContext메소드가 있어서 ApplicationContext를 주입할 수 있다.
이러한 Aware 인터페이스는 Spring 2.0에 추가되었는데 과거에는 ApplicationContext나 BeanFactory등의 구현체를 주입 받을 수 없었다.
그래서 해당 구현체들에 접근하려면 Aware 인터페이스를 만들어 주입해주어야 했지만 지금은 바로 주입 가능하니 사용하지 않는다.
그래도 객체가 생성된 이후에 주입해주어야 할 때 사용할 수 있다.

<br>

# 3. BeanFactory의 후처리 진행

빈 팩토리 준비가 끝났으면 빈 팩토리를 후처리 한다.
예를 들어 서블릿 기반의 웹 애플리케이션이라면 서블릿 관련 클래스들도 빈으로 등록되어야 하므로 빈 팩토리에 추가 작업이 필요하다.
이러한 이유로 스프링은 이 과정에 다시 한 번 템플릿 메소드 패턴을 적용해서 각각의 애플리케이션 타입에 맞는 후처리를 진행하도록 하고 있다.
빈 팩토리의 후처리가 마무리되면 이제 빈 팩토리가 동작할 준비가 끝난 것이다.

<br>

# 4. BeanFactoryPostProcessor 실행

> BeanFactoryPostProcessor는 빈을 탐색하는 것처럼 빈 팩토리가 준비된 후에 해야하는 후처리기들을 실행하는 것이다

* 대표적으로 싱글톤 객체로 인스턴스화할 빈을 탐색하는 작업을 진행한다
* 스프링은 인스턴스화를 진행할 빈의 목록(Bean Definition)을 로딩하는 작업과 실제 인스턴스화를 하는 작업을 나눠서 처리한다
  * 이때 인스턴스로 만들 빈의 목록을 찾는 단계가 이것(BeanFactoryPostProcessor 실행)에 속한다
* 빈 목록은 @Configuration 클래스를 파싱해서 가져오는데, `BeanFactoryProcessor`의 구현체 중 하나인 `ConfigurationClassPostProcessor`가 진행한다
  * 아래에서는 `BeanDefinitionRegistryPostProcessor`이 `BeanFactoryProcessor`을 상속받는다
* `ConfigurationAnnotationProcessor`의 `processConfigBeanDefinitions`를 통해 파싱이 진행된다

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor {
  public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    //1. 파싱을 진행할 설정 빈 이름을 찾는다 -> 메인 클래스만 남게 된다
    for (String beanName : candidateNames) {
      BeanDefinition beanDef = registry.getBeanDefinition(beanName);
      if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
        if (logger.isDebugEnabled()) {
          logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
        }
      } else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
        configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
      }
    }

    //2. 파싱할 클래스가 없으면 종료함
    if (configCandidates.isEmpty()) {   //@Configuration 클래스가 발견되지 않으면 즉시 반환
      return;
    }

    //3. @Order를 참조해 파싱할 클래스들을 정렬한다
    configCandidates.sort((bd1, bd2) -> {   //해당되는 경우 이전에 결정된 @Order 값을 기준으로 정렬
      int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
      int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
      return Integer.compare(i1, i2);
    });

    //4. 빈 이름 생성 전략을 찾아서 처리한다
    //포함된 애플리케이션 컨텍스트를 통해 제공된 사용자 정의 Bean 이름 생성 전략을 감지합니다.
    // Detect any custom bean name generation strategy supplied through the enclosing application context
    SingletonBeanRegistry sbr = null;
    if (registry instanceof SingletonBeanRegistry _sbr) {
      sbr = _sbr;
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

    //5. 파서를 생성하고 모든 @Configuration 클래스를 파싱한다
    // Parse each @Configuration class
    ConfigurationClassParser parser = new ConfigurationClassParser(         //파서 생성
      this.metadataReaderFactory, this.problemReporter, this.environment,
      this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
      StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
      parser.parse(candidates);
      parser.validate();

      Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
      configClasses.removeAll(alreadyParsed);

      //모델을 읽고 해당 콘텐츠를 기반으로 Bean Definitions를 생성합니다.
      if (this.reader == null) {
        this.reader = new ConfigurationClassBeanDefinitionReader(
          registry, this.sourceExtractor, this.resourceLoader, this.environment,
          this.importBeanNameGenerator, parser.getImportRegistry());
      }
      this.reader.loadBeanDefinitions(configClasses);
      alreadyParsed.addAll(configClasses);
      processConfig.tag("classCount", () -> String.valueOf(configClasses.size())).end();

      candidates.clear();
      if (registry.getBeanDefinitionCount() > candidateNames.length) {
        String[] newCandidateNames = registry.getBeanDefinitionNames();
        Set<String> oldCandidateNames = Set.of(candidateNames);
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

    //ImportAware @Configuration 클래스를 지원하려면 ImportRegistry를 Bean으로 등록한다
    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
      sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    //필요한 경우 PropertySourceDescriptors를 미리 저장하여 제공합니다.
    this.propertySourceDescriptors = parser.getPropertySourceDescriptors();

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory cachingMetadataReaderFactory) {
      //외부에서 제공되는 MetadataReaderFactory에서 캐시를 지웁니다.(이건 작동하지 않는다)
      //공유 캐시의 경우 ApplicationContext에 의해 지워지기 때문이다
      cachingMetadataReaderFactory.clearCache();
    }
  }
}
```

빈 팩토리의 `beanDefinitionNames`는 저장된 빈 이름 목록을 조회하는데 아직 우리가 추가한 빈들이 파싱되지 않았으므로 현재 애플리케이션 메인 클래스와 스프링에서 정의한 클래스들만이 존재한다.
그리고 위의 필터링 과정을 거쳐 `configCandidates`에는 메인 애플리케이션 클래스만이 남게 된다.
메인 클래스의 `@SpringBootApplication`이 가지고 있는 `@SpringBootApplication`이 `@SpringBootConfiguratrion(@Configurarion)`을 가지고 있기
때문이다.

<br>

그리고 클래스를 파싱하는 `ConfigurationClassParser`의 `parse()`를 통해 메인 클래스 하위의 빈들을 찾아 모두 빈 정보 객체(Bean Definition)으로 만들어 등록한다.
`parse`가 진행되고 나면 이제 `beanDefinitionNames`로 우리가 만든 빈 정보로 불러와진다.
아직 객체로 인스턴스화 하지는 않았으므로 `beanFactory`에 빈이 존재하지는 않는다.
빈 목록을 불러오는 작업은 `BeanFactoryPostProcessor`구현제 중 하나인 `CofigurationAnnotationProcessor`가 담당할 뿐이고, 그 외에 등록된
다른 `BeanFactoryPostProcessor` 모두 객체로 만들고 실행시킨다.

<br>

**즉 이 단계는 `BeanFactory`가 준비되고 빈을 인스턴스화 하기 전의 중간 단계로써 빈의 목록을 불러오고 불러온 빈의 메타 정보를 조작하기 위한 `BeanFactoryPostProcessor`를 객체로
만들어 실행 시키는 것이다.**

<br>

# 5. BeanPostProcessor 등록

> 빈들이 생성되고 나서 빈의 내용이나 빈 자체를 변경하기 위한 빈 후처리기인 BeanPostProcessor를 등록해주고 있다.

```
registerBeanPostProcessors(beanFactory);
beanPostProcess.end();
```

* `@Value`, `@PostConstruct`, `@Autowired` 등이 `BeanPostProcessor`에 의해 처리된다.
  * 이를 위해 BeanPostProcessor 구현제들이 등록된다
  * ex) `CommonAnnotationBeanPostProcessor`는 `@PostConstruct`와 `@PreDestroy`를 처리하기 위해 등록된다
* 그러므로 **의존성 주입은 빈 후처리기에 의해 처리되는 것이다**
* **실제 빈 대신에 프록시 빈으로 교체되는 작업 역시 빈 후처리기에 의해 처리된다**
* BeanPostProcessor는 빈이 생성된 후 초기화 메소드가 호출되기 직전과 호출된 직후에 처리 가능한 2개의 메소드를 제공한다

<br>

# 6. MessageSource 및 Event Multicaster 초기화

BeanPostProcessor를 등록한 후에 다국어 처리를 위한 MessageSource와 ApplicationListener에 event를 publish하기 위한 Event Multicaster를 초기화한다.

```
initMessageSource();           
initApplicationEventMulticaster();
```

<br>

# 7. onRefresh(웹 서버 생성 및 실행)

* SpringBoot는 기존 SpringFramework와 달리 내장 웹 서버를 탑재하고 있다
* 애플리케이션이 시작될 때 웹 서버 객체를 만들어서 실행해야 한다
  * 문제는 애플리케이션 타입에 따라 웹 서버 생성 로직이 달라져야 한다는 것이다
    * 웹 애플리케이션이 아니라면 웹 서버가 없다
    * 서블릿 웹인 경우에는 톰캣으로 만든다
    * 리액티브 웹인 경우 리액터 네티로 만든다
    * 즉 애플리케이션 컨텍스트에 따라 다른 웹 서버 로직이 호출되어야 한다
* 스프링은 위 문제를 템플핏 메소드 패턴을 적용해서 문제를 해결한다.
* 모든 애플리케이션 컨텍스트에 동일한 로직들(빈을 찾고 객체로 만들어 후처리하는 것)은 큰 틀에서 refresh라는 메소드로 부모 클래스에 작성해 둔다
* 달라져야 하는 부분들(서로 다른 웹 서버 생성)은 onRefresh메소드를 오버라이딩하도록 하는 것이다

```java
public class ServletWebServerApplicationContext {
  @Override
  protected void onRefresh() {
    super.onRefresh();
    try {
      createWebServer();
    } catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start web server", ex);
    }
  }

  private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
      StartupStep createWebServer = getApplicationStartup().start("spring.boot.webserver.create");
      ServletWebServerFactory factory = getWebServerFactory();
      createWebServer.tag("factory", factory.getClass().toString());
      this.webServer = factory.getWebServer(getSelfInitializer());
      createWebServer.end();
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
        new WebServerGracefulShutdownLifecycle(this.webServer));
      getBeanFactory().registerSingleton("webServerStartStop",
        new WebServerStartStopLifecycle(this, this.webServer));
    } else if (servletContext != null) {
      try {
        getSelfInitializer().onStartup(servletContext);
      } catch (ServletException ex) {
        throw new ApplicationContextException("Cannot initialize servlet context", ex);
      }
    }
    initPropertySources();
  }
}
```

서블릿 기반에서는 톰캣으로 웹 서버를 만들어주어야 하기 때문에 웹 서버를 생성하기 위해 팩토리 객체에게 위임하는데, getWebServer메소드를 호출하면 내부에서 톰캣 객체를 생성해서 반환한다.

```java
public class TomcatServletWebServerFactory {
  @Override
  public WebServer getWebServer(ServletContextInitializer... initializers) {
    if (this.disableMBeanRegistry) {
      Registry.disableRegistry();
    }
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    for (LifecycleListener listener : this.serverLifecycleListeners) {
      tomcat.getServer().addLifecycleListener(listener);
    }
    Connector connector = new Connector(this.protocol);
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
      tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
  }
}
```

<br>

# 8. ApplicationListener 조회 및 등록

```
registerListeners();
```

이후에는 `ApplicationListener`의 구현체들을 찾아서 `EventMultiCaster`에 등록한다.

<br>

# 9. 빈들의 인스턴스화 및 후처리

```
finishBeanFactoryInitialization(beanFactory);
```

등록된 `Bean Definition`을 바탕으로 객체를 생성할 차례이다.
`Bean Definition`에는 빈 이름, 스코프 등과 같은 정보가 있어서 이를 바탕으로 객체를 생성하게 된다.

```java
public abstract class AbstractApplicationContext {
  protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    //이 컨텍스트에 대한 변환 서비스를 초기화합니다.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
      beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
        beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    //BeanFactoryPostProcessor가 없는 경우 기본 내장 값 확인자(default embedded value resolver)를 등록한다
    if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    //변환기를 조기에 등록할 수 있도록 LoadTimeWeaverAware Bean을 일찍 초기화한다
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
    }

    //type matching을 위해 임시 ClassLoader 사용을 중지한다
    beanFactory.setTempClassLoader(null);

    //추가 변경을 예상하지 않고 모든 Bean 정의 메타데이터를 캐시하도록 허용한다
    beanFactory.freezeConfiguration();

    //나머지(lazy-init가 아닌) 싱글톤을 모두 인스턴스화 한다
    beanFactory.preInstantiateSingletons();
  }
}
```

객체를 생성하는 부분은 가장 마지막 줄은 `preInstantiateSingletons()`인데 그 전에 더 남은 빈 팩토리 작업과 누락된 빈 정보가 없으므로 빈 팩토리의 설정과 BeanDefinitionNames를
freeze해주고 있다.
그리고 객체를 생성하는데 `BeanFactory`의 구현체인 `DefaultListableBeanFactory`를 통해 해당 로직을 보자.

```java
public class DefaultListableBeanFactory {
  @Override
  public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        if (isFactoryBean(beanName)) {
          Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
          if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
            getBean(beanName);
          }
        } else {
          getBean(beanName);
        }
      }
    }

    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
        StartupStep smartInitialize = getApplicationStartup().start("spring.beans.smart-initialize")
          .tag("beanName", beanName);
        smartSingleton.afterSingletonsInstantiated();
        smartInitialize.end();
      }
    }
  }
}
```

여기서 getBean()메소드만 보도록 하자.
DefaultListableBeanFactory의 getBean내부에서는 요청을 AbstractBeanFactory의 doGetBean으로 위임한다.

```java
public abstract class AbstractBeanFactory {
  protected <T> T doGetBean(
    String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
    throws BeansException {

    String beanName = transformedBeanName(name);
    Object beanInstance;

    //빈이 이미 등록되었거나 캐싱된 빈이 존재하는지 검사한다
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
        if (isSingletonCurrentlyInCreation(beanName)) {
          logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
            "' that is not fully initialized yet - a consequence of a circular reference");
        } else {
          logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
        }
      }
      //이미 빈이 등록되었거나 캐싱된 빈이 존재하는 경우 생성하지 않는다
      beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
      // 이 Bean 인스턴스를 이미 생성 중인 경우 실패합니다.
      // 이것은 아마도 순환 참조 내에 있을 것이다
      if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
      }

      //이 팩토리에 Bean Definition이 있는지 확인한다
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        //찾을 수 없음 -> 상위(parent) 항목을 확인
        String nameToLookup = originalBeanName(name);
        if (parentBeanFactory instanceof AbstractBeanFactory abf) {
          return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
        } else if (args != null) {
          //명시적인 인수를 사용하여 부모에게 위임
          return (T) parentBeanFactory.getBean(nameToLookup, args);
        } else if (requiredType != null) {
          //인수 없음 -> 표준 getBean 메소드에 위임
          return parentBeanFactory.getBean(nameToLookup, requiredType);
        } else {
          return (T) parentBeanFactory.getBean(nameToLookup);
        }
      }

      if (!typeCheckOnly) {
        markBeanAsCreated(beanName);
      }

      StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
        .tag("beanName", name);
      try {
        if (requiredType != null) {
          beanCreation.tag("beanType", requiredType::toString);
        }
        RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd, beanName, args);

        //현재 Bean이 의존하는 Bean의 초기화를 보장한다
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
            } catch (NoSuchBeanDefinitionException ex) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
            }
          }
        }

        // Create bean instance.
        if (mbd.isSingleton()) {
          sharedInstance = getSingleton(beanName, () -> {
            try {
              return createBean(beanName, mbd, args);
            } catch (BeansException ex) {
              // Explicitly remove instance from singleton cache: It might have been put there
              // eagerly by the creation process, to allow for circular reference resolution.
              // Also remove any beans that received a temporary reference to the bean.
              destroySingleton(beanName);
              throw ex;
            }
          });
          beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        } else if (mbd.isPrototype()) {
          //프로토타입인 경우에 다흔 객체를 생성한다
          Object prototypeInstance = null;
          try {
            beforePrototypeCreation(beanName);
            prototypeInstance = createBean(beanName, mbd, args);
          } finally {
            afterPrototypeCreation(beanName);
          }
          beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        } else {
          String scopeName = mbd.getScope();
          if (!StringUtils.hasLength(scopeName)) {
            throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
          }
          Scope scope = this.scopes.get(scopeName);
          if (scope == null) {
            throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
          }
          try {
            Object scopedInstance = scope.get(beanName, () -> {
              beforePrototypeCreation(beanName);
              try {
                return createBean(beanName, mbd, args);
              } finally {
                afterPrototypeCreation(beanName);
              }
            });
            beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
          } catch (IllegalStateException ex) {
            throw new ScopeNotActiveException(beanName, scopeName, ex);
          }
        }
      } catch (BeansException ex) {
        beanCreation.tag("exception", ex.getClass().toString());
        beanCreation.tag("message", String.valueOf(ex.getMessage()));
        cleanupAfterBeanCreationFailure(beanName);
        throw ex;
      } finally {
        beanCreation.end();
        if (!isCacheBeanMetadata()) {
          clearMergedBeanDefinition(beanName);
        }
      }
    }

    return adaptBeanInstance(name, beanInstance, requiredType);
  }
}
```

* doGetBean() 내부에서는 해당 빈 이름으로 만들어진 빈이 존재하는지 검사하여 있으면 꺼내서 반환하고 없으면 생성해서 반환한다.
* AbstractBeanFactory의 코드에서 싱글톤 객체를 만드는 createBean()부분은 추상 메소드이며 세부 구현은 AbstractAutowireCapableBeanFactory를 참조해야 한다.
  * 템플릿 메소드 패턴이 적용된 것이다
* 마지막으로 BeanUtils의 instantiateClass에서 객체를 만들어준다
  * 내부적으로 자바 리플렉션 패키지의 생성자 클래스인(Constructor)를 통해 인스턴스화한다

```java
public class BeanUtils {
  public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
      ReflectionUtils.makeAccessible(ctor);
      if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
        return KotlinDelegate.instantiateClass(ctor, args);
      } else {
        int parameterCount = ctor.getParameterCount();
        Assert.isTrue(args.length <= parameterCount, "Can't specify more arguments than constructor parameters");
        if (parameterCount == 0) {
          return ctor.newInstance();
        }
        Class<?>[] parameterTypes = ctor.getParameterTypes();
        Object[] argsWithDefaultValues = new Object[args.length];
        for (int i = 0; i < args.length; i++) {
          if (args[i] == null) {
            Class<?> parameterType = parameterTypes[i];
            argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
          } else {
            argsWithDefaultValues[i] = args[i];
          }
        }
        return ctor.newInstance(argsWithDefaultValues);
      }
    } catch (InstantiationException ex) {
      throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    } catch (IllegalAccessException ex) {
      throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    } catch (IllegalArgumentException ex) {
      throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
    } catch (InvocationTargetException ex) {
      throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
    }
  }
}
```

생성자로 객체를 만들었으면 @Value, @PostConstruct, @Autowired에 의한 값은 모두 Null이다.
그러므로 만들어진 객체를 대상으로 BeanPostProcessor를 적용해 @Value, @PostConstruct, @Autowired 등을 처리해주고 객체의 생성을 마무리한다.

<br>

# 10. refresh 마무리 단계

모든 빈들을 인스턴스화 하였다면 이제 마무리를 해야 한다.
여기에서는 애플리케이션 컨텍스트를 준비하기 위해 사용되었던 `resourceCache`를 제거하고, `Lifecycle Processor`를 초기화하여 refresh를 전파하고 최종 이벤트를 전파하며 마무리된다.

* `Lifecycle Processor`에는 웹 서버와 관련된 부분이 있어서 **refresh가 전파되면 웹 서버가 실행**된다.

```
finishRefresh();
```

<br>

## 만약 refreshContext가 실패한 경우의 마무리 단계

언제나 실패를 하는 가능성은 존재한다(예를 들어 톰캣 포트가 이미 사용 중인 포트일 때).
그러면 스프링은 catch 구문에서 모든 빈들을 제거하고 `active = false`로 변경하고 애플리케이션을 종료한다.

```java
public abstract class AbstractApplicationContext {
  @Override
  public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

      //코드 생략

      try {
        //이 부분에서 에러 발생
      } catch (RuntimeException | Error ex) {
        if (logger.isWarnEnabled()) {
          logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
        }
        //dangling resources를 방지하기 위해 생성된 싱글톤을 Destroy한다
        // Destroy already created singletons to avoid dangling resources.
        destroyBeans();

        // active 플래그를 리셋한다
        cancelRefresh(ex);

        //호출자에게 예외를 전파한다
        throw ex;
      } finally {
        contextRefresh.end();
      }
    }
  }
}
```













