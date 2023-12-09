---
title: SpringApplication.run() 분석
date: 2023-12-09 20:00:00 +0800
categories: [Spring, Analyze]
tags: [spring, code, analyze]
render_with_liquid: false
---


SpringApplication의 run 메소드는 아래의 세 종류가 있다. 하나씩 자세히 보도록 하자.

```java
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {
    //1. 실행 시간 측정 시작
    Startup startup = Startup.create();
    if (this.registerShutdownHook) {
      SpringApplication.shutdownHook.enableShutdownHookAddition();
    }

    //2. BootstrapContext 생성
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;

    //3. Java AWT Headless Property 설정
    configureHeadlessProperty();

    //4. 스프링 애플리케이션 리스너 조회 및 starting 처리
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
      //5. Arguments 래핑 및 Environment 준비
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

      //6. 배너 출력
      Banner printedBanner = printBanner(environment);

      //7. 애플리케이션 컨텍스트 생성
      context = createApplicationContext();
      context.setApplicationStartup(this.applicationStartup);

      //8. Context 준비 단계
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

      //9. Context Refresh 단계
      refreshContext(context);

      //10. Context Refresh 후처리 단계
      afterRefresh(context, applicationArguments);

      //11. 실생 시간 출력 및 리스너 stared 처리
      startup.started();
      if (this.logStartupInfo) {
        new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), startup);
      }
      listeners.started(context, startup.timeTakenToStarted());

      //12. Runners 실행
      callRunners(context, applicationArguments);
    } catch (Throwable ex) {
      if (ex instanceof AbandonedRunException) {
        throw ex;
      }
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
    }
    try {
      if (context.isRunning()) {
        listeners.ready(context, startup.ready());
      }
    } catch (Throwable ex) {
      if (ex instanceof AbandonedRunException) {
        throw ex;
      }
      handleRunFailure(context, ex, null);
      throw new IllegalStateException(ex);
    }
    return context;
  }
}
```

순서는 다음과 같다.

1. 실행 시간 측정 시작
2. BootStrapContext 생성
3. Java AWT Headless Property 설정
4. 스프링 애플리케이션 리스너 조회 및 starting 처리
5. Argument 래핑 및 Environment 준비
6. 배너 출력
7. 애플리케이션 컨텍스트 생성
8. Context 준비단계
9. Context Refresh 단계
10. Context Refresh 후처리 단계
11. 실행 시간 출력 및 리스너 started 처리
12. Runners 실행

이전의 스프링 버전에서는 IgnoreJavaBeans라는 과정도 있었다

```java
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {

    // 1. StopWatch로 실행 시간 측정 시작
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // 2. BootStrapContext 생성
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;

    // 3. Java AWT Headless Property 설정
    configureHeadlessProperty();

    // 4. 스프링 애플리케이션 리스너 조회 및 starting 처리
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {

      // 5. Arguments 래핑 및 Environment 준비
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

      // 6. IgnoreBeanInfo 설정
      configureIgnoreBeanInfo(environment);

      // 7. 배너 출력
      Banner printedBanner = printBanner(environment);

      // 8. 애플리케이션 컨텍스트 생성
      context = createApplicationContext();
      context.setApplicationStartup(this.applicationStartup);

      // 9. Context 준비 단계
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

      // 10. Context Refresh 단계
      refreshContext(context);

      // 11. Context Refresh 후처리 단계
      afterRefresh(context, applicationArguments);

      // 12. 실행 시간 출력 및 리스너 started 처리
      stopWatch.stop();
      if (this.logStartupInfo) {
        new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);

      // 13. Runners 실행
      callRunners(context, applicationArguments);
    } catch (Throwable ex) {
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
    }

    try {
      listeners.running(context);
    } catch (Throwable ex) {
      handleRunFailure(context, ex, null);
      throw new IllegalStateException(ex);
    }
    return context;
  }
}
```

<br>

# 1. 실행 시간 측정 시작

```
long startTime = System.nanoTime();
```

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2023-12-09T22:17:14.909+09:00  INFO 45064 --- [           main] c.e.d.DemoSpringProjectApplication       : Starting DemoSpringProjectApplication using Java 18.0.1.1 with PID 45064 (C:\Users\usr\IdeaProjects\demoSpringProject\build\classes\java\main started by usr in C:\Users\usr\IdeaProjects\demoSpringProject)
2023-12-09T22:17:14.910+09:00  INFO 45064 --- [           main] c.e.d.DemoSpringProjectApplication       : No active profile set, falling back to 1 default profile: "default"
2023-12-09T22:17:15.508+09:00  INFO 45064 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2023-12-09T22:17:15.514+09:00  INFO 45064 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-12-09T22:17:15.514+09:00  INFO 45064 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
2023-12-09T22:17:15.553+09:00  INFO 45064 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-12-09T22:17:15.553+09:00  INFO 45064 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 611 ms
2023-12-09T22:17:15.825+09:00  INFO 45064 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-12-09T22:17:15.831+09:00  INFO 45064 --- [           main] c.e.d.DemoSpringProjectApplication       : Started DemoSpringProjectApplication in 1.19 seconds (process running for 1.56)
```

스프링을 실행하면 제일 기본적으로 나오는 출력 로그이다. 여기서 아래에서 3번째 줄을 보면 `run` 안에서 가장 먼저 측정을 시작함을 알 수 있다.

```
//스프링 2.6.3 코드
StopWatch stopWatch = new StopWatch();
stopWatch.start();
```

* 스프링 2.6.3 에서는 `StopWatch`로 실행 시간 측정을 시작한다

<br>

# 2. BootStrapContext 생성

> 애플리케이션 컨텍스트가 준비될 때까지 환경 변수들을 관리하는 스프링의 Environment 객체를 후처리하기 위한 임시 컨텍스트

그 외에도 SpringApplication.run()이 호출될 때나 애플리케이션 컨텍스트가 준비 마무리되었을 때 호출하기 위한 리스너를 등록할 수 있다.

```
DefaultBootstrapContext bootstrapContext = createBootstrapContext();
ConfigurableApplicationContext context = null;
```

```java
public class SpringApplication {
  private DefaultBootstrapContext createBootstrapContext() {
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
    return bootstrapContext;
  }
}
```

동작 순서는 아래와 같다.

1. DefaultBootstrapContext 객체를 생성한다
2. spring.factories에서 불러왔던 bootstrapRegistryInitializers를 모두 DefaultBootstrapContext에 초기화한다
3. ApplicationContext를 선언해주는데, null로만 선언해주고 객체의 생성은 다음 과정에서 해준다.

<br>

# 3. Java AWT Headless Property 설정

> 디스플레이 장치가 없는 서버 환경에서 UI 클래스를 사용할 수 있도록 하는 옵션이다

* 서버에서 이미지를 만들어서 반환해주어야 하는 경우 이미지 관련 클래스 등이 필요할 수 있는데 이때 Headless 모드를 주지 않으면 해당 클래스를 사용할 수 없고 에러가 발생한다
* 이때 Headless 모드를 true로 주면 사용 불가능한 UI 클래스들을 특별 객체로 만들어준다
  * 대표적으로 java.awt.Toolkit이 있다
* 만약 Headless 모드인데 디스플레이 장치가 필수인 기능(예를 들어 화면에 띄우는 기능)을 호출한다면 Headless 에러를 던진다
* SpringBoot에서는 기본적으로 headless 모드가 true라서 java.awt 등의 패키지로 이미지 관련 처리를 할 수 있다

```java
public class SpringApplication {
  private boolean headless = true;

  private void configureHeadlessProperty() {
    System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
      System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
  }
}
```

<br>

# 4. 스프링 애플리케이션 리스너 조회 및 starting 처리

> 애플리케이션 컨텍스트를 준비할 때 호출되어야 하는 리스너들을 찾아서 BootStrapContext의 리스너로 실행하게 해준다

* 생성 시간이 긴 객체의 경우 객체 생성을 위한 리스너를 만들어 등록하면 BootStrapContext가 애플리케이션 컨텍스트를 준비함과 동시에 객체를 생성하도록 한다
  * 이를 통해 Lazy하게 접근 가능하다
  * Lazy : 필요할 때 객체를 생성해서 접근한다

```java
//스프링 3.2.0
public class SpringApplication {
  private SpringApplicationRunListeners getRunListeners(String[] args) {
    ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);
    argumentResolver = argumentResolver.and(String[].class, args);
    List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,
      argumentResolver);
    SpringApplicationHook hook = applicationHook.get();
    SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;
    if (hookListener != null) {
      listeners = new ArrayList<>(listeners);
      listeners.add(hookListener);
    }
    return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);
  }

  private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);
  }
}
```

리스너를 조회하는 코드를 보면 `spring.factories`에서 `SpringApplicationRunListener`를 Key로 대상을 도회하고 있는데 여기서 왜 스프링이 `loadFactoryNames`에
캐싱을 적용했는지 알 수 있다.

* 이전에 `SpringApplication`의 생성과 초기화를 공부하면서 `BootstrapRegistryInitializer`단계에서 모든 jar 파일에서 `spring.factorie`s를 조회해 캐싱해둔
  상태이므로 메모리에서 빠르게 조회할 수 있다
* 조회된 대상을 객체로 만들어 `BootStrapContext`에 연결해서 실행한다
* 현재 버전 기준으로 `SpringApplicationRunListener`의 구현제로는 컨텍스트 시작 중에 리스너에게 애플리케이션의 시작 및 실행 단계를 알리기
  위한 `EventPublishingRunListener`밖에 존재하지 않는다.
* 여기서 스프링은 인터페이스를 분리하여 추상화하려고 매우 노력하는 것을 알 수 있다

```java
//스프링 2.6.3
public class SpringApplication {
  private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[]{SpringApplication.class, String[].class};
    return new SpringApplicationRunListeners(logger,
      getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args),
      this.applicationStartup);
  }

  private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
  }
}
```

<br>

# 5. Argument 래핑 및 Environment 준비

```java
public class SpringApplication {
  private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                     DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
    // Create and configure the environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());    //래핑 해주는 부분
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(bootstrapContext, environment);
    DefaultPropertiesPropertySource.moveToEnd(environment);
    Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
      "Environment prefix cannot be set via properties.");
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
      EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
      environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
  }
}
```

제일 먼저 `String[]` 형태의 인자를 스프링 부트를 위한 인자인 `ApplicationArgument`로 래핑해준다.
그리고 이를 Environment를 준비하는 prepareEnvironment에 넘겨준다.

```java
public interface ApplicationArguments {

  String[] getSourceArgs();

  Set<String> getOptionNames();

  boolean containsOption(String name);

  List<String> getOptionValues(String name);

  List<String> getNonOptionArgs();

}
```

```java
//스프링 3.2.0
public class SpringApplication {
  private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
      return this.environment;
    }
    ConfigurableEnvironment environment = this.applicationContextFactory.createEnvironment(this.webApplicationType);
    if (environment == null && this.applicationContextFactory != ApplicationContextFactory.DEFAULT) {
      environment = ApplicationContextFactory.DEFAULT.createEnvironment(this.webApplicationType);
    }
    return (environment != null) ? environment : new ApplicationEnvironment();
  }
}
```

그 다음에 만들어진 `Environment`에 `Property`나 Profile 등과 같은 값들을 셋 해주고 `SpringApplication`에 바인딩 해준다.

```java
//스프링 2.6.3
public class SpringApplication {
  private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
      return this.environment;
    }
    switch (this.webApplicationType) {
      case SERVLET:
        return new ApplicationServletEnvironment();
      case REACTIVE:
        return new ApplicationReactiveWebEnvironment();
      default:
        return new ApplicationEnvironment();
    }
  }
}
```

4단계에 스프링 3.2.0 코드를 보면

```
ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);
argumentResolver = argumentResolver.and(String[].class, args);
```

이러한 부분이 나온다. 여기에도 `String[].class`이 나오는데 이는 `Argument`를 래핑하는 과정이 아니다. 이것은 리스너를 불러오기 위한 설정 값을 가져오는 것이다.

<br>

# 6. 배너 출력

> 애플리케이션이 시작되면 스프링 부트 배너가 출력된다.

* 이는 커스터마이징 가능하다
  * 텍스트
  * 색상
  * gif 형태의 배너

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
```

<br>

# 7. 애플리케이션 컨텍스트 생성

> 팩토리 클래스에 생성을 위임한다.

```java
public class SpringApplication {
  private ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;

  protected ConfigurableApplicationContext createApplicationContext() {
    return this.applicationContextFactory.create(this.webApplicationType);
  }
}
```

애플리케이션 컨텍스트 생성을 위해서도 webApplicationType가 사용되는데 해당 값을 팩토리 클래스의 create 메소드의 인자로 넘겨주고 있다.
팩토리 클래스에서 해당 타입을 통해 애플리케이션 타입 중 하나를 생성하여 반환한다.

```java
//스프링 3.2.0
@FunctionalInterface
public interface ApplicationContextFactory {

  ApplicationContextFactory DEFAULT = new DefaultApplicationContextFactory();

  default Class<? extends ConfigurableEnvironment> getEnvironmentType(WebApplicationType webApplicationType) {
    return null;
  }

  default ConfigurableEnvironment createEnvironment(WebApplicationType webApplicationType) {
    return null;
  }

  ConfigurableApplicationContext create(WebApplicationType webApplicationType);

  static ApplicationContextFactory ofContextClass(Class<? extends ConfigurableApplicationContext> contextClass) {
    return of(() -> BeanUtils.instantiateClass(contextClass));
  }

  static ApplicationContextFactory of(Supplier<ConfigurableApplicationContext> supplier) {
    return (webApplicationType) -> supplier.get();
  }

}
```

이전 스프링 버전에서는 3가지 애플리케이션 타입 중 하나를 생성해서 반환했다.

```java
//스프링 2.6.3
@FunctionalInterface
public interface ApplicationContextFactory {

  ApplicationContextFactory DEFAULT = (webApplicationType) -> {
    try {
      switch (webApplicationType) {
        case SERVLET:
          return new AnnotationConfigServletWebServerApplicationContext();
        case REACTIVE:
          return new AnnotationConfigReactiveWebServerApplicationContext();
        default:
          return new AnnotationConfigApplicationContext();
      }
    } catch (Exception ex) {
      throw new IllegalStateException("Unable create a default ApplicationContext instance, "
        + "you may need a custom ApplicationContextFactory", ex);
    }
  };

  ConfigurableApplicationContext create(WebApplicationType webApplicationType);

  static ApplicationContextFactory ofContextClass(Class<? extends ConfigurableApplicationContext> contextClass) {
    return of(() -> BeanUtils.instantiateClass(contextClass));
  }

  static ApplicationContextFactory of(Supplier<ConfigurableApplicationContext> supplier) {
    return (webApplicationType) -> supplier.get();
  }
}
```

<br>

# 8. Context 준비 단계

> Context가 생성된 후에 해주어야 하는 후처리 작업들과 빈들을 등록하는 refresh 단계를 위한 전처리 작업 등이 수행된다

```java
public class SpringApplication {
  private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
                              ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
                              ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    addAotGeneratedInitializerIfNecessary(this.initializers);
    applyInitializers(context);
    listeners.contextPrepared(context);
    bootstrapContext.close(context);
    if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
      autowireCapableBeanFactory.setAllowCircularReferences(this.allowCircularReferences);
      if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
        listableBeanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
      }
    }
    if (this.lazyInitialization) {
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    if (this.keepAlive) {
      KeepAlive keepAlive = new KeepAlive();
      keepAlive.start();
      context.addApplicationListener(keepAlive);
    }
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    if (!AotDetector.useGeneratedArtifacts()) {
      // Load the sources
      Set<Object> sources = getAllSources();
      Assert.notEmpty(sources, "Sources must not be empty");
      load(context, sources.toArray(new Object[0]));
    }
    listeners.contextLoaded(context);
  }
}
```

1. 제일 먼저 앞에서 생성했던 Environment를 애플리케이션 컨텍스트에 설정해준다.
2. 이전의 작업들을 진행하면서 beanNameGenerator(빈 이름 지정 클래스), resourceLoader(리소스를 불러오는 클래스), conversionService(프로퍼티의 타입 변환)등이 생성되었으면
   싱글톤 빈으로 등록해준다.
3. SpringApplication 생성 단계에서 찾았던 initializer들을 inilialize해주는 작업이 진행된다
4. 애플리케이션 컨텍스트가 생성되고 Initializer들의 Initialize까지 진행되었으므로 더이상 BootStrapContext는 불필요하여 이를 종료해준다
5. 래핑한 ApplicationArgument와 배너 클래스를 빈으로 등록해준다
6. 순환 참조 여부와 빈 정보 오버라이딩 여부를 설정해준다
   - 기본적으로 순환 참조는 불가능하며 순환 참조가 발생할 경우 에러가 발생하며 애플리케이션이 종료된다
   - 이는 생성자 주입일 경우에만 실행 시점에 발생한다
   - @Autowired 필드 주입인 경우에는 호출 시에 발생한다
     - 이는 우리가 생성자 주입을 사용해야 하는 이유 중 하나이다
   - 마찬가지로 동일한 이름의 빈이 여러 개 있을 경우에도 에러가 발생하고 애플리케이션이 종료된다
   - SpringApplicationBuilder나 properies에서 해당 옵션을 바꿀 수 있지만 되도록이면 바꾸지 않는 것이 좋다
     - 기본 값은 false이다
7. LazyInitializeBean을 처리하는 빈 팩토리 후처리기 등록하며 소스를 불러온다
8. 컨텍스트 리스너들을 연결하면서 Context 준비 단계를 마무리한다

<br>

# 9. Context Refresh 단계

> 우리가 만든 빈들을 찾아서 등록하고 웹 서버를 만들어 실행하는 등의 핵심 작업들이 진행된다

* 이 단계를 거치면 모든 객체들이 싱글톤으로 인스턴스화 된다
* 만약 에러가 발생하면 등록된 모든 빈들을 제거한다
* 그래서 refresh가 진행되면 모든 빈이 인스턴스화 되거나 모든 빈이 존재하지 않거나 둘 중 하나이다
* 애플리케이션 컨텍스트에서 가장 중요한 단계이다

<br>

# 10. Context Refresh 후처리 단계

> refresh 이후에 후처리 해주는 단계

* 과거에는 애플리케이션 컨텍스트 생성 후에 초기화 작업을 해주는 ApplicationRunner, CommandLineRunner 를 호출하는 callRunners()가 내부에 존재했다
* 지금은 별도의 단계로 빠져있어 메소드가 비어있다

<br>

# 11. 실행 시간 출력 및 리스너 stared 처리

이후에 애플리케이션을 시작하는데 걸린 시간을 로그로 남기고 리스너들을 started 처리한다.

```
2023-12-09T22:17:15.831+09:00  INFO 45064 --- [           main] c.e.d.DemoSpringProjectApplication       : Started DemoSpringProjectApplication in 1.19 seconds (process running for 1.56)
```

<br>

# 12. Runners 실행

> 애플리케이션이 실행된 이후에 초기화 작업을 필요로 하는 경우가 많은데 이때 보통 사용하는 선택지이다

Runner를 구현하여 빈으로 등록하면 callRunner에서는 CommandLineRunner와 ApplicationRunner 구현체들을 찾아서 run 메소드를 실행시킨다

```java
//스프링 3.2.0
public class SpringApplication {
  private void callRunners(ApplicationContext context, ApplicationArguments args) {
    context.getBeanProvider(Runner.class).orderedStream().forEach((runner) -> {
      if (runner instanceof ApplicationRunner applicationRunner) {
        callRunner(applicationRunner, args);
      }
      if (runner instanceof CommandLineRunner commandLineRunner) {
        callRunner(commandLineRunner, args);
      }
    });
  }

  private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
    try {
      (runner).run(args);
    } catch (Exception ex) {
      throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
    }
  }

  private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
    try {
      (runner).run(args.getSourceArgs());
    } catch (Exception ex) {
      throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
    }
  }
}
```
```java
//스프링 2.6.3
public class SpringApplication{
  private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
      if (runner instanceof ApplicationRunner) {
        callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner) {
        callRunner((CommandLineRunner) runner, args);
      }
    }
  }
}
```

<br>

## Runner 예시
```java
@SpringBootApplication
public class DemoSpringApplication {
    @Bean
    ApplicationRunner run(ConditionEvaluationReport report){
        return args -> {
            System.out.println(
                    report.getConditionAndOutcomesBySource().entrySet().stream()
                            .filter(co -> co.getValue().isFullMatch())
                            .filter(co -> co.getKey().indexOf("Jmx") < 0)    //JMX 내용 제거
                            .map(co->{
                                co.getValue().forEach(c ->{
                                    System.out.println("\t"+c.getOutcome());
                                });
                                System.out.println();
                                return co;
                            }).count()
            );
        };
    }

    public static void main(String[] args) {
        SpringApplication.run(DemoSpringApplication.class, args);
    }

}
```
이처럼 Runner를 등록하면
```
Positive matches:
-----------------

   AopAutoConfiguration matched:
      - @ConditionalOnProperty (spring.aop.auto=true) matched (OnPropertyCondition)

   AopAutoConfiguration.ClassProxyingConfiguration matched:
      - @ConditionalOnMissingClass did not find unwanted class 'org.aspectj.weaver.Advice' (OnClassCondition)
      - @ConditionalOnProperty (spring.aop.proxy-target-class=true) matched (OnPropertyCondition)

   ApplicationAvailabilityAutoConfiguration#applicationAvailability matched:
      - @ConditionalOnMissingBean (types: org.springframework.boot.availability.ApplicationAvailability; SearchStrategy: all) did not find any beans (OnBeanCondition)

   GenericCacheConfiguration matched:
      - Cache org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration automatic cache type (CacheCondition)

이하 너무 기니까 생략
```
이렇게 나온다.
