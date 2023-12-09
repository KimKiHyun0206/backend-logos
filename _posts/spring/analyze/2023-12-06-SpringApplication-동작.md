---
title: SpringApplication 동작
date: 2023-12-06 20:00:00 +0800
categories: [Spring, Analyze]
tags: [spring, code, analyze]
render_with_liquid: false
---

Toby-Spring 강의를 듣고 나서 SpringApplication이 어떻게 동작하는지 한 번 정리해자.
먼저 전체적인 코드는 위와 같다.

```java
public class TobySpringApplication {
  public static void main(String[] args) {
    GenericWebApplicationContext applicationContext = new GenericWebApplicationContext();
    applicationContext.registerBean(HelloController.class);
    applicationContext.registerBean(SimpleHelloService.class);
    applicationContext.refresh();


    ServletWebServerFactory serverFactory = new TomcatServletWebServerFactory();
    WebServer webServer = serverFactory.getWebServer(servletContext -> {
      servletContext.addServlet("dispatcherServlet",
        new DispatcherServlet(applicationContext)
      ).addMapping("/*");
    });
    webServer.start();
  }

}
```

이제 코드를 하나씩 보면서 어떻게 동작하는지 보자.

```java
  GenericWebApplicationContext applicationContext=new GenericWebApplicationContext();
  applicationContext.registerBean(HelloController.class);
  applicationContext.registerBean(SimpleHelloService.class);
  applicationContext.refresh();
```

* `GenericWebApplicationContext` : dispatcherServlet을 등록하기 위해서 만든 ServletContainer이다
* `registerBean()` : 인자로 받는 클래스를 Bean으로 등록할 것이라고 알려주는 것이다
* `refresh()` : 여태까지 register한 클래스들을 Bean으로 등록하는 것이다
* 실제로 등록되는 순간은 `refresh()`하는 순간이다.

이 과정들은 Web에 서비스를 올리기 전에 먼저 필요한 빈들을 등록하는 과정이다

<br>

```java
  ServletWebServerFactory serverFactory=new TomcatServletWebServerFactory();
  WebServer webServer=serverFactory.getWebServer(servletContext->{
  servletContext.addServlet("dispatcherServlet",
  new DispatcherServlet(applicationContext)
  ).addMapping("/*");
  });
  webServer.start();
```

* `ServletWebServerFactory` : 실제 웹 서버를 띄우기 위한 클래스이다
  * 위에서는 `TomcatServer`를 사용하기 위해서 `TomcatServletWebServerFactory`를 사용했다
* `WebServer` : serverFactory에서 서버를 받아와서 서버를 만든다
  * `dispatcherServlet`에게 작업을 위임한다
  * 요청을 디스패치할 오브젝트를 찾아야하는데 그때 찾아야할 `ServletContainer`를 전달해준다
* `addMapping` : 기본적인 주소를 무엇으로 할지 결정해준다
  * '*'은 all이라는 의미이다

일단 대략적인 SpringApplication의 시작 흐름은 이렇다. 그럼이제 실제 코드를 보면서 내부에서 어떻게 동작하는지 보자.

<br>

제일 기본적인 SpringApplication 코드는 이렇다

```java

@SpringBootApplication
public class MySpringApplication {
  public static void main(String[] args) {
    SpringApplication.run(MySpringApplication.class, args);
  }
}
```
