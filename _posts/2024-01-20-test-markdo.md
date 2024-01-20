---
layout: post
title: Spring - 의존관계 주입의 선택
tags: [Spring]
comments: true
---

## 어떤 방법을 선택할까?

앞에서 살펴봤듯이 의존관계 주입에는 여러가지 방법이 있다. 하지만 실질적으로는 생성자 주입밖에 쓰지 않는다.

1. 생성자로 의존관계를 주입하면 그 이후에 변경될 일이 없다. 사실 모든 애플리케이션에서 의존관계는 최대한 바뀌지 않는게 좋다.
   
2. 만약 Setter로 의존관계를 주입하면 순수 자바로 코드를 작성할 때 의존관계를 누락할 위험성이 있다. 하지만 생성자 주입을 쓰면 의존관계를 주입하지 않을 시 컴파일 에러를 띄워준다.

3. 생성자로 의존관계를 주입하면 final 키워드를 쓸 수 있다. 혹시 의존관계를 누락하는 실수를 없애준다.

## 생성자 의존관계 주입 최적화

하지만 모든 방법중 생성자 의존관계 주입이 제일 길고 귀찮은 것은 사실이다. 이런 귀찮은것을 편하게해주는 `lombok`라이브러리가 존재한다.

```java
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

```java
import lombok.RequiredArgsConstructor;

@Component
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService{
    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;
}
```

lombok의 `RequiredArgsConstructor`는 필수 인자(final)가 들어가는 생성자를 생성해준다. 실제로 클래스를 열어보면 생성자가 생성된것을 볼 수 있다.

> 이 기능 외에도 `getter` `setter`등 편리한 기능이 많다.

> 필수 인자가 추가되어도 따로 코드를 수정할 필요가 없다.

