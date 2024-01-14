---
layout: post
title: Spring - 스프링 컨테이너와 스프링 빈
tags: [Spring]
comments: true
---

## 스프링 컨테이너로 변환

우리는 지금까지 스프링을 1도 쓰지 않고 순수 자바코드를 썼다. 이를 스프링으로 변환해보자

```java
@Configuration
public class AppConfig {
    
    @Bean
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }
    
    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
    
    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }
    
    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }
}
```

먼저 Config 파일 맨위에 Configuration 이라는 어노테이션을 붙여주고, 각 메서드에 Bean 어노테이션을 붙여준다.

```java
ApplicationContext app = new AnnotationConfigApplicationContext(AppConfig.class); 
app.getBean("memberService", MemberService.class);
AppConfig appConfig = new AppConfig();
MemberService memberService = appConfig.memberService();
OrderService orderService = appConfig.orderService();
```

그리고 스프링 컨테이너를 생성하여 빈을 등록해준다. Application은 인터페이스로 AnnotationConfigApplicationContext는 구현체의 한 종류이다. 이 메서드의 인자로 AppConfig를 넘겨주어서 설정정보를 넣어준다.

그러면 Bean이 붙은 메서드를 모두 호출하여 해당 객체를 스프링 컨테이너에 등록한다. 객체가 생성됨과 동시에 의존관계도 형성된다. 

## 스프링 빈

```java
String[] beanNames = app.getBeanDefinitionNames();
    for(String beanName : beanNames){
        BeanDefinition definition = app.getBeanDefinition(beanName);
            
        if(definition.getRole() == BeanDefinition.ROLE_APPLICATION){
        Object bean = app.getBean(beanName);
        System.out.println("bean : " + bean);
    }
}
```

스프링 컨테이너에 등록된 스프링빈 정보를 보고 싶으면 위의 코드를 쳐보면 된다. 스프링 빈에는 두가지 종류가 있다.

- ROLE_APPLICATION
  
  사용자가 직접 등록한 빈
  
- ROLE_INFRASTRUCTURE
  
  스프링에서 내부에서 사용하는 빈

<div style="height: 28px"></div>

스프링 빈의 정보를 보는 방법에도 여러가지가 있다.

- 이름으로 검색하기(객체이름도 써야함)
- 추상화 객체로 검색하기
- 구현 객체로 검색하기

이 중 추상화 객체로 검색하기와 구현 객체로 검색하기는 객체로 검색하기 때문에 여러개의 빈이 나올 수 있다. 이 경우 `getBean()`으로 했을 경우 에러가 난다. 또한 구현 객체로 검색하는 것은 구현체에 의존한다는 것이기 때문에 별로 좋은 코드가 아니다.

추상화 객체로 검색을 했을 때에는 하위 객체의 빈도 모두 검색이 된다. 

## BeanDefinition


