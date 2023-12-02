---
title: EnableAutoConfiguration
date: 2023-12-01 20:00:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---




> 우리가 필요할 것 같은 빈들을 자동으로 설정해주는 자동 설정 기능을 활성화하는 애노테이션이다

자동 설정을 적용할 클래스 선별은 AutoConfigurationImportSelector가 처리해준다.
이때 특정 클래스를 설정으로 추가하는  @Import를 통해 해당 클래스를 추가해준다.
즉, @EnableAutoConfiguration을 붙이면 AutoConfigurationImportSelector까지 추가된다
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY= "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```
AutoConfigurationImportSelector는 자동 설정할 대상들을 결정하는데, 자동 설정이 가능한 목록은 spring-boot-autoconfigure.jar의 META0INF 디렉토리의 spring.factories에서 확인할 수 있다.
해달 피일을 보면 다양한 자동 설정을 스프링이 지원하고 있음을 알 수 있다.

<br>

AutoConfigurationImportSelector는 다음 메소드를 통해 자동 설정이 가능한 후보군들을 모두 불러온다
```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. 
        If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```
