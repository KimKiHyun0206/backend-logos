---
title: SpringApplication.run() 분석
date: 2023-12-09 20:00:00 +0800
categories: [Spring, Analyze]
tags: [spring, code, analyze]
render_with_liquid: false
---

```java
//스프링 3.2.0 코드
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      context.setApplicationStartup(this.applicationStartup);
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
      if (this.logStartupInfo) {
        new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
      }
      listeners.started(context, timeTakenToStartup);
      callRunners(context, applicationArguments);
    } catch (Throwable ex) {
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
    }
    try {
      Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
      listeners.ready(context, timeTakenToReady);
    } catch (Throwable ex) {
      handleRunFailure(context, ex, null);
      throw new IllegalStateException(ex);
    }
    return context;
  }

  public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[]{primarySource}, args);
  }

  public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
  }
}
```

SpringApplication의 run 메소드는 위의 세 종류가 있다. 하나씩 자세히 보도록 하자.

<br>

```java
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {
    //1. 실행 시간 측정 시작
    long startTime = System.nanoTime();

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

      //6. IgnoreBeanInfo 설정
      configureIgnoreBeanInfo(environment);

      //7. 배너 출력
      Banner printedBanner = printBanner(environment);

      //8. 애플리케이션 컨텍스트 생성
      context = createApplicationContext();
      context.setApplicationStartup(this.applicationStartup);

      //9. Context 준비 단계
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

      //10. Context Refresh 단계
      refreshContext(context);

      //11. Context Refresh 후처리 단계
      afterRefresh(context, applicationArguments);

      //12. 실행 시간 출력 및 리스너 started 처리
      Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
      if (this.logStartupInfo) {
        new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
      }
      listeners.started(context, timeTakenToStartup);

      //13. Runners 실행
      callRunners(context, applicationArguments);
    } catch (Throwable ex) {
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
    }
    try {
      Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
      listeners.ready(context, timeTakenToReady);
    } catch (Throwable ex) {
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
6. IgnoreBeanInfo 설정
7. 배너 출력
8. 애플리케이션 컨텍스트 생성
9. Context 준비 단계
10. Context Refresh 단계
11. Context Refresh 후처리 단계
12. 실행 시간 출력 및 리스너 started 처리
13. Runners 실행

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

2023-12-09T20:54:40.608+09:00  INFO 44856 --- [           main] c.e.d.DemoSpringProjectApplication       : Starting DemoSpringProjectApplication using Java 18.0.1.1 with PID 44856 (C:\Users\usr\IdeaProjects\demoSpringProject\build\classes\java\main started by usr in C:\Users\usr\IdeaProjects\demoSpringProject)
2023-12-09T20:54:40.609+09:00  INFO 44856 --- [           main] c.e.d.DemoSpringProjectApplication       : No active profile set, falling back to 1 default profile: "default"
2023-12-09T20:54:41.273+09:00  INFO 44856 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2023-12-09T20:54:41.281+09:00  INFO 44856 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-12-09T20:54:41.281+09:00  INFO 44856 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
2023-12-09T20:54:41.323+09:00  INFO 44856 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-12-09T20:54:41.323+09:00  INFO 44856 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 681 ms
2023-12-09T20:54:41.590+09:00  INFO 44856 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-12-09T20:54:41.595+09:00  INFO 44856 --- [           main] c.e.d.DemoSpringProjectApplication       : Started DemoSpringProjectApplication in 1.274 seconds (process running for 1.671)
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


