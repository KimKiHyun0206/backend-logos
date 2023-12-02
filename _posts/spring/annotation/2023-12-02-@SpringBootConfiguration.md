---
title: SpringBootConfiguration
date: 2023-12-02 20:00:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---



> @Configuration의 하위 애노테이션으로 @Configuration과 동일한 역할을 수행한다

실제로 두 코드 사이에 별다른 차이가 없다

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {

  @AliasFor(annotation = Configuration.class)
  boolean proxyBeanMethods() default true;

}
```

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

  @AliasFor(annotation = Component.class)
  String value() default "";

  boolean proxyBeanMethods() default true;

}
```

* **@SpringBootConfiguration 애노테이션은 애플리케이션에 하나만 존재해야 한다**
* 일반적으로 @SpringBootApplication 안에 포함되어 있으므로 따로 작성할 필요는 없다
* 이 애노테이션이 존재하는 이유는 해당 **애노테이션을 기준으로 설정들을 불러오기 위함**이다
  * 대표적인 예시로 @SpringBootTest는 테스트를 위해 @SpringBootConfiguration을 가진 클래스를 찾는다
  * 그리고 찾은 클래스를 중심으로 하위의 설정들을 자동으로 찾는다
  * 일반적으로 @SpringBootConfiguration은 @SpringBootApplication 안에 포함되어있으므로 메인 클래스가 찾아진다
