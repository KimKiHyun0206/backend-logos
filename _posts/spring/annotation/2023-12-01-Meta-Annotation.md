---
title: 메타 애노테이션
date: 2023-12-01 20:10:00 +0800
categories: [Spring, Annotation]
tags: [java, annotation]
render_with_liquid: false
---

Spring에는 Annotation에 대한 기능을 다양하게 제공한다.

예를 들어 @Controller, @Service, @Repository 등이 있다.

Annotation은 각 기능에 필요한 만큼 기능을 가지고 있으며, 이러한 내용을 잘 알지 못해도 필요한 기능만 쉽게 사용할 수 있도록 제공된다.

하지만 개발을 하다 보면 이를 만드는 프로그램에 맞게 커스텀할 필요가 있다.

이때 필요한 다양한 Meta Annotation에 대해 알아볼 것이다.

<br>

## Meta-Annotation

> 다른 Annntation에서도 사용되는 Annotation을 뜻한다

* 주로 Custom-Annotation을 생성할 때 주로 사용된다

예를 들어 @Service의 코드는 아래와 같다

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {
  //세부 내용은 생략
}
```

1. 서비스는 @Component를 가지고 있기 때문에 @Component의 기능을 수행한다
2. @Target이 ElementType.Type이기 때문에 이는 타입 선언시 사용한다는 의미임을 알 수 있다
3. @Retention이 RetentionPolicy.RUNTIME이기 때문에 이는 컴파일 이후에도 JVM에 의해서 계속 참조가 가능함을 의미한다

<br>

## @Target

> 자바 컴파일러가 이 Annotation을 어디에 적용할지 결정하기 위해 사용한다

```java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
  /**
   * Returns an array of the kinds of elements an annotation type
   * can be applied to.
   * @return an array of the kinds of elements an annotation type
   * can be applied to
   */
  ElementType[] value();
}

```

| 종류                          | 의미               |
|:----------------------------|:-----------------|
| ElementType.PACKAGE         | 패키지              |
| ElementType.TYPE            | 타입               |
| ElementType.ANNOTATION_TYPE | 어노테이션 타입         |
| ElementType.CONSTRUCTOR     | 생성자              |
| ElementType.FIELD           | 멘버 변수            |
| ElementType.LOCAL_VARIABLE  | 지역 변수            |
| ElementType.METHOD          | 메소드              |
| ElementType.PARAMETER       | 전달인자(파라미터)       |
| ElementType.TYPE_PARAMETER  | 전달인자 타입(파라미터 타입) |
| ElementType.TYPE_USE        | 타입               |

* 여러 개를 선택할 수 있다

<br>

## @Retention

> Annotation이 실제로 적용되고 유지되는 범위를 의미한다

Annotation Tpye을 어디까지 보유할지 결정한다

Policy에 관련된 Annotation으로 컴파일하고나서 언제까지 유호한지 구분해준다

```java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
  /**
   * Returns the retention policy.
   * @return the retention policy
   */
  RetentionPolicy value();
}
```

| 종류                      | 설명                                                        |
|:------------------------|:----------------------------------------------------------|
| RetentionPolicy.RUNTIME | 컴파일 이후에도 JVM에 의해서 계속 참조가 가능하다.<br/>주로 리플렉션이나 로깅에 많이 사용된다. |
| RetentionPolicy.CLASS   | 컴파일러가 클래스를 참조할 때까지 유효하다                                   |
| RetentionPolicy.SOURCE  | 컴파일 전까지만 유효하다.<br/>즉, 컴파일 이후에는 사라진다                       |

* Default는 CLASS이다
* 셋 중 하나만 선택할 수 있다
