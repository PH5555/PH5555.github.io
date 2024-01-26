---
layout: post
title: Spring - 리퀘스트 스코프
tags: [Spring, 스프링 기본 핵심 원리]
comments: true
---

## 리퀘스트 스코프 개발

```java
@Component
@Scope(value = "request")
public class MyLogger{
    private String uuid;
    private String url;
    
    public void setUrl(String url){
        this.url = url;
    }
    
    public void log(String message){
    	System.out.println("[" + uuid + "]" + "[" + url + "]" + message);
    }
    
    @PostConstruct
    public void init(){
        this.uuid = UUID.randomUUID().toString();
        System.out.println("[" + uuid + "] request scope bean create : " + this);
    }
    
    @PreDestroy
    public void close(){
        System.out.println("[" + uuid + "] request scope bean close : " + this);
    }
}
```

```java
@Controller 
@RequiredArgsConstructor 
public class LogDemoController{
    private final LogDemoService logDemoService;
    private final MyLogger myLogger;
    
    @RequestMapping("log-demo")
    @ResponseBody
    public String logDemo(HttpServletRequest request){
        String url = request.getRequestURL().toString();
        myLogger.setUrl(url);
        
        myLogger.log("controller test");
        logDemoService.logic("testId");
        return "ok";
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class LogDemoService{
    private final MyLogger myLogger;
    
    public void logic(String id){
        myLogger.log("service : " + id);
    }
}
```

url/log-demo를 요청하면 logDemo가 실행되고 로그를 남겨주는 예제이다. 

## 리퀘스트 스코프의 문제점 

위 예제를 실행해보면 안타깝게도 에러가 난다. 왜냐하면 Request 스코프의 특성때문이다. Request 스코프는 사용자가 요청을 보내면 빈을 생성하고 요청이 끝나면 종료된다. 우리가 의존관계 주입을 할 때 MyLogger 빈을 주입받는데, MyLogger 는 요청이 들어오면 생성되므로 스프링 컨테이너에서 찾을 수 없어서 에러가 난것이다. 

## 스코프와 Provider 

전에 배웠던 Provider로 위의 문제를 해결할 수 있다. 

```java
@Controller @RequiredArgsConstructor public class LogDemoController {
  private final LogDemoService logDemoService;
  private final ObjectProvider<MyLogger> myLoggerProvider;

  @RequestMapping("log-demo")
  @ResponseBody
  public String logDemo(HttpServletRequest request) {
    String requestURL = request.getRequestURL().toString();
    MyLogger myLogger = myLoggerProvider.getObject();
    myLogger.setRequestURL(requestURL);
    myLogger.log("controller test");
    logDemoService.logic("testId");
    return "OK";
  }
}
```

MyLogger를 ObejctProvider로 감싸서 의존관계주입을 지연시켰다. http요청이 들어오고 빈이 생성되면 생성된 빈을 찾음으로써 에러를 해결한다.

## 스코프와 프록시

Provider로 해결을 해도 좋지만 더 좋은 방법이 있다.

```java
@Controller 
@RequiredArgsConstructor 
public class LogDemoController{
    private final LogDemoService logDemoService;
    private final MyLogger myLogger;
    
    @RequestMapping("log-demo")
    @ResponseBody
    public String logDemo(HttpServletRequest request){
        String url = request.getRequestURL().toString();
        myLogger.setUrl(url);
        
        myLogger.log("controller test");
        logDemoService.logic("testId");
        return "ok";
    }
}
```

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MyLogger{
    private String uuid;
    private String url;
    
    public void setUrl(String url){
        this.url = url;
    }
    
    public void log(String message){
    	System.out.println("[" + uuid + "]" + "[" + url + "]" + message);
    }
    
    @PostConstruct
    public void init(){
        this.uuid = UUID.randomUUID().toString();
        System.out.println("[" + uuid + "] request scope bean create : " + this);
    }
    
    @PreDestroy
    public void close(){
        System.out.println("[" + uuid + "] request scope bean close : " + this);
    }
}
```

Scope 설정정보에 `proxyMode = ScopedProxyMode.TARGET_CLASS` 를 넣는다. 이 때 프록시 모드는 해당 스코프를 적용하는 대상이 클래스이면 `TARGET_CLASS`, 인터페이스면 `INTERFACE`로 하면 된다. 이러고 실행을 하면 정상적으로 실행이 된다.

![proxy](/assets/img/proxy.PNG)

프록시 모드를 설정하게 되면 CGLIB 라이브러리가 해당 클래스를 상속받는 프록시 빈을 생성하여 등록한다. 따라서 의존관계에도 가짜 프록시 빈이 등록된다. 이 가짜 프록시 빈은 실제 http 요청이 오면 그때 실제 빈을 요청하는 내부로직이 있다. 

- 해당 클래스를 상속받기 때문에 클라이언트는 실제 클래스와 동일하게 사용할 수 있다(다형성)

- Provider와 프록시 둘다 핵심은 진짜 객체 조회를 꼭 필요한 지점까지 지연조회한다는 것이다.

- 싱글톤 스코프처럼 동작하지만 실제 내부 동작은 다르기 때문에 최소화해서 사용해야한다. 무분별하게 사용하면 유지보수가 힘들어진다.
