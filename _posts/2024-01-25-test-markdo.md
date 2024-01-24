---
layout: post
title: Spring - 프로토타입 스코프
tags: [Spring]
comments: true
---

## 빈의 생명주기

빈은 스프링 컨테이너의 생성과 함께 생성되어서 스프링 컨테이너가 종료될 때까지 유지된다. 이것은 스프링 빈을 싱글톤 스코프로 만들었기 때문이다. 이것처럼 빈 스코프는 빈이 존재할 수 있는 범위를 만한다.

1. 싱글톤 : 하나의 인스턴스를 공유하여 사용한다. 스프링 컨테이너가 시작, 종료될때까지 유지된다.
2. 프로토타입 : 스프링 컨테이너는 빈이 생성되고 의존관계주입까지만 관여하고 더는 관리하지 않는다.
3. 웹 스코프 :

스코프는 `@Scope()`로 지정할 수 있다.

```
public void prototypeBeanFind() {
  AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(PrototypeBean.class);
  System.out.println("find prototypeBean1");
  PrototypeBean prototypeBean1 = ac.getBean(PrototypeBean.class);
  System.out.println("find prototypeBean2");
  PrototypeBean prototypeBean2 = ac.getBean(PrototypeBean.class);
  System.out.println("prototypeBean1 = " + prototypeBean1);
  System.out.println("prototypeBean2 = " + prototypeBean2);
  assertThat(prototypeBean1).isNotSameAs(prototypeBean2);
  ac.close(); //종료
}
@Scope("prototype")
static class PrototypeBean {
  @PostConstruct
  public void init() {
    System.out.println("PrototypeBean.init");
  }

  @PreDestroy
  public void destroy() {
  System.out.println("PrototypeBean.destroy");
  }
}

```

위 코드를 실행하면 마지막 종료 메시지인 PrototypeBean.destroy가 나오지 않는다. 이건 프로토타입 스코프의 특징때문이다. 프로토타입 스코프는 빈을 생성하고 의존관계를 주입하는것까지만 관여하고 그 이후부터는 빈을 조회한 클라이언트가 관리를 하기 때문이다. 그렇기 때문에 종료메시지를 출력하려면 코드에 직접 `prototypeBean1.destroy()`를 해야한다.

