---
title: EnableAutoConfiguration
date: 2023-12-02 20:00:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---

> 우리가 필요할 것 같은 빈들을 자동으로 설정해주는 자동 설정 기능을 활성화하는 애노테이션이다

자동 설정을 적용할 클래스 선별은 `AutoConfigurationImportSelector`가 처리해준다.
이때 특정 클래스를 설정으로 추가하는 `@Import`를 통해 해당 클래스를 추가해준다.
즉, `@EnableAutoConfiguration`을 붙이면 `AutoConfigurationImportSelector`까지 추가된다

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

  String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

  Class<?>[] exclude() default {};

  String[] excludeName() default {};
}
```

`AutoConfigurationImportSelector`는 자동 설정할 대상들을 결정하는데, 자동 설정이 가능한 목록은 `spring-boot-autoconfigure.jar`의 META-INF 디렉토리의
`spring.factories`에서 확인할 수 있다.
해달 파일을 보면 다양한 자동 설정을 스프링이 지원하고 있음을 알 수 있다.

<br>

`AutoConfigurationImportSelector`는 다음 메소드를 통해 자동 설정이 가능한 후보군들을 모두 불러온다

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,AnnotationAttributes attributes){
  List<String> configurations=SpringFactoriesLoader.loadFactoryNames(
  getSpringFactoriesLoaderFactoryClass(),getBeanClassLoader());
  Assert.notEmpty(configurations,"No auto configuration classes found in META-INF/spring.factories. 
  If you are using a custom packaging,make sure that file is correct.");
  return configurations;
  }
```

`SpringFactoriesLoader.loadFactoryNames`로 가보면 클래스 로더를 사용해서 `spring.factories`에서 값을 불러옴을 확인할 수 있다

```java
public final class SpringFactoriesLoader {

  public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";


  private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);

  static final Map<ClassLoader, Map<String, List<String>>> cache = new ConcurrentReferenceHashMap<>();


  private SpringFactoriesLoader() {
  }

  public static <T> List<T> loadFactories(Class<T> factoryType, @Nullable ClassLoader classLoader) {
    Assert.notNull(factoryType, "'factoryType' must not be null");
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
      classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    List<String> factoryImplementationNames = loadFactoryNames(factoryType, classLoaderToUse);
    if (logger.isTraceEnabled()) {
      logger.trace("Loaded [" + factoryType.getName() + "] names: " + factoryImplementationNames);
    }
    List<T> result = new ArrayList<>(factoryImplementationNames.size());
    for (String factoryImplementationName : factoryImplementationNames) {
      result.add(instantiateFactory(factoryImplementationName, factoryType, classLoaderToUse));
    }
    AnnotationAwareOrderComparator.sort(result);
    return result;
  }


  public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader; //클래스 로더를 사용해서 값을 가져온다
    if (classLoaderToUse == null) {
      classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
  }

  private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
      return result;
    }

    result = new HashMap<>();
    try {
      Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
      while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        UrlResource resource = new UrlResource(url);
        Properties properties = PropertiesLoaderUtils.loadProperties(resource);
        for (Map.Entry<?, ?> entry : properties.entrySet()) {
          String factoryTypeName = ((String) entry.getKey()).trim();
          String[] factoryImplementationNames =
            StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
          for (String factoryImplementationName : factoryImplementationNames) {
            result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
              .add(factoryImplementationName.trim());
          }
        }
      }

      // Replace all lists with unmodifiable lists containing unique elements
      result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
        .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
      cache.put(classLoader, result);
    } catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
        FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
  }

  @SuppressWarnings("unchecked")
  private static <T> T instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader) {
    try {
      Class<?> factoryImplementationClass = ClassUtils.forName(factoryImplementationName, classLoader);
      if (!factoryType.isAssignableFrom(factoryImplementationClass)) {
        throw new IllegalArgumentException(
          "Class [" + factoryImplementationName + "] is not assignable to factory type [" + factoryType.getName() + "]");
      }
      return (T) ReflectionUtils.accessibleConstructor(factoryImplementationClass).newInstance();
    } catch (Throwable ex) {
      throw new IllegalArgumentException(
        "Unable to instantiate factory class [" + factoryImplementationName + "] for factory type [" + factoryType.getName() + "]",
        ex);
    }
  }

}
```

`loadSpringFactories` 메소드는 전체 `jar` 파일로부터 `spring.factories`들을 불러오는 기능으로, 스프링 애플리케이션 시작 시에 **매우 자주 호출**된다.
문제는 해당 호출이 디스크에서 값을 읽어오므로 처리 속도가 **상당히 느리다**는 것인데 이 때문에 **스프링은 캐싱 방식을 적용**하고 있다.
`loadSpringFactories`메소드의 처음 로직을 보면 캐시에서 값을 꺼내고 없으면 디스크에서 읽어오도록 처리하는 부분이 있다.

```java
public final class SpringFactoriesLoader{
  private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    Map<String, List<String>> result = cache.get(classLoader);  //이 부분에서 캐시를 사용한다
    if (result != null) {   //읽어온 값이 있다면 그대로 리턴한다
      return result;
    }
    
    //읽어온 값이 없다면 값을 디스크에서 읽어오는 작업을 한다
    result = new HashMap<>();
    try {
      Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);    //META-INF/spring.factories 에서 값을 읽어올 것임
      while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        UrlResource resource = new UrlResource(url);
        Properties properties = PropertiesLoaderUtils.loadProperties(resource);
        for (Map.Entry<?, ?> entry : properties.entrySet()) {
          String factoryTypeName = ((String) entry.getKey()).trim();
          String[] factoryImplementationNames =
            StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
          for (String factoryImplementationName : factoryImplementationNames) {
            result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
              .add(factoryImplementationName.trim());
          }
        }
      }

      // Replace all lists with unmodifiable lists containing unique elements
      result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
        .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
      cache.put(classLoader, result);
    } catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
        FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
  }
}
```

자동 설정을 위한 `auto-configiration` 클래스들은 `@Configuration`이 있는 설정 클래스들인데,
어떤 클래스의 존재 유무를 판단하기 위해 `@Conditional` 애노테이션이 사용된다.
많은 조건 애노테이션들 중에서 다음이 가장 많이 사용된다.

* `@ConditionalOnClass` : 해당 클래스가 클래스 패스에 존재하는 경우
* `@ConditionalOnBean` : 해당 클래스나 이름이 빈 팩토리에 포함되어 있는 경우
* `@ConditionalOnMissingBean` : 해당 빈이 등록되어있지 않을 경우
* 기타...

> ConditionalOnMissingBean이 자주 사용되는 이유는 우리가 직접 설정한 빈과 자동 설정으로
> 등록되는 빈이 중복될 경우 우리가 직접 설정한 빈에 우선순위를 부여하기 위함이다.

### @EnableAutoConfiguration이 붙어있는 클래스는 특별한 의미를 갖는다

> 특정 작업들을 위한 베이스 패키지가 되기 때문이다

대표적으로 JPA의 `@Entity` 클래스를 탑색하는 작업은 `@EnableAutoConfiguration`이 붙어있는 클래스의 패키지를 기준으로 진행된다.
일반적으로 `@EnableAutoConfiguration`은 `@SpringBootApplication` 안에 포함되어있으므로 자동 설정이 활성화 되지만,
해당 애노테이션을 직접 붙여주는 경우에는 위와 같은 이유(빈에 우선순위 부여)로 루트 패키지에 있는 클래스에 붙여주는 것이 좋다.
