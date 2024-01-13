---
layout: post
title: Spring - 객체 지향 원리 적용
tags: [Spring]
comments: true
---

## 기능이 변경되었다..!

악덕한 기획자들은 요구사항을 매번 바꾸기 마련이다..ㅠ 만약 현재 VIP 고객들을 대상으로 1000원을 할인하는 행사가 있다고 하자. 근데 기획자가 가격이 다른 상품마다 다른 할인율을 적용하고 싶다고 한다. 

현재 코드는 다음과 같다

```java
public class FixDiscountPolicy implements DiscountPolicy {
    
    private int discountFixPolicy = 1000;
    
    @Override 
    public int discount(Member member, int price){
        if(member.getGrade() == Grade.VIP){
            return discountFixPolicy;	
        }
        else{
            return 0;
        }
    }
}
```

```java
public class OrderServiceImpl implements OrderService{
    
    private MemberRepository memberRepository = new MemoryMemberRepository();
    private DiscountPolicy discountPolicy = new FixDiscountPolicy();
    
    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
     
		return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

할인을 구현하는 FixDiscountPolicy는 DiscountPolicy 인터페이스를 구현하기 때문에 FixDiscountPolicy를 새로운 구현으로 바꿔끼우면 된다.

```java
public class RateDiscountPolicy implements DiscountPolicy {
    private final int rate = 10;
    @Override 
    public int discount(Member member, int price){
        if(member.getGrade() == Grade.VIP){
            return price * rate / 100;
        }
        else{
            return 0;
        }
    }
}
```

```java
public class OrderServiceImpl implements OrderService{
    
    private MemberRepository memberRepository = new MemoryMemberRepository();
    private DiscountPolicy discountPolicy = new DiscountPolicy();
    
    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);
     
		return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

애초부터 인터페이스로 설계를 했기 때문에 해당 코드는 꽤 괜찮게 보인다. (나도 처음엔 그랬다 ㅎㅎ)

하지만 이 코드는 DIP, OCP원칙을 위반한다.

> 의존관계 역전 원칙은 소프트웨어 모듈들을 분리하는 특정 형식을 지칭한다. 이 원칙을 따르면, 상위 계층이 하위 계층에 의존하는 전통적인 의존관계를 반전시킴으로써 상위 계층이 하위 계층의 구현으로부터 독립되게 할 수 있다.

> 개방-폐쇄 원칙은 '소프트웨어 개체는 확장에 대해 열려 있어야 하고, 수정에 대해서는 닫혀 있어야 한다'는 프로그래밍 원칙이다.

먼저 DIP(의존관계 역전 원칙)을 보겠다. 

![a](https://github.com/CookieDOG/cookie-dog-mobile/assets/47862506/e46d303a-d240-4029-92f5-0b9f6115533b)

의존관계 역전 원칙에 의하면 상위 계층이 하위 계층에 의존해야한다. 우리가 원하던 설계는 위의 그림과 같이 사용자가 인터페이스에만 의존하는 설계이다.

![b](https://github.com/CookieDOG/cookie-dog-mobile/assets/47862506/41cc3c7a-d87b-40b9-84ef-7b9f05ff3965)

하지만 실제로 까보면 사용자는 인터페이스만 의존할뿐만 아니라 구현자체에도 의존하고 있다. `private DiscountPolicy discountPolicy = new DiscountPolicy();` 이 구문에서 알 수 있다.

그래서 우리는 애플리케이션의 전체 동작 방식을 구성하기위해서 구현객체를 생성하고 연결하는 책임을 가지는 별도의 설정 클래스를 만들어야한다.

```java
public class AppConfig {
    public MemberService memberService() {
        return new MemberServiceImpl(new MemoryMemberRepository());
    }
    
    public OrderService orderService() {
        return new OrderServiceImpl(new MemoryMemberRepository(), new RateDiscountPolicy());
    }
}
```

```java
private MemberRepository memberRepository;
private DiscountPolicy discountPolicy;

public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

이렇게 되면 의존관계에 대한 고미은 외부에 맡기고 실행에만 집중하면 된다. 이를 의존관계주입 `DI`라고 한다.
