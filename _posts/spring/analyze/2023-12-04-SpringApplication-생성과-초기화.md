---
title: SpringApplication 생성과 초기화
date: 2023-12-04 20:00:00 +0800
categories: [Spring, Analyze]
tags: [spring, code, analyze]
render_with_liquid: false
---

### SpringBootApplication.run

실제 SpringApplication 클래스의 코드는 너무 많으니 보고 싶은 run 메소드만 보자.

```java
public class SpringApplication {

  public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[]{primarySource}, args);
  }

  public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
  }

  public ConfigurableApplicationContext run(String... args) {
    //실제 로직 
  }
}
```

`SpringApplication`의 `run` 메소드는 `ConfigurableApplicationContext`를 반환하는데, `SpringApplication` 클래스가 어떠한 부모 클래스나 인터페이스를 가지지
않으므로,
**run 내부에서 ApplicationContext를 만들어 실행하고 반환함을 알 수 있다.**

<br>

## SpringBootApplication 초기화 및 실행

`SpringApplication` 객체를 생성하는 생성자 코드는 다음과 같다. 여기서 `primarySources`는 애플리케이션의 메인 클래스이며
전달된 메인 클래스가 `null`이면 에러를 반환하도록 되어있다.

```java
public class SpringApplication {
  public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
  }

  public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = getBootstrapRegistryInitializersFromSpringFactories();
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
  }
}
```

메인 클래스가 `null`인지 검사한 후에 5 단계가 있다

1. 클래스 패스로부터 애플리케이션 타입을 추론한다
2. BootstrapRegistryInitializer를 불러오고 셋해준다
3. ApplicationContextInitializer를 찾아서 셋해준다
4. ApplicationListener를 찾아서 셋해준다
5. 메인 클래스를 추론한다

### 클래스 패스로부터 애플리케이션 타입을 추론한다

> SpringBoot 애플리케이션 실행 극 초반 단계에 현재 애플리케이션 타입이 무엇인지 판별한다.

Spring 5.0 부터는 3가지가 존재한다

1. AnnotationConfigApplicationContext : 웹이 아닌 애플리케이션
2. AnnotationConfigServletWebServerApplication : 서블릿 기반의 웹 애플리케이션
3. AnnotationConfigReactiveWebServerApplicationContext : 리액티브 웹 애플리케이션

- 이중 이랙티브 웹 애플리케이션은 스프링 5.0 부터 추가된 것이다

그리고 앱 타입을 판단하는 기준은 **클래스 로더를 통해 클래스 패스에 해당하는 클래스가 존재하는지를 기준**으로 한다

```java
public class WebApplicationType {
  static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
        return WebApplicationType.NONE;
      }
    }
    return WebApplicationType.SERVLET;
  }
}
```

위 코드를 아래의 SpringApplication에서 사용한다.

```java
public class SpringApplication {
  public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
      getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
  }
}
```

이렇게 초반에 결정된 애플리케이션 타입은 이후에 **`ApplicationContext`나 `Environment`의 구현체를 만드는데 사용**된다.
위의 코드를 보면 한가지를 알 수 있는데, 만약 web(servlet)과 webflux 의존성이 모두 존재하는 경우하면 web(servlet) 애플리케이션으로 만들어진다는 것이다.
또한 이 애플리케이션 타입은 `AutoConfig`를 위한 `Enum`으로 매핑되어 `AutoConfig`의 컨디셔널 조건으로도 활용된다.
대표적으로 서블릿 웹 애플리케이션을 위한 `AutoConfig`클래스인 `WebMvcAutoConfiguration`에는 다음과 같이 `ConditionalOnWebApplication`이 Servlet일 경우로
설정되어있다.

```java

@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
  ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {
  //자세한 코드는 생략함
}
```

<br>

### BootstrapRegistryInitializer를 불러오고 셋해줌

> BootstrapRegistry(BootstrapContext)는 실제 구동되는 애플리케이션 컨텍스트가 준비되고 환경 변수들을 관리하는 스프링의 Environment 객체가 후처리되는 동안에 이용되는 **임시
컨텍스트 객체**이다.

애플리케이션이 준비되는 동안에는 BootstrapRegistry를 사용해야 하는데, 이 객체를 초기화하는데 사용되는 Initializer들은 SpringFactory에서 가져오고 객체로 만들어 셋하는 작업을 진행하는
것이다.

```java
//Spring 2.6.3
public class SpringApplication {
  private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
  }
}
```

```java
//Spring 3.2.0
public class SpringApplication {
  private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);
  }
}
```

객체로 만든 클래스를 SpringFactory에서 조회하는 부분은 loadFactoryNames 메소드이다.
SpringFactory는 jar 안에 META-INF/spring.factories에 존재하는 텍스트 파일로써 설정을 위해 필요한 클래스 정보들이 있다.

```
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer, \
org.springframework.boot.zutoconfigure.logging.ConditionalEvaluationReportLoggingListener
```

이 값은 Key-Value 형태인데, 이 형태는 jar 파일에 따라 `인터페이스 이름-구현체 목록` 또는 `설정 클래스 이름-필요한 클래스 목록` 등으로 구성된다.
구현체가 여러 개인 경우에는 콤마로 구현체들을 나열한다.
이러한 형태의 SpringFactory 정보들을 불러오는 loadFactoryNames 코드는 다음과 같다.

```java
public class SpringFactoriesLoader {
  private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 캐싱이 적용되어 있음
    MultiValueMap<String, String> result = (MultiValueMap) cache.get(classLoader);
    if (result != null) {
      return result;
    } else {
      try {
        //Get the spring.factories File path for
        Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
        LinkedMultiValueMap result = new LinkedMultiValueMap();

        while (urls.hasMoreElements()) {
          //Load one of them spring.factories file
          URL url = (URL) urls.nextElement();
          UrlResource resource = new UrlResource(url);
          Properties properties = PropertiesLoaderUtils.loadProperties(resource);
          //The spring.factories All key value pairs in the file
          Iterator var6 = properties.entrySet().iterator();

          while (var6.hasNext()) {
            //A collection of implementation classes for one of the interfaces
            Entry<?, ?> entry = (Entry) var6.next();
            List<String> factoryClassNames = Arrays.asList(StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
            result.addAll((String) entry.getKey(), factoryClassNames);
          }
        }

        cache.put(classLoader, result);
        return result;
      } catch (IOException var9) {
        throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var9);
      }
    }
  }
}
```

Spring은 다운 받은 모든 jar 파일 안에서 META-INF 하위에 `spring.factories`가 존재하는지 스캔한다. 그리고 존재한다면 해당 값을 모두 불러온다.
`spring.factories`가 존재하는 프로젝트로는 크게 `spring-boot`, `spring-boot-autoconfigure`, `spring-data-jpa`가 있다.
스캔한 값들은 이후에 `ApplicationContextInitializer`나 `ApplicationListener` 그리고 `AutoConfig`에서도 동일하게 사용되므로 **연속적인 File I/O로 인한 성능
문제를 방지하기 위해 캐싱을 적용**한다.
이 단계에서는 캐싱된 내역이 없어서 파일로부터 내용을 읽어왔지만 다음 단계에서부터는 이미 읽어와서 캐시에 저장해두었기 때문에 값을 불러오지 않는다.
그리고 불러온 클래스 목록들은 `createSpringFactoriesInstances`를 통해 객체로 생성되어 반환된다.

> #### SpringApplication 에서는 두 가지 부분에서 캐싱이 적용된다
> - BootstrapRegistryInitializer를 불러오고 셋 해주는 과정
>   - 연속적인 File I/O로 인한 성능 문제를 방지하기 위해 캐싱을 사용한다
> - @EnableAutoConfiguration 에서 `spring.factories`를 불러오는 작업
>   - 이때 디스크에서 값을 읽어오기 때문에 작업이 늦기 때문에 캐싱을 사용한다

<br>

### ApplicationContextInitializer를 찾아서 셋 해준다

> 실제로 사용되는 ApplicationContext를 위한 Initializer들을 로딩한다.

* 이 과정은 위의 BootstrapRegistryInitializer를 불러오는 과정과 거의 동일하다
* 하지만 BootstrapRegistryInitializer 과정과 두 가지 차이점이 있다
  * Initializer들을 로딩하는 과정에서 이미 Bootstrap 단계에서 spring.factories를 스캔했으므로 파일에서 값을 찾는 것이 아닌 캐싱된 값에서 값을 찾는다
  * spring.factories에서 읽은 값들의 key로 ApplicationContextInitializer에 해당하는 Value들만 조회해 객체로 가져온다
* SpringBoot는 해당 구현체들을 생성한 다음에 Initializer 값을 셋 해준다

```java
public class SpringApplication {

  public void setInitializers(Collection<? extends ApplicationContextInitializer<?>> initializers) {
    this.initializers = new ArrayList<>(initializers);
  }
}
```

<br>

### ApplicationListener를 찾아서 셋 해준다

> ApplicationListener들을 불러오고 listener 값을 셋해주는 부분이다.

이 부분은 ApplicationContextInitializer과 거의 동일하며 ApplicationListener 클래스를 Key 타입으로 준다는 것만 다르다.

* ApplicationListener는 옵저버 패턴을 기반으로 애플리케이션을 구독하는 리스너이다

<br>

### 메인 클래스를 추론한다

```java
//Spring 2.6.3
public class SpringApplication {
  private Class<?> deduceMainApplicationClass() {
    try {
      StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
      for (StackTraceElement stackTraceElement : stackTrace) {
        if ("main".equals(stackTraceElement.getMethodName())) {
          return Class.forName(stackTraceElement.getClassName());
        }
      }
    } catch (ClassNotFoundException ex) {
      // Swallow and continue
    }
    return null;
  }
}
```

```java
//Spring 3.2.0
public class SpringApplication {
  private Class<?> deduceMainApplicationClass() {
    return StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE)
      .walk(this::findMainClass)
      .orElse(null);
  }
}
```

이 단계에선 다음 순서를 거친다

1. 의도적으로 `RuntimeExcpetion`을 발생시킨다 -> 해당 StackTrace를 가져온다
2. 그리고 `main`이 붙어 있는 메소드를 찾으면 Main 클래스라고 판단하고 해당 클래스의 이름을 찾는다
3. 찾아낸 메인 애플리케이션 클래스는 이후에 `Logger`를 만들어 로그를 남기거나 리스너를 등록할 때 사용된다


