---
title: SpringBootApplication
date: 2023-12-01 20:10:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
  @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)})
public @interface SpringBootApplication {

  @AliasFor(annotation = EnableAutoConfiguration.class)
  Class<?>[] exclude() default {};

  @AliasFor(annotation = EnableAutoConfiguration.class)
  String[] excludeName() default {};

  @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
  String[] scanBasePackages() default {};
  
  @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
  Class<?>[] scanBasePackageClasses() default {};

  @AliasFor(annotation = ComponentScan.class, attribute = "nameGenerator")
  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;
  
  @AliasFor(annotation = Configuration.class)
  boolean proxyBeanMethods() default true;

}
```

<br>

## 메타 애노테이션

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
  @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)})
```

| 애노테이션                    | 설명                                                |
|:-------------------------|:--------------------------------------------------|
| @Target                  | 타입에만 붙일 수 있는 애노테이션이다                              |
| @Retention               | 런타임 동안에는 계속 참조 가능하다                               |
| @Inherited               | 자식 클래스에게 애노테이션이 상속되도록 한다                          |
| @SpringBootConfiguration | 환경 설정 빈 구성을 자동으로 찾을 수 있게 해준다                      |
| @EnableAutoConfiguration | @CompnentScan으로 빈이 등록된 이후, 추가적인 빈들을 읽어 등록하는 애노테이션 |
| @ComponentScan           | 스프링 빈으로 등록될 것임을 알려준다                              |
| @Filter                  | 스프링 빈으로 등록할 것과 등록하지 않을 것을 구분해준다                   |


<br>

# 메소드
* exclude : 특정 클래스를 자동 설정에서 제외함
* excludeName : 클래스의 이름으로 자동 설정에서 제외함
* scanBasePackages : 컴포넌트 스캔(빈 탐색)을 진행할 베이스 패키지 설정
* scanBaseClasses : 컴포넌트 스캔을 진행할 베이스 클래스를 설정
* nameGenerator : 빈 이름 생성을 담당할 클래스를 설정함
* proxyBeanMethods : @Bean 메소드를 프록시 방식으로 처리하도록 설정한다

## proxyBeanMethods
```java
@Configuration(proxyBeanMethods = false)
public class Config {

  @Bean
  public SampleBean sampleBean() {
    return new SampleBean();
  }

  @Bean
  public BeanA beanA() {
    return new BeanA(sampleBean());
  }

  @Bean
  public BeanB beanB() {
    return new BeanB(sampleBean());
  }
}
```
위의 코드로 빈을 생성하면 원래는 하나만 생성되어야 하는 `SampleBean`이 두 개가 만들어진다.
그 이유는 `proxyBeanMethods = false`로 설정되었기 때문이다. 이는 Config 클래스에 프록시가 적용되지 않았기 때문이다.
하지만 이 상황은 보통 개발자들이 원하는 상황이 아니기 때문에 스프링은 기본값으로 `proxyBeanMethods = true`를 기본값으로 가지고 있으며 바이트 조작 라이브러리인 CGLib을 통해 프록시 패턴을 적용한다.
그렇게 해당 `@Bean` 메소드가 호풀되어 이미 빈이 생성되었으면 존재하는 빈을 찾아서 반환하여 싱글톤을 유지하도록 하는 것이다.

