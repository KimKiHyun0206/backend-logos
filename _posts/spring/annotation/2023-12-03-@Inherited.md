---
title: Inherited Annotation
date: 2023-12-03 20:20:00 +0800
categories: [Spring, Annotation]
tags: [spring, code, annotation]
render_with_liquid: false
---

# Inherited

> @Inherited 자식 클래스가 부모에 선언된 애노테이션을 같이 사용하고 싶을 때 선언한다

이 애노테이션의 주석을 해석하면 다음과 같다

```markdown
Indicates that an annotation interface is automatically inherited.  
**If an Inherited meta-annotation is present on an annotation interface
declaration, and the user queries the annotation interface on a class
declaration, and the class declaration has no annotation for this interface,
then the class's superclass will automatically be queried for the
annotation interface.**  This process will be repeated until an annotation for
this interface is found, or the top of the class hierarchy (Object)
is reached. If no superclass has an annotation for this interface, then
the query will indicate that the class in question has no such annotation.

Note that this meta-annotation interface has no effect if the annotated
interface is used to annotate anything other than a class. Note also
that this meta-annotation only causes annotations to be inherited
from superclasses; annotations on implemented interfaces have no effect.

@author Joshua Bloch
@since 1.5
@jls 9.6.4.3 @Inherited

주석 인터페이스가 자동으로 상속됨을 나타냅니다.
주석 인터페이스에 상속된 메타 주석이 있는 경우
선언 및 사용자는 클래스의 주석 인터페이스를 쿼리합니다.
선언이 있고 클래스 선언에는 이 인터페이스에 대한 주석이 없습니다.
그러면 클래스의 슈퍼클래스가 자동으로 쿼리됩니다.
주석 인터페이스. 이 프로세스는 주석이 추가될 때까지 반복됩니다.
이 인터페이스가 발견되었거나 클래스 계층 구조(객체)의 최상위에 있습니다.
도달했습니다. 이 인터페이스에 대한 주석이 있는 슈퍼클래스가 없는 경우
쿼리는 문제의 클래스에 그러한 주석이 없음을 나타냅니다.

이 메타 주석 인터페이스는 주석이 달린 경우에는 효과가 없습니다.
인터페이스는 클래스 이외의 항목에 주석을 다는 데 사용됩니다. 참고하세요
이 메타 주석으로 인해 주석만 상속됩니다.
슈퍼클래스에서; 구현된 인터페이스의 주석은 효과가 없습니다.
```

즉 자동으로 상속 받는다는 의미이다.

```java

@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface MyInherited {
  String value() default "hi";
}
```

위 인터페이스를 구현하고

```java

@MyInherited
public class ClassA {
}

public class ClassB extends ClassA {
}
```

두 ClassA에 애노테이션을 붙인 다음에 ClassB에 상속시킨다.

```java
public class DemoSpringApplication {

  public static void main(String[] args) {
    Annotation[] str = new ClassB().getClass().getAnnotations();

    for (Annotation an : str) {
      System.out.println(an);
    }
  }
}
```

그리고 위 코드를 통해 애노테이션이 @Inherited를 통해 제대로 상속되었는지 보기 위해 출력을 해보면.
> @com.example.demospring.MyInherited("hi")

이처럼 로그가 나온다.
