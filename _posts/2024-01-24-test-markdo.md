---
layout: post
title: Spring - 빈 생명주기 콜백
tags: [Spring, 스프링 기본 핵심 원리]
comments: true
---

## 빈의 생명주기

네트워크 소켓을 연결하거나 데이터베이스에 연결할 때 보면 연결을 완료하고 종료하는 시점에 보면 연결을 끊는다. 스프링도 이와 같이 종료작업을 할 수 있다.

public class NetworkClient{
    private String url;
    
    public NetworkClient() {
        System.out.println("생성자호출 url : " + url);
        connect();
        call("초기화 연결 메시지");
    }
    
    public void setUrl(String url) {
        this.url = url;
    }
    
    public void connect(){
        System.out.println("connet url : " + url);
    }
    
    public void call(String message){
        System.out.println("call : " + url + " message : " + message);
    }
    
    public void disConnect(){
        System.out.println("disconnect : " + url);
    }
}

해당 클래스를 만들어준다.

```java
@Configuration 
static class Config{
    
    @Bean
    public NetworkClient networkClient(){
        NetworkClient networkClient = new NetworkClient();
        networkClient.setUrl("http://hello");
        return networkClient;
    } 
}
```

```
생성자 호출, url = null
connect: null
call: null message = 초기화 연결 메시지
```

실행해보면 다음과 같은 메시지가 출력된다. 출력은 생성자에서 하는데 생성자가 실행되는 시점에는 url에 아무런 값이 들어가지 않았으니 당연한 결과이다.

`스프링 컨테이너 생성 -> 스프링 빈 생성 -> 의존관계 주입 -> 초기화 콜백 사용 -> 소멸전 콜백 -> 스프링 종료`

스프링의 이벤트 라이프사이클은 다음과 같다. 항상 컨테이너가 먼저 생성되고 의존관계가 주입된다. 초기화는 의존관계가 주입되고 설정되어야하므로 이때 콜백이 호출되어서 초기화를 할 수 있도록 해준다.

> 생성자는 필수 정보(파라미터)를 받고, 메모리를 할당해서 객체를 생성하는 책임을 가진다. 반면에 초기화는 이렇게 생성된 값들을 활용해서 외부 커넥션을 연결하는등 무거운 동작을 수행한다. 따라서 생성자 안에서 무거운 초기화 작업을 함께 하는 것 보다는 객체를 생성하는 부분과 초기화 하는 부분을 명확하게 나누는 것이 유지보수 관점에서 좋다. 물론 초기화 작업이 내부 값들만 약간 변경하는 정도로 단순한 경우에는 생성자에서 한번에 다 처리하는게 더 나을 수 있다.

콜백을 지원하는 방법에는 크게 3가지가 있다.

1. InitializingBean, DisposableBean

```java
public class NetworkClient implements InitializingBean, DisposableBean {
   private String url;
   public NetworkClient() {
     System.out.println("생성자 호출, url = " + url);
   }
   public void setUrl(String url) {
     this.url = url;
   }
   //서비스 시작시 호출
   public void connect() {
     System.out.println("connect: " + url);
   }
   public void call(String message) {
     System.out.println("call: " + url + " message = " + message);
   }
   //서비스 종료시 호출
   public void disConnect() {
     System.out.println("close + " + url);
   }
   @Override
   public void afterPropertiesSet() throws Exception {
     connect();
     call("초기화 연결 메시지");
   }
   @Override
   public void destroy() throws Exception {
     disConnect();
   }
}
```

InitializingBean, DisposableBean 인터페이스를 쓰면 `afterPropertiesSet` 메서드와 `destroy` 메서드를 구현해야한다. `afterPropertiesSet`가 의존관계주입후에 실행되는 콜백이고  `destroy`가 스프링 컨테이너가 종료되기 전에 호출되는 콜백이다. 이곳에 초기화와 종료 로직을 넣으면 된다.

- 인터페이스이기 때문에 메서드 이름을 바꿀 수 없다.
  
- 코드를 변경할 수 없는 외부 라이브러리에 적용할 수 없다.

2. 초기화, 소멸 메서드 적용

빈 정보에 ` @Bean(initMethod = "init", destroyMethod = "close")` 처럼 등록하여 쓸 수 있다. 

- 메서드 이름을 자유롭게 줄 수 있다.
  
- 스프링 빈이 스프링 코드에 의존하지 않는다.
  
- 외부 라이브러리에도 적용할 수 있다.

3. @PostConstruct, @PreDestroy 어노테이션

```java
@PostConstruct
public void init() {
  System.out.println("NetworkClient.init");
  connect();
  call("초기화 연결 메시지");
}

@PreDestroy
public void close() {
  System.out.println("NetworkClient.close");
  disConnect();
}
```

- 스프링 코드에 의존하지 않는다. (해당 어노테이션은 스프링이 아니라 자바 자체의 어노테이션이다.)

- 컴포넌트 스캔에 잘 어울린다.

- 외부 라이브러리에는 적용할 수 없다.

> 정리하자면 평소에는 @PostConstruct, @PreDestroy 를 쓰고 외부 라이브러리를 수정해야할때에만 @Bean의 초기화 소멸 메서드를 이용하자
