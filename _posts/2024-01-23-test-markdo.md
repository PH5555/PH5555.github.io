---
layout: post
title: Spring - 조희 빈이 2개 이상일 때 해결
tags: [Spring, 스프링 기본 핵심 원리]
comments: true
---

## 조회 빈이 2개 이상이면?

지금까지 DiscountPolicy를 쓸 때 한 구현체에만 `@Component`를 붙였다. 하지만 두개의 구현체 모두 빈으로 만들면 어떻게 될까?

```java
OrderService orderService = app.getBean(OrderService.class);
```

```
Parameter 1 of constructor in project.order.OrderServiceImpl required a single bean, but 2 were found:
        - fixDiscountPolicy: defined in file [/workspace/spring_study1/build/classes/java/main/project/discount/FixDiscountPolicy.class]
        - rateDiscountPolicy: defined in file [/workspace/spring_study1/build/classes/java/main/project/discount/RateDiscountPolicy.class]
```

single bean 이 필요하지만 2개가 찾아졌다고 나온다. 빈을 찾을 때 이름, 구현체의 타입으로 찾은것이 아니고 인터페이스 타입으로 찾았기 때문에 2개가 나온것이다.

1. 필드명 매칭

```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy fixDiscountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = fixDiscountPolicy;
}
```

빈 이름과 필드명을 매칭시켜주면 필드명의 빈으로 의존관계 주입이 된다. 해당 예시에서는 `DiscountPolicy discountPolicy`를 `DiscountPolicy fixDiscountPolicy`로 바꿔서 의존관계를 주입했다.

2. @Qulifier 어노테이션

```java
@Component
@Qualifier("fixDiscountPolicy")
```

```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, @Qualifier("fixDiscountPolicy") DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

먼저 구현체의 상단에 `@Qualifier("")`을 붙여주고 이름을 부여해준다. 그리고 이 구현체를 사용하는 곳에 가서 해당 코드를 붙여주면 된다.

3. @Primary 어노테이션

`@Primary` 어노테이션은 간단하게 해당 어노테이션을 붙인 빈을 최우선으로 사용한다.

> `@Primary`보다 `@Qulifier`이 더 우선순위가 높다.

4. Map

```java
private final MemberRepository memberRepository;
private final Map<String, DiscountPolicy> discountPolicy;

@Autowired
public OrderServiceImpl(MemberRepository memberRepository, Map<String, DiscountPolicy> discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}

@Override
public Order createOrder(Long memberId, String itemName, int itemPrice, String discountCode) {
    DiscountPolicy dis = discountPolicy.get(discountCode);
    Member member = memberRepository.findById(memberId);
    int discountPrice = dis.discount(member, itemPrice);
 
return new Order(memberId, itemName, itemPrice, discountPrice);
}
```

조회된 빈이 모두필요할 때는 Map, List를 쓰면 된다. DiscountPolicy로 받던것을 Map<String, DiscountPolicy>로 받으면 된다. String에는 빈이름이, DiscountPolicy에는 빈이 들어온다. 

```java
Order order = orderService.createOrder(1L, "book", 20000, "fixDiscountPolicy");
```

사용할 때는 빈이름을 맞춰서 넣어주면된다. 이 방법은 다형성을 활용하고 싶을 때 유용하다.

## 의존관계 자동, 수동 뭐가 더 좋을까?

결론부터 말하면 자동을 훨씬 많이 쓴다. 빈이 한두개일 때는 괜찮지만 실제 현업에서는 빈이 백개 천개가 넘어간다. 그 모든 빈에 일일이 `@Bean`을 붙이고 관리하기란 너무 힘든일이다. 게다가 자동 의존관계주입도 `OCP` `DIP`를 지킨다. 

하지만 수동 의존관계주입이 완전히 필요가 없는것은 아니다. 기술지원로직(로그, 데이터베이스 관련)은 수동 의존관계주입을 하는것이 좋다. 기술지원로직은 업무로직보다 그 수가 월등히 적고 영향을 주는 필드가 훨씬 넓다. 또한 업무 로직에 비해 에러가 직접적으로 보이지 않는다. 따라서 기술지원로직은 수동으로 등록해서 에러가 명확히 드러나게 해주는것이 좋다.

또한 다형성을 활용할 때 수동으로 하는것이 좋다. 위에서 우리가 `Map<String, DiscountPolicy> discountPolicy` 을 썼는데 이 코드만 보고는 DiscountPolicy에 어떤것이 들어올지 예상이 가지 않는다. 이런 경우 특정 패키지에 빈을 같이 묶어두면 가독성이 올라간다.

```java
@Configuration
public class DiscountPolicyConfig {
 
 @Bean
 public DiscountPolicy rateDiscountPolicy() {
 return new RateDiscountPolicy();
 }
 @Bean
 public DiscountPolicy fixDiscountPolicy() {
 return new FixDiscountPolicy();
 }
}

```
