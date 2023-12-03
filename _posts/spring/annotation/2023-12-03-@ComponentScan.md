---
title: ComponentScan Annotation
date: 2023-12-03 20:10:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---

> @ComponentScan은 **빈을 등록하기 위한 애노테이션들을 탐색하는 위치를 지정**한다

```java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {

  @AliasFor("basePackages")
  String[] value() default {};

  @AliasFor("value")
  String[] basePackages() default {};

  Class<?>[] basePackageClasses() default {};

  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

  Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;

  ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;

  String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;

  boolean useDefaultFilters() default true;

  Filter[] includeFilters() default {};

  Filter[] excludeFilters() default {};

  boolean lazyInit() default false;

  @Retention(RetentionPolicy.RUNTIME)
  @Target({})
  @interface Filter {

    FilterType type() default FilterType.ANNOTATION;

    @AliasFor("classes")
    Class<?>[] value() default {};

    @AliasFor("value")
    Class<?>[] classes() default {};

    String[] pattern() default {};
  }
}
```

* basePackagesClasses나 basePackages를 통해 베이스 패키지를 설정할 수 있지만 설정하지 않으면 해당 애노테이션이 붙은 클래스를 기준으로 진행한다
* @ComponentScan은 @SpringBootApplication 안에 포함되어있으므로 이러한 이유로 스프링 부트의 메인 클래스는 루트 패키지에 두는 것이 좋다
  * 그렇지 않으면 따로 베이스 패키지를 설정해주어야 하니까

<br>

스캔이 진행되면 @Configuration과 @Bean 및 @Component의 하위 애노테이션들이 있는 클래스 및 메소드를 찾는다.
@SpringBootApplication에는 includeFilter와 excludeFilter가 있는데 각각 다음 기능을 한다

* includeFilter : 해당 클래스들을 컴포넌트 스캔 대상에 포함한다
* excludeFilter : 해당 클래스들을 컴포넌트 스캔 대상에서 제외한다

스프링 부트에서는 TypesExcludeFilter와 AutoConfigurationExcludeFilter를 컴포넌트 스캔 대상에서 제외하는데,
TypeExcludeFilter와 AutoConfigurationFilter는 거의 spring-boot-test-에서 내부적으로 사용되므로 스캔에서 제외된다.
