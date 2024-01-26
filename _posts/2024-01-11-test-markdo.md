---
layout: post
title: Spring - 객체 지향 원리 적용
tags: [Spring, 스프링 기본 핵심 원리]
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

![di1](/assets/img/di1.png)

의존관계 역전 원칙에 의하면 상위 계층이 하위 계층에 의존해야한다. 우리가 원하던 설계는 위의 그림과 같이 사용자가 인터페이스에만 의존하는 설계이다.

![di2](/assets/img/di2.png)

하지만 실제로 까보면 사용자는 인터페이스만 의존할뿐만 아니라 구현자체에도 의존하고 있다. `private DiscountPolicy discountPolicy = new DiscountPolicy();` 이 구문에서 알 수 있다.

그래서 우리는 애플리케이션의 전체 동작 방식을 구성하기위해서 구현객체를 생성하고 연결하는 책임을 가지는 별도의 설정 클래스를 만들어야한다.

또한, DIP가 위반됐기때문에 OCP도 자동적으로 위반하게 된다. 위의 코드를 보면 할인 정책을 바꾸기 위해서 할인 정책 구현을 만들고 클라이언트 코드까지 바꿔버리는것을 알 수 있다. 이는 수정에 대해서는 닫혀 있어야 한다는 OCP를 위반한다.

## 의존관계 주입(Dependency Injection)

우리는 DIP를 지키기 위해서 코드를 다음과 같이 수정할 수 있다. 

```java
private MemberRepository memberRepository;
private DiscountPolicy discountPolicy;

public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

MemoryMemberRepository와 FixDiscountPolicy라는 구현체를 없애버렸다. 하지만 이렇게하면 멤버에 null이 들어가기 때문에 당연히 실행이 안된다. 그렇기 때문에 의존관계를 외부로부터 주입받아야한다. 이를 AppConfig를 통해서 해결한다.


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

이렇게 되면 의존관계에 대한 고미은 외부에 맡기고 실행에만 집중하면 된다. 이를 의존관계주입 `DI`라고 한다. 

의존관계에는 크게 `정적인 클래스 의존 관계`와 `실행 시점에 결정되는 동적인 객체 의존 관계`가 있다.

- 정적 클래스 의존 관계
  클래스가 사용하는 import 코드만 보고도 의존관계를 파악할 수 있다. 위의 예시에서는 OrderSerciceImpl은 MemberRepository와 DiscountPolicy에 의존한다는 것을 알 수 있다. 하지만 실제 어떤 객체가 OrderServiceImpl에 주입될지는 알 수 없다.

- 동적 클래스 의존 관계
애플리케이션 실행시점에 외부에서 실제 구현 객체를 생성하고 클라이언트에 전달해서 클라이언트와 서버의 실제 의존관계가 연결된다.

또한 DI컨테이너는 IoC컨테이너라고도 하는데 IoC는 제어의 역전이다. 프로그램의 제어 흐름을 AppConfig가 가져가 버려서 클라이언트 코드인 OrderServiceImpl은 구현 객체를 모른다. 이렇게 프로그램의 제어 흐름을 직접 제어하는 것이 아니라 외부에서 관리하는 것을 제어의 역전이라고 한다.

