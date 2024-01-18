---
layout: post
title: Spring - 의존관계 주입의 여러가지 방법
tags: [Spring]
comments: true
---

## 의존관계 주입의 여러가지 방법

지금까지 의존관계 주입을 수동과 자동으로 해봤다. 자동 의존관계 주입의 방법에는 총 4가지가 있다.

1. 생성자 의존관계 주입
   
```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

우리가 그동안 사용한 기본적인 방법이다. 생성자 위에 `@Autowired`를 붙이면 사용할 수 있다. 

- 생성자는 생성될 때 딱 <b>1번</b> 실행되기 때문에 1번 실행될 것을 보장할 수 있다.
- 생성자로 의존관계를 주입할 시, 생성자는 호출시점이외에는 실행되지 않기 때문에 `불변` `필수` 의존관계이다.

> 생성자가 단 1개일 때, @Autowired를 붙이지 않아도 의존관계가 주입된다.

2. 수정자(Setter) 의존관계 주입

```java
private int ex;

public void setEx(int ex){
  this.ex = ex;
}
```

자바에서 필드값을 수정할 때는 위와같은 Setter를 이용한다. 이와 같이 Setter를 이용해서 의존관계를 주입해줄 수 있다.

```java
@Autowired
public void setMemberRepository(MemberRepository memberRepository){
    this.memberRepository = memberRepository;
}

@Autowired
public void setDiscountPolicy(DiscountPolicy discountPolicy){
    this.discountPolicy = discountPolicy;
}
```

- 생성자는 언제든 호출할 수 있고, 필수로 실행되지 않아도 되기때문에 `선택` `변경`가능성이 있는 의존관계에 사용한다.

> 수정자 의존관계 주입은 생성자 의존관계 주입보다 항상 늦게 호출된다. 하지만 수정자 의존관계 주입간의 순서는 보장되지 않는다.

3. 필드 의존관계 주입

말그대로 필드 자체에서 의존관계를 주입하는 방법이다.

```java
@Autowired private final MemberRepository memberRepository;
@Autowired private final DiscountPolicy discountPolicy;
```

하지만 이 방법은 사용하지 않을 것을 권장한다. 왜냐하면 스프링을 사용하지 않고 순수 java로만 테스트코드를 작성할 때 이 코드가 있으면 의존관계를 주입하지 못하기 때문이다.
   
4. 메서드 의존관계 주입

메서드에 @Autowired를 붙이는 방법이다. 

```java
@Autowired
public void setting(MemberRepository memberRepository, DiscountPolicy discountPolicy){
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

사실 Setter 의존관계 주입이랑 다를게 없다. 다만 여러가지 의존관계 주입을 다 같이 할 수 있다는 차이점이 있다. 거의 사용하지 않는 방법이다.

## 옵션 처리

가끔씩 주입할 스프링 빈이 없어도 동작해야할 때가 있다. 하지만 지금까지의 코드는 항상 의존관계를 주입하려고 한다. 이를 required 옵션을 주어 해결할 수 있다.

```java
@Autowired(required = false)
public void NoBean1(Member member){
    System.out.println("nobean1 : " + member);
}

@Autowired
public void NoBean2(@Nullable Member member){
    System.out.println("nobean2 : " + member);
}

@Autowired
public void NoBean3(Optional<Member> member){
    System.out.println("nobean3 : " + member);
}
```

자동 주입을 옵션으로 처리하는 방법에는 `required = false` `@Nullable` `Optional` 3가지 방법이 있다.

```
nobean2 : null
nobean3 : Optional.empty
```

해당 코드를 실행해보면 위와같은 결과를 얻게 된다. 첫번째 방법인 `required = false`는 주입할 대상이 없으면 아예 메서드를 실행하지 않는다. 해당 방법을 제외한 나머지 방법들은 메서드 자체는 실행된다.
