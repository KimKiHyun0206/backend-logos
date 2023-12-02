---
title: SpringBootApplication
date: 2023-12-02 20:10:00 +0800
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

| 애노테이션                    | 설명                                                                                 |
|:-------------------------|:-----------------------------------------------------------------------------------|
| @Target                  | 타입에만 붙일 수 있는 애노테이션이다                                                               |
| @Retention               | 런타임 동안에는 계속 참조 가능하다                                                                |
| @Inherited               | 자식 클래스에게 애노테이션이 상속되도록 한다                                                           |
| @SpringBootConfiguration | 환경 설정 빈 구성을 자동으로 찾을 수 있게 해준다                                                       |
| @EnableAutoConfiguration | @CompnentScan으로 빈이 등록된 이후, 추가적인 빈들을 읽어 등록하는 애노테이션                                  |
| @ComponentScan           | 스프링 빈으로 등록될 것임을 알려준다                                                               |
| @Filter                  | FilterType 인터페이스를 구현한 TypeExcludeFilter 클래스와 AutoConfigurationExcludeFilter를 사용한다. |

## 동작

@SpringBootApplication 애노테이션을 통해 설정한 값들은 각각 다음의 기능으로 연결되어 동작한다

```java
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

* `exclude` : 특정 클래스를 자동 설정에서 제외한다
* `excludeName` : 클래스의 이름으로 자동 설정에서 제외한다
* `scanBasePackages` : 컴토넌트 스캔을 진행할 베이스 패키지를 설정한다
* `scanBasePackageClass` : 컴포넌트 스캔을 진행할 베이스 클래스를 설정한다
* `nameGenerator` : 빈 이름 생성을 담당할 클래스를 설정한다
* `proxyBeanMethods` : @Bean 메소드를 프록시 방식으로 처리하도록 설정한다

### proxyBeanMethods

> @Bean으로 빈을 등록하는 메소드에 프록시 패턴을 적용할 것인지를 결정하는 속성이다

* 기본값은 true이며 별다른 설정을 하지 않았다면 @Bean 메소드에 프록시가 기본으로 적용된다
* @Bean 메소드에 프록시가 필요한 이유는 해당 메소드를 직접 호출하는 경우에도 항상 싱글톤 스코프를 강제하여 한 개의 객체만 생성하기 위함이다

코드를 예시로 들어보자

```java

@Configuration(proxyBeanMethods = false)
public class Config {
  @Bean
  public SampleBean sampleBean() {
    return new SampleBean();
  }

  @Bean
  public BeanOne beanOne() {
    return new BeanOne(sampleBean());
  }

  @Bean
  public BeanTwo beanTwo() {
    return new BeanTwo(sampleBean());
  }
}
```

위의 코드에서 SampleBean 객체를 생성하는 메소드를 직접 호출하고 있다.
만약 proxyBeanMethods를 false로 설정하면 Config클래스에 프록시가 적용되지 않아 총 2개의 SampleBean이 생성된다.
하지만 개발자들은 보통 객체가 하나만 생성되어 싱글톤으로 관리되어지기를 바란다.
스프링은 기본적으로 `proxyBeanMethods = true` 로 설정하고 바이트 조작 라이브러리인 CGLib을 통해 프록시 패턴을 적용한다.
그래서 @Bean 메소드가 호출되어 빈을 생성할 때 이전에 생성된 같은 타입의 빈이 있으면 찾아서 반환함으로써 싱글톤을 유지한다.

