---  
emoji: 📝  
title: '생명주기가 다른 빈 주입하기(Scoped Proxy Bean)'   
date: '2022-07-13 23:00:00'  
author: 어썸오  
tags: spring
categories: projects
---  

## 문제 상황

프로젝트 도중 singleton scope 빈에 request scope을 주입해서 사용해야 할 상황이 생겼다.

아래와 같이 인가 처리를 위한 커스텀 인터셉터에서 인증 정보를 담아두는 객체를 주입받아 사용해야 했기 때문이다.

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {

    private final AuthenticationContext authenticationContext;

    ...

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        String token = extractToken(header);
        String subject = jwtTokenProvider.extractSubject(token);
        authenticationContext.setPrincipal(subject);

        return true;
    }
}
```

```java
    @Component
    @RequestScope
    public class AuthenticationContext {

        private String principal;

        public String getPrincipal() {
            return principal;
        }

        public void setPrincipal(String principal) {
            this.principal = principal;
        }
    }
```

**AuthenticationContext**는 **principal**이라는 상태를 가지기 때문에 싱글톤으로 사용할 경우 멀티 쓰레드 환경에서 버그를 일으킬 수 있다. 하나의 요청에서는 같은 인증 정보를 사용하기 때문에 빈의 스코프를 request로 제한하여 사용하면 된다.

애플리케이션은 문제없이 잘 작동했다. 그런데 다음과 같은 의문이 들었다. **커스텀 인터셉터는 싱글톤 스코프니까 애플리케이션에서 단 한 번만 초기화될 텐데, 어째서 AuthenticationContext는 매번 새로 생성될까?**

실제로 로그를 찍어보면 **AuthenticationInterceptor**의 참조값은 변하지 않지만, **AuthenticationContext**의 참조값은 매 요청마다 변한다.

## Bean Scope

일반적으로 싱글톤 스코프 빈에 범위가 더 좁은 스코프를 가진 빈을 주입하면 주입받은 빈은 단 한 번밖에 초기화되지 않는다.

아래처럼 빈 스코프를 프로토타입으로 설정하면 컨테이너에 빈을 요청할 때마다 새로운 객체를 생성해서 반환한다.

```java
class PrototypeTest {

    @Test
    void prototypeBean() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        PrototypeBean prototypeBean1 = ac.getBean(PrototypeBean.class);
        PrototypeBean prototypeBean2 = ac.getBean(PrototypeBean.class);

        assertThat(prototypeBean1).isNotSameAs(prototypeBean2);
    }

    @Configuration
    static class Config {

        @Bean
        @Scope("prototype")
        public PrototypeBean prototypeBean() {
            return new PrototypeBean();
        }
    }

    static class PrototypeBean {

    }
}
```

<br>

하지만 프로토타입 빈을 싱글톤 빈에 주입한다면, 프로토타입 빈 또한 한 번만 생성된다.

```java
class PrototypeTest {

    @Test
    void prototypeBeanInSingletonBean() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        SingletonBean singletonBean1 = ac.getBean(SingletonBean.class);
        SingletonBean singletonBean2 = ac.getBean(SingletonBean.class);

        assertThat(singletonBean1.getPrototypeBean())
                .isSameAs(singletonBean2.getPrototypeBean());
    }

    @Configuration
    static class Config {

        @Bean
        @Scope(value = "prototype")
        public PrototypeBean prototypeBean() {
            return new PrototypeBean();
        }

        @Bean
        public SingletonBean singletonBean() {
            return new SingletonBean(prototypeBean());
        }
    }

    static class PrototypeBean {

    }

    static class SingletonBean {
        private PrototypeBean prototypeBean;

        public SingletonBean(PrototypeBean prototypeBean) {
            this.prototypeBean = prototypeBean;
        }

        public PrototypeBean getPrototypeBean() {
            return prototypeBean;
        }
    }
}
```

## 생명 주기 보장하기

하지만 싱글톤 빈 안에 더 좁은 스코프의 빈을 주입할 때 해당 빈을 새로 생성해서 사용하고 싶은 경우가 있다.

이때는 두 가지 방법이 있는데 조금 차이가 있다.

-   ObjectProvider
    -   getObject를 호출할 때 객체를 생성한다.
-   Scoped Proxy Bean
    -   프록시를 넣어놨다가 메서드 호출 시점에 실제 객체를 생성하고 요청을 위임한다
    -   @Scope 애너테이션의 proxyMode를 TARGET\_CLASS로 설정하면 된다.


### ObjectProvider

주입할 빈을 직접 사용하지 않고 **ObjectProvider**를 주입한다. 해당 빈을 사용할 때는 getObject()를 호출해서 빈을 가져와서 사용한다. getObject를 호출할 때까지 빈 생성을 지연할 수 있다.

```java
class PrototypeTest {

    @Test
    void prototypeBeanInSingletonBean() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        SingletonBean singletonBean1 = ac.getBean(SingletonBean.class);
        SingletonBean singletonBean2 = ac.getBean(SingletonBean.class);

        assertThat(singletonBean1.getPrototypeBean())
                .isNotSameAs(singletonBean2.getPrototypeBean());
    }

    @Configuration
    static class Config {

        @Bean
        @Scope(value = "prototype")
        public PrototypeBean prototypeBean() {
            return new PrototypeBean();
        }

        @Bean
        public SingletonBean singletonBean() {
            return new SingletonBean();
        }
    }

    static class PrototypeBean {

    }

    static class SingletonBean {

        @Autowired
        private ObjectProvider<PrototypeBean> prototypeBeanObjectProvider;

        public PrototypeBean getPrototypeBean() {
            return prototypeBeanObjectProvider.getObject();
        }
    }
}
```

### Scoped Proxy Bean

다른 방법으로는 **proxyMode** 값을 설정해 Scoped Proxy Bean 사용하는 것이 있다.

주의할 점은 **프로토타입 스코프임에도 불구하고 컨테이너에서 여러 번 요청해도 같은 객체가 나온다**는 점이다. 그리고 이 객체는 실제 객체가 아닌 프록시이다.

```java
class PrototypeBeanProxyTest {

    @Test
    void prototypeBean() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        PrototypeBean prototypeBean1 = ac.getBean(PrototypeBean.class);
        PrototypeBean prototypeBean2 = ac.getBean(PrototypeBean.class);

        assertThat(prototypeBean1).isSameAs(prototypeBean2); // 두 객체가 같다
    }

    @Test
    void prototypeBeanInSingletonBean() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        SingletonBean singletonBean1 = ac.getBean(SingletonBean.class);
        SingletonBean singletonBean2 = ac.getBean(SingletonBean.class);

        assertThat(singletonBean1.getPrototypeBean())
                .isSameAs(singletonBean2.getPrototypeBean()); // 마찬가지
    }

    @Configuration
    static class Config {

        @Bean
        @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
        public PrototypeBean prototypeBean() {
            return new PrototypeBean();
        }

        @Bean
        public SingletonBean singletonBean() {
            return new SingletonBean(prototypeBean());
        }
    }

    static class PrototypeBean {

    }

    static class SingletonBean {

        private PrototypeBean prototypeBean;

        public SingletonBean(PrototypeBean prototypeBean) {
            this.prototypeBean = prototypeBean;
        }

        public PrototypeBean getPrototypeBean() {
            return prototypeBean;
        }
    }
}
```

<br> 

**proxyMode를 TARGET\_CLASS로 설정하면 메서드를 호출하는 시점에 실제 객체를 생성한다.** 그래서 로그를 찍어보면, 메서드를 호출하기 전과 호출할 때의 객체의 주소 값이 다르다.

```java
class PrototypeBeanProxyTest {

    @Test
    void test() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        SingletonBean singletonBean = ac.getBean(SingletonBean.class);

        //increaseAndGetCount를 호출할 때 프로토타입 빈의 주소를 콘솔에 출력하도록 설정
        assertThat(singletonBean.increaseAndGetCount()).isEqualTo(1);

        //In Singleton = PrototypeBeanProxyTest$PrototypeBean@53499d85
        //this = PrototypeBeanProxyTest$PrototypeBean@1133ec6e
    }

    @Configuration
    static class Config {

        @Bean
        @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
        public PrototypeBean prototypeBean() {
            return new PrototypeBean();
        }

        @Bean
        public SingletonBean singletonBean() {
            return new SingletonBean(prototypeBean());
        }
    }

    static class PrototypeBean {

        private int count = 0;

        public int increaseAndGetCount() {
            System.out.println("this = " + this);
            count++;
            return count;
        }
    }

    static class SingletonBean {

        private PrototypeBean prototypeBean;

        public SingletonBean(PrototypeBean prototypeBean) {
            this.prototypeBean = prototypeBean;
        }

        public int increaseAndGetCount() {
            System.out.println("In Singleton = " + prototypeBean);
            return prototypeBean.increaseAndGetCount();
        }

        public PrototypeBean getPrototypeBean() {
            return prototypeBean;
        }
    }
}
```

## @RequestScope

프로젝트에서 **@RequestScope**를 사용하면 아무 문제가 없었던 이유는 **@RequestScope가 @Scope(value = “request”, proxyMode = ScopedProxyMode.TARGET\_CLASS)의 숏컷**이기 때문이다.

또한 Request 스코프의 경우 빈의 생명 주기가 웹 요청과 같기 때문에 Application을 처음 구동할 때는 생성되지 않는다. 만약 Request 스코프의 빈을 다른 생명 주기의 빈에 주입해서 사용할 때는 반드시 프록시 모드로 사용해야 한다.

```java
package org.springframework.web.context.annotation;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_REQUEST)
public @interface RequestScope {

    /**
     * Alias for {@link Scope#proxyMode}.
     * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @AliasFor(annotation = Scope.class)
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

<br> 

가장 처음 의문으로 돌아가보자.

> 커스텀 인터셉터는 싱글톤 스코프니까 애플리케이션에서 단 한 번만 초기화될 텐데, 어째서 AuthenticationContext는 매번 새로 생성될까?

이에 대한 답은 다음과 같다. **AuthenticationInterceptor**가 초기화될 때, **AuthenticationContext**의 프록시 객체가 주입된다, AuthenticationContext의 메서드가 호출될 때마다 실제 빈을 생성하고 요청을 위임한다.

## 참고자료

[Spring.io: beans-factory-scopes-other-injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-scopes-other-injection)

```toc
```