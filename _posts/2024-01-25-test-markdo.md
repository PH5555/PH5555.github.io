---
layout: post
title: Spring - 프로토타입 스코프
tags: [Spring, 스프링 기본 핵심 원리]
comments: true
---

## 프로토타입 스코프란?

빈은 스프링 컨테이너의 생성과 함께 생성되어서 스프링 컨테이너가 종료될 때까지 유지된다. 이것은 스프링 빈을 싱글톤 스코프로 만들었기 때문이다. 이것처럼 빈 스코프는 빈이 존재할 수 있는 범위를 만한다.

1. 싱글톤 : 하나의 인스턴스를 공유하여 사용한다. 스프링 컨테이너가 시작, 종료될때까지 유지된다.
2. 프로토타입 : 스프링 컨테이너는 빈이 생성되고 의존관계주입까지만 관여하고 더는 관리하지 않는다.
3. 웹 스코프 : 웹 요청이 나갈때까지 유지되거나 웹 세션이 종료될 때까지 유지된다.

스코프는 `@Scope()`로 지정할 수 있다.

```java
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

## 프로토타입 스코프와 싱글톤 스코프를 같이 쓸 때의 문제점

![singleton1](/assets/img/scope.PNG)

위 사진과 같이 싱글톤 빈에서 프로토타입 스코프를 사용할 수 있다. 이와 같이 코드를 짤 경우 새로운 빈을 생성할 때마다 새로운 객체가 생성되길 기대할 것이다.(프로토타입 빈)

```java
static class ClientBean{
  private final PrototypeBean prototypeBean;
  
  @Autowired
  public ClientBean(PrototypeBean prototypeBean){
      this.prototypeBean = prototypeBean;
  }
  
  public int logic(){
      prototypeBean.addCount();
      int count = prototypeBean.getCount();
      return count;
  }
}

@Scope("prototype")
static class PrototypeBean{
  
  private int count = 0;
  
  public void addCount(){
      count++;
  }
  
  public int getCount(){
      return count;
  }
  
  @PostConstruct 
  public void init(){
      System.out.println("init");
  }
  
  @PreDestroy 
  public void destroy(){
      System.out.println("destroy");
  }
}
```

```java
AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext(ClientBean.class, PrototypeBean.class);
        
ClientBean bean1 = app.getBean(ClientBean.class);
int count1 = bean1.logic();
System.out.println(count1);
ClientBean bean2 = app.getBean(ClientBean.class);
int count2 = bean2.logic();
System.out.println(count2);

System.out.println("bean1" + bean1);
System.out.println("bean2" + bean2);

app.close();
```

즉 이 코드에서 bean1에서 1, bean2에서 1이 나오길 기대할 것이다. 하지만 직접 실행해보면 bean1에서 1, bean2에서 2가 나온다.

왜냐하면 싱글톤빈이 생성되고 의존관계가 주입되면 다시는 주입이 일어나지 않기 때문이다. 빈을 사용할 때마다 새롭게 생성되는것이 아니다.

## 해결방법

```java
static class ClientBean{
    @Autowired
    private ApplicationContext ac;
    
    public int logic(){
      PrototypeBean prototypeBean = ac.getBean(PrototypeBean.class);
        prototypeBean.addCount();
        int count = prototypeBean.getCount();
        return count;
    }
}
```

이렇게 요청을 할 때마다 빈을 새롭게 생성해주는 간단한 방법이 있다. 하지만 이 방법은 스프링 컨테이너에 종속적인 코드가 되고 단위테스트도 어렵게 된다.

```java
static class ClientBean{
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeProvider;
    
    public int logic(){
      PrototypeBean prototypeBean = prototypeProvider.getObject();
        prototypeBean.addCount();
        int count = prototypeBean.getCount();
        return count;
    }
}
```

위와같은 문제점을 해결하기 위해서 빈을 컨테이너에서 찾아주는 DL 서비스를 제공하는 ObjectProvider가 있다. 실행해보면 ObjectProvider가 항상 새로운 빈을 생성하는것을 볼 수 있다.

> 의존관계 검색(Dependency Lookup)은 의존관계가 있는 객체를 외부에서 주입 받는 것이 아닌, 의존관계가 필요한 객체에서 직접 검색하는 방식을 말합니다. 헷갈릴 수 있는 부분이 클라이언트 객체(의존관계가 필요한 객체)에서는 의존하고자 하는 인터페이스 타입만 지정해서 검색할 뿐 해당 인터페이스를 구현한 구체적인 클래스 객체에 대한 결정과 해당 객체에 대한 생명 주기는 IoC 컨테이너에서 책임집니다.

